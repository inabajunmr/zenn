---
title: "Hydra と Spring Security を連携してOAuth 2.0 のアクセストークンによる認可をする"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Hydra", "SpringSecurity", "oauth2"]
published: false
---

# Hydra と Spring Security を連携

[Spring Security](https://spring.io/projects/spring-security) でログインを実装したアプリケーション（ユーザーが ID とパスワードを入力して、認証すると各機能が使える）と [Hydra](https://github.com/ory/hydra) を連携して、ユーザーのアクセストークンを発行し、アクセストークンを利用して Spring で実装した API を呼びだす、をやります。

:::message
構成もコードも本番で動作するには色々不足しているので、動作のイメージ程度に読んでください。
:::

シーケンスは以下となります。

<!-- ![](http://www.plantuml.com/plantuml/png/lLJFgzD04BxdhrZUNRpd8A_WmTmUIl7Y9VY2FD2yHveAthex5LDRIHIrg9HAqTgcQfMWKXlzPuRjZrF_1JURDjjY33bvoGuxttppxJTCZckkwRZrUtSryxPxTjzqBLAjQDKAkNI589iYZ96zGpP0Y10y1Sh9EPPRT-oS3DBFo-VpTM-mvRtsuDGnzI8WqZQorfh-M7wOgvwoRjjcItLquy8vi-ySwMcIf8K88K8ZOOQFieiIdbLaSk4JiYyI6nseJ74wCQvUFhcfwjDhBJqKJWcOhC8dgr6yy0B-k8_isRhnUjb_ZxHxoyuWo2A466624210ef6-GjOcrUi64JNSVyb_TxEdAjSApL1AobcHudT6yA2pQFgv5Y0C4xdiU_AbdHHPmXBb2N_X8stom4CcWZT8XyXcEnUwI4EaOp7z1JpaRxyg0meffOh_QJCHa-N-L3evG-WXH2BuvxBe-VZ27VhxQdzXm9DZcCfCc0Hi9AczVC475NdSQeCsiISfVqcy6Cqt4KZRuatS4ljLEJ48VneZZVYZdy9crR4SWrOEjlTSmtQTyzg59VgSs_aFt1t7bdqg_c5_0000) -->

![](/images/hydra-with-spring.png)

## 実装

コード全体は[こちら](https://github.com/inabajunmr/spring-security-hydra)を参照ください。

### Spring 側

#### 認可リクエスト〜ログイン

username と パスワードで認証可能なアプリケーションを設定します。

```java
@EnableWebSecurity
@Configuration
class WebSecurityConfig(private val restTemplate: RestTemplate) : WebSecurityConfigurerAdapter() {

    @Throws(Exception::class)  // @formatter:off
    override fun configure(http: HttpSecurity) {

        http.authorizeRequests { requests ->
            requests
                    .antMatchers(HttpMethod.GET, "/")
                    .authenticated()
                    .antMatchers("/consent")
                    .authenticated()
                    .antMatchers(HttpMethod.GET, "/login")
                    .authenticated()
        }.formLogin()
                .successHandler(HydraAuthenticationSuccessHandler(restTemplate))
                .loginPage("/login")
                .permitAll()
    }

    override fun configure(auth: AuthenticationManagerBuilder) {
        auth.inMemoryAuthentication()
                .withUser("user1").password("{noop}password").roles("USER")
                .and()
                .withUser("user2").password("{noop}password").roles("USER")
    }
}
```

Hydra からリダイレクトされたリクエストに対してユーザーがログインを完了したら、再度 Hydra にリダイレクトする必要がありますが、今回は [SavedRequestAwareAuthenticationSuccessHandler](https://docs.spring.io/spring-security/site/docs/5.5.1/api/org/springframework/security/web/authentication/SavedRequestAwareAuthenticationSuccessHandler.html) を拡張する形で実装してみます。

```java
/**
 * AuthenticationSuccessHandler for redirect to Hydra
 */
class HydraAuthenticationSuccessHandler(private val restTemplate: RestTemplate) : SavedRequestAwareAuthenticationSuccessHandler() {

    private val requestCache: RequestCache = HttpSessionRequestCache()

    override fun onAuthenticationSuccess(request: HttpServletRequest, response: HttpServletResponse, authentication: Authentication) {

        val savedRequest = this.requestCache.getRequest(request, response)
        logger.info("savedRequest:${savedRequest}")

        if (savedRequest != null) {
            if (savedRequest.parameterMap["login_challenge"] != null) {
                // Hydra にログイン成功を通知
                val builder = UriComponentsBuilder.fromHttpUrl("http://localhost:9001/oauth2/auth/requests/login/accept")
                        .queryParam("login_challenge", savedRequest.parameterMap["login_challenge"]!![0])
                val req = RequestEntity.put(builder.toUriString())
                        .body(HydraAcceptLoginRequest(authentication.name))
                logger.info(req.toString())
                var res = restTemplate.exchange(req, HydraAcceptLoginResponse::class.java)
                logger.info("Hydra response: ${res.body.toString()}")
                this.requestCache.removeRequest(request, response)

                // Hydra にリダイレクト
                redirectStrategy.sendRedirect(request, response, res.body?.redirectTo)
                return
            }
        }

        super.onAuthenticationSuccess(request, response, authentication)
    }
}
```

Hydra に認可リクエストを送るとログイン画面にリダイレクトされるのですが、この際 `login_challenge` というパラメーターが付与されます。
本実装では、`login_challenge` が付与されたリクエストを受け取った場合、[ログイン完了後に Hydra の API をコール](https://www.ory.sh/hydra/docs/reference/api#operation/acceptLoginRequest)し、Hydra から取得したリダイレクト URI にリダイレクトしています。


#### 同意の取得


同意用のコントローラーを定義します。

```java
@Controller
class ConsentController(private val restTemplate: RestTemplate, private val session: HttpSession) {

    private val logger = org.apache.commons.logging.LogFactory.getLog(this.javaClass)

    @GetMapping("consent")
    fun consentForm(@RequestParam("consent_challenge") challenge: String, model: Model): String {
        // https://www.ory.sh/hydra/docs/reference/api#operation/getConsentRequest
        val builder = UriComponentsBuilder.fromHttpUrl("http://localhost:9001/oauth2/auth/requests/consent")
                .queryParam("consent_challenge", challenge)
        var res = restTemplate.getForEntity(builder.toUriString(), HydraConsentRequestInformationResponse::class.java)
        logger.info("Hydra response: ${res.body}")

        model.addAttribute("requestedScope", res.body?.requestedScope)
        model.addAttribute("clientId", res.body?.client?.clientId)

        session.setAttribute("consent_challenge", challenge)
        session.setAttribute("requested_scope", res.body?.requestedScope)
        return "consent"
    }

    @PostMapping("consent")
    fun consent(): String {

        var challenge = session.getAttribute("consent_challenge")
        session.removeAttribute("consent_challenge")
        var requestedScope = session.getAttribute("requested_scope")
        session.removeAttribute("requested_scope")

        val builder = UriComponentsBuilder.fromHttpUrl("http://localhost:9001/oauth2/auth/requests/consent/accept")
                .queryParam("consent_challenge", challenge)
        val reqBody = HydraAcceptConsentRequest()
        reqBody.grantScope = requestedScope as List<String>?

        val req = RequestEntity.put(builder.toUriString())
                .body(reqBody)
        var res = restTemplate.exchange(req, HydraAcceptConsentResponse::class.java)
        return "redirect:${res.body?.redirectTo}"
    }
}
```

Hydra から同意取得のためにリダイレクトしてくるリクエストには、`consent_challenge` が含まれています。
この値を[再度 Hydra にわたす](https://www.ory.sh/hydra/docs/reference/api#operation/getConsentRequest)ことで、クライアントやクライアントが同意を得ようとしているスコープなどの情報を取得できます。
これらを使って同意取得画面を描画します。ちなみに[ログインのタイミングでも同じようなことができます](https://www.ory.sh/hydra/docs/reference/api#operation/getLoginRequest)が、今回はやっていません。

ユーザーが同意したら、同意画面描画時にセッションに入れておいた `consent_challenge` やスコープの値を [Hydra に渡します](https://www.ory.sh/hydra/docs/reference/api#operation/acceptConsentRequest)。
ログインのときのように Hydra へのリダイレクト URI が返ってくるので、ユーザーをその URI にリダイレクトさせます。
結果、Hydra は認可リクエストで指定したリダイレクト URI に認可レスポンスを返します。

#### リソースサーバーの設定

Introspection したいので、[NimbusOpaqueTokenIntrospector](https://docs.spring.io/spring-security/site/docs/5.5.1/api/org/springframework/security/oauth2/server/resource/introspection/NimbusOpaqueTokenIntrospector.html) の Bean を登録します。

```java
@Bean
fun introspector(): OpaqueTokenIntrospector {
    return NimbusOpaqueTokenIntrospector(introspectionUri, clientId, clientSecret)
}
```

WebSecurityConfig にリソースサーバーの設定を追加します。
`api` スコープで `/api/` 以下の API を呼び出せるようにしておきます。

```java
http.authorizeRequests { requests ->
        requests
                .antMatchers("/api/**").hasAnyAuthority("SCOPE_api")
    }.oauth2ResourceServer { oauth2 -> oauth2.opaqueToken() }
```

適当に API を実装します。

```java
@RestController
class ApiController {

    @GetMapping("/api")
    fun sub(authentication: BearerTokenAuthentication): String {
        return authentication.tokenAttributes["sub"].toString()
    }
}
```

### Hydra の設定

#### CLI のインストール

今回はマイグレーションのために Hydra をインストールします。

```
brew tap ory/hydra
brew install ory/hydra/hydra
```

#### PostgreSQL

立てて hydra データベースを作っておきます。

```
docker run -p 5432:5432 --name hydra-postgres -e POSTGRES_PASSWORD=pass -d postgres
psql -h localhost -p 5432 -U postgres
create database hydra;
quit
```

以下のコマンドで Hydra 用に DB の準備を行います。

```
export DSN=postgres://postgres:pass@localhost:5432/hydra?sslmode=disable
hydra migrate sql --yes $DSN
```

#### Hydra の起動

Hydra を起動します。

```
docker network create hydra-for-spring
docker pull oryd/hydra:v1.10.5-pre.1

# Run Hydra
export DSN=postgres://postgres:pass@host.docker.internal:5432/hydra?sslmode=disable
docker run -d \
  --name hydra-for-spring \
  --network hydra-for-spring \
  -p 9000:4444 \
  -p 9001:4445 \
  -e SECRETS_SYSTEM=hydra-secret-123456 \
  -e DSN=$DSN \
  -e URLS_SELF_ISSUER=http://localhost:9000/ \
  -e URLS_CONSENT=http://localhost:8080/consent \
  -e URLS_LOGIN=http://localhost:8080/login \
  oryd/hydra:v1.10.5-pre.1 serve all --dangerous-force-http 
docker logs hydra-for-spring
```

Hydra にクライアントを登録しておきます。

```
hydra clients create --id hydra-for-spring --secret secret --scope api --callbacks https://example.com/callback --endpoint=http://localhost:9001
```

## 動かしてみる

Spring 側のアプリケーションも起動して認可コードグラントを試してみます。

以下の URI にアクセスして認可リクエストをはじめます。

```
http://localhost:9000/oauth2/auth?client_id=hydra-for-spring&response_type=code&state=3cf22d86-fbde-11eb-9a03-0242ac130003&scope=api&redirect_uri=https://example.com/callback
```

Spring 側のログイン画面にリダイレクトします。

![](/images/hydra-with-spring-login.png)


ログインします。

同意画面に遷移するので、同意します。

![](/images/hydra-with-spring-concent.png)


同意すると以下のような認可レスポンスが返ってきます。

```
https://example.com/callback?code=Z6kPXaFZerxjzFPf_LzHIp8pvotCTOgGOMtPzOn5ytk.HG7X884LANAZCl8mjIc3yxrb6nUUGn_dJ7A6Y8u9rO8&scope=api&state=3cf22d86-fbde-11eb-9a03-0242ac130003
```

Hydra にトークンリクエストをします。

```
curl -X POST -u "hydra-for-spring:secret" -d "grant_type=authorization_code" -d "code=Z6kPXaFZerxjzFPf_LzHIp8pvotCTOgGOMtPzOn5ytk.HG7X884LANAZCl8mjIc3yxrb6nUUGn_dJ7A6Y8u9rO8" -d "redirect_uri=https://example.com/callback" http://localhost:9000/oauth2/token
```

アクセストークンが返ってきます。

```json
{"access_token":"LBE1ihNYOo1vQIYoAOFIuPROcU1n8U56LUGMCvrJNUM.z8R41KeRGkPmOxftVNPcaZPDWoz_GF4MmV01HvvrB5Q","expires_in":3600,"scope":"api","token_type":"bearer"}
```

Spring の API を呼んでみます。

```
curl -H GET 'http://localhost:8080/api' -H 'Content-Type:application/json;charset=utf-8' -H 'Authorization: Bearer LBE1ihNYOo1vQIYoAOFIuPROcU1n8U56LUGMCvrJNUM.z8R41KeRGkPmOxftVNPcaZPDWoz_GF4MmV01HvvrB5Q'
```

アクセストークン発行時にログインしたユーザーの username が返ってきました。

```
user1
```

### ログインとか同意完了時に呼び出す Hydra の API の認証は？

この辺の API は[インターネットに露出しちゃ駄目](https://www.ory.sh/hydra/docs/production#exposing-administrative-and-public-api-endpoints)とのことです。

[Hydra の API は public と admin に別れていて](https://www.ory.sh/hydra/docs/reference/api/)、この辺の API は admin に分類されます。
Introspection も Admin なので露出したい場合どうするか考える必要がありそうです。

ちなみに上記の手順で構築すると public が 9000、admin が 9001 のポートで起動します。

## まとめ

* Hydra はログインや同意を行うアプリケーションと連携して OAuth 2.0 や OIDC のプロトコルを提供するための OSS です
* 本記事では Spring Security でログインを実装したアプリと Hydra を連携してみました
* 全然本番で動かせる構成じゃないので参考程度です

## 参考

* [Hydra](https://github.com/ory/hydra)
* [5 Minute Tutorial | ORY Hydra](https://www.ory.sh/hydra/docs/5min-tutorial)
* [Run ORY Hydra in Docker | ORY Hydra](https://www.ory.sh/hydra/docs/configure-deploy)
* [Database Setup and Configuration | ORY Hydra](https://www.ory.sh/hydra/docs/dependencies-environment)
* [HTTP API Documentation | ORY Hydra](https://www.ory.sh/hydra/docs/reference/api)
* [Spring Security](https://spring.io/projects/spring-security) 
* [SavedRequestAwareAuthenticationSuccessHandler](https://docs.spring.io/spring-security/site/docs/5.5.1/api/org/springframework/security/web/authentication/SavedRequestAwareAuthenticationSuccessHandler.html)
* [NimbusOpaqueTokenIntrospector](https://docs.spring.io/spring-security/site/docs/5.5.1/api/org/springframework/security/oauth2/server/resource/introspection/NimbusOpaqueTokenIntrospector.html)
* [Spring Security Reference | 12.3.15. How Opaque Token Authentication Works](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#oauth2resourceserver-opaque-architecture)
* [Spring Security 5.3.3で Resource Server を構成する](https://dev.classmethod.jp/articles/resource-server-configuration-with-spring-security5/)
* [OAuth 2.0 Resource Server With Spring Security 5 | Baeldung](https://www.baeldung.com/spring-security-oauth-resource-server)
