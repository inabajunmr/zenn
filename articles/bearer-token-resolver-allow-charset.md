---
title: "Spring Security でボディでアクセストークンを受け付けるときに content-type の charset を許容"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["spring security"]
published: true
---

https://docs.spring.io/spring-security/reference/5.7.3/servlet/oauth2/resource-server/opaque-token.html

:::message
Spring Security 5.7.3 でしか試してないです
:::

https://docs.spring.io/spring-security/reference/5.7.3/servlet/oauth2/resource-server/opaque-token.html

のような感じでリソースサーバーを構成すると、[DefaultBearerTokenResolver](https://github.com/spring-projects/spring-security/blob/5.7.x/oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/DefaultBearerTokenResolver.java#L130) でリクエストからトークンを抽出します。

このとき allowFormEncodedBodyParameter=true になっていると https://www.rfc-editor.org/rfc/rfc6750#section-2.2 のようにリクエストボディに指定された `access_token=mF_9.B5f-4.1JqM` といったトークンを利用できます。

仕様では

> The HTTP request entity-header includes the "Content-Type" header field set to "application/x-www-form-urlencoded".

とあり、Spring Security でも [このように ContentType をチェックしている](https://github.com/spring-projects/spring-security/blob/5.7.x/oauth2/oauth2-resource-server/src/main/java/org/springframework/security/oauth2/server/resource/web/DefaultBearerTokenResolver.java#L131) のですが、完全一致なので `application/x-www-form-urlencoded;charset=UTF-8` といったリクエストを許容しません。

仕様上これを許容すべきでないのかどうかはよくわかりませんが、[OIDC の Conformance Testing](https://openid.net/certification/testing/) で UserInfo エンドポイントをリクエストする際にこのようなリクエストをしてきて、リソースサーバーが応答できないという問題がありました。

## どうするか

BearerTokenResolver を実装したクラスを作り DefaultBearerTokenResolver をコピーして当該部分だけよしなに書き換え、Bean に登録してアプリを起動すると実装が差し替わってよしなにしてくれます。以下の実装はサンプルで適当なので動かないケースがあるかもしれません。

```java
private boolean isParameterTokenSupportedForRequest(final HttpServletRequest request) {
    var contentType =
            request.getContentType() != null ? request.getContentType().split(";")[0] : null;
    return (("POST".equals(request.getMethod())
                    && MediaType.APPLICATION_FORM_URLENCODED_VALUE.equals(contentType)
            || "GET".equals(request.getMethod())));
}

private boolean isParameterTokenEnabledForRequest(final HttpServletRequest request) {
    var contentType =
            request.getContentType() != null ? request.getContentType().split(";")[0] : null;
    return ((this.allowFormEncodedBodyParameter
                    && "POST".equals(request.getMethod())
                    && MediaType.APPLICATION_FORM_URLENCODED_VALUE.equals(contentType))
            || (this.allowUriQueryParameter && "GET".equals(request.getMethod())));
    }
```
