---
title: "Java で OAuth 2.0 の AZ / OIDC の IdP のためのライブラリを実装しながら考えてたこと"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth2", "Hydra", "openidconnect"]
published: false
---

Digital Identity技術勉強会 #iddance Advent Calendar 2022 22 日目の記事です。

https://qiita.com/advent-calendar/2022/iddance

## AzIdP4J

[AzIdP4J](https://github.com/inabajunmr/azidp4j) という Java で認可サーバーや IdP を実装するためのライブラリを書いています。ライブラリを書く上で色々考えていたことについて書きます。

https://github.com/inabajunmr/azidp4j

こんな雰囲気で動くライブラリを書いています。

```java
// This is constructed via http request generally.
var authorizationRequestQueryParameterMap =
        Map.of(
                "scope", "openid item:read",
                "response_type", "code",
                "client_id", "xyz-client",
                "redirect_uri", "https://client.example.com/callback",
                "state", "abc",
                "nonce", "xyz");
var authorizationRequest =
        new AuthorizationRequest(
                "inabajun", // authenticated user
                Instant.now().getEpochSecond(), // authenticated time
                Set.of("openid", "item:read"), // consented scope
                authorizationRequestQueryParameterMap);
var authorizationResponse = azidp.authorize(authorizationRequest);
System.out.println(authorizationResponse.redirect.redirectTo);
// https://client.example.com/callback?code=890d9cca-11a2-47b8-b879-1f584fdb0354&state=abc
```

## 認可サーバーや IdP がほしい場合

認可サーバーや IdP がほしい場合、以下のパターンがあるかと思います。

* 全部自前で実装する
* ライブラリを使って実装する
* 連携して認可サーバーや IdP として動く製品を利用する
* スタンドアロンで動く製品を利用する

自前実装の場合当然コードをたくさん書かなければならなかったり、仕様を理解したりする必要があります。実装はともかく仕様の理解、特にどの仕様までサポートすればよいのかを追いかけたり、ほしい機能がすでに定義されているのかを探したりするのがなかなか大変です。

個人的にプロトコルにライブラリに関する処理を丸投げして他はよしなに自分で書きたいと思うことが多かったので、「ライブラリを使って認可サーバーや IdP を実装する」のためのライブラリを実装することにしました。（あと Spring Security OAuth が EOL なので Java にしました）

## ポリシーを決める

ライブラリはインターフェースさえ良ければなんとかなるという感覚があります。
インターフェースに悩んだときの基準として、大枠で以下のようなライブラリの指針を決めました。

* 特定のフレームワークに依存せず利用できる
* アクセストークンと ID トークンをなるべく簡単に払い出せる
* ユーザーの体験をできる限り任意に実装できる
* データストアを持たない

### 特定のフレームワークに依存せず利用できる

OAuth 2.0 や OpenID Connect はそもそもが Web に依存した仕様であり、クライアントが一切何もしなくても使えるようなものを実装しようとすると HTTP のリクエストを直接受け付ける必要があります。
しかし、Java で Web アプリを作る場合多くの場合フレームワークによってインターフェースにいくつかのパターンがあり、いずれのケースでもなるべく利用できるようにしたいと考えました。
そのため、直接 HTTP リクエストを受け付けるのではなくもう少し抽象的なインターフェースを持つようにしました。

例えばトークンリクエストやトークンレスポンスは以下のようなインターフェースで表現し、HTTP でのやり取りはクライアントがよしなに頑張るような構造にしました。

* [TokenRequest](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/request/TokenRequest.java)
* [TokenResponse](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/response/TokenResponse.java)

[Spring ベースで実装するとこんな感じ](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j-spring-security-sample/src/main/java/org/azidp4j/springsecuritysample/handler/TokenEndpointHandler.java#L50)になります。

```java
@RequestMapping(
        value = "/token",
        method = RequestMethod.POST,
        consumes = MediaType.APPLICATION_FORM_URLENCODED_VALUE)
public ResponseEntity<Map> tokenEndpoint(@RequestParam MultiValueMap<String, Object> body) {

    // クライアント認証など...
    // Map のリクエストを詰め替え
    var request = new TokenRequest(authenticatedClientId, body.toSingleValueMap();
    // Token Request
    var response =
            azIdP.issueToken(request);

    // AzIdP4J のレスポンスを HTTP レスポンスに詰め替え
    return ResponseEntity.status(response.status).body(response.body);
}
```

逆にプロトコルの範囲内でもフレームワークへ依存する可能性が大きそうな場所はライブラリでは対応せず、クライアントの実装に任せる方針としました。
例えばクライアント認証であったり、Bearer トークンによる認可はフレームワーク側で機能を持っている場合が多そうだったので悩んだ結果サポートしないことにしました。

### アクセストークンと ID トークンをなるべく簡単に払い出せる

OAuth 2.0 や OIDC を利用するケースは色々ありますが、認可コードフローで各トークンを払い出せる、を簡単に導入できる、を目標にすることにしました。
これは自分がいろいろなユースケースについていまいち理解できていないためでもあるのですが、いろいろできるようにするとどうしてもインターフェースがややこしくなりそうだったため、トークンの払い出しをややこしくするくらいなら機能ごとサポートしない、という方向で物事を考えることにしました。

例えば UserInfo エンドポイントをサポートするかいまだにちょっと悩んでいるのですが、これをサポートしようとするとどうしてもユーザーの概念をライブラリに持ち込む必要がありそうで使い方がややこしくなりそうでインターフェーズを決めきれずにいます。こういったものはライブラリの外でも実装可能なのであれば基本的に対象から外す方針としました。

### ユーザーの体験をできる限り任意に実装できる

認可リクエストはリクエストを受けた後の挙動がプロトコルの世界で閉じません。例えば prompt=login のリクエストに対してどのような手段や体験で認証を行うかはプロトコルの外側の話になります。こういったユーザー体験に強く関わる部分はサービス側でできる限り自由に実装できるようにしたいという気持ちがありました。

ユーザーの認証の例でいうと、ライブラリでは認証処理を一切持たず、認証が行われていた場合はそのユーザーの ID を受け取り、追加の認証が必要であればその旨だけを返却し、処理自体はクライアント側のコードでよしなにやってもらう、といったインターフェースにしています。

```Java
// authzReq は認証したユーザーの subject を持っているが、未認証も表現できる
var response = azIdP.authorize(authzReq);
switch (response.next) {
    case redirect -> {
        // response.redirect.redirectTo にリダイレクト
    }
    case errorPage -> {
        // リダイレクトできないが、エラーになるようなケースでのエラー処理
    }
    case additionalPage -> {
        // ログインや同意など必要な処理が返却されるので、よしなに実装する
    }
}
```

https://github.com/inabajunmr/azidp4j/blob/main/docs/endpoint-implementations.md#authorization-request

このあたりのインターフェースは一番悩みました。未だにこれが成立しているのか、これによってユーザー体験が制限されないのか、といったところに全然自信が無いのでなんかあればコメントいただけると嬉しいです。

### データストアを持たない

これは [ory/fosite](https://github.com/ory/fosite) がそんな感じのインターフェースで良さそうだったので真似しました。
トークンなどはどうしてもデータの管理をする必要がでてきますが、インターフェースだけ提供するので任意のデータストアを使って実装してね、という構造にしました。

* [Token Stores Configuration](https://github.com/inabajunmr/azidp4j/blob/main/docs/config.md#token-stores-configuration)

## やることとやらないことを決める

実装していると色々サポートしたい気持ちになってきますが、個人的なモチベーションをコントロールするため、やることとやらないことをある程度明示的に決めることにしました。
まず最初のマイルストーンとして以下をやることにしました。

* 最低限動くものを作る
* ドキュメントをしっかり書く

これをやるために、当面以下をスコープから外しました。

* スペック全体はカバーしない
    * 例えば Request object をサポートしない、など
    * 自分が具体的な需要を理解できていない仕様もなるべくサポートしない

今のところ認識している範囲では後から実装してもなんとかなるような気がしていますが、サポート対象の仕様を考えるのはなかなか大変そうです。

### 最低限動くものを作る

全体として何をサポートするのかは決めず、まず認可リクエストを受け付けてアクセストークン、ID トークンを発行できる、というところを目標にコードを書きました。
この間 [Conformance Test](https://openid.net/certification/testing/) をちょくちょく動かしながら実装することにしました。これは進捗が可視化されるのでモチベーション的にだいぶプラスになりました。ちょくちょく自分が理解できていなかった仕様もあぶり出されてよかったです。

### ドキュメントをしっかり書く

Conformance Test やコードを通じていろいろわからなかったところがあぶり出されていくのはよかったのですが、そもそもインターフェースとして成立しているのか、全く考慮できていないポイントがないか、といった観点は自分ではいまいちわからなかったためリアクションが欲しく、ドキュメントを色々書きました。サンプルコードを割と厚めに書いたと思うのでリアクションいただけると助かります。

https://github.com/inabajunmr/azidp4j/tree/main/docs

## 開発の進め方

「特定のフレームワークに依存しない」というポリシーから外れないよう、ライブラリを利用した Identity Provider をライブラリと同時に 2 つ実装することにしました。

* [Spring による実装](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-spring-security-sample)
* [with com.sun.net.httpserver.HTTPServer による実装](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-httpserver-sample)

このアプローチはインターフェースを決めるのにだいぶ影響があった気がするので良かった気がします。
例えばライブラリはクライアント認証の機能を持っていませんが、ライブラリと組み合わせたときに実際に自前でクライアント認証を実装できるのか？といったようなテーマを考えるのに役立ちました。

また、複数実装するかどうかはともかくライブラリの開発はテストケースとは別の利用者側の実装を同時に行えると良い気がしました。やはり実際に動くアプリケーションとテストケースだと、後者では想定していなかったケースが出てきます。

## どうだったか

プロトコルの話もそうでない話もあんまりわかってないなという気持ちになったのでよかったです。
あるエンドポイントに対して OAuth 2.0 と OIDC どっちの仕様を見ればいいんだ？となったりして以下の表を作って眺めたりしていました。

https://gist.github.com/inabajunmr/8641807eb05a4f4871da86a09c1e6c45

https://gist.github.com/inabajunmr/75cc9834ec0b5a77306253bd9197cbd5

しかしまだよくわかってない感がだいぶあるので RP やリソースサーバーも込みで実装しながらインターフェースが成立しているのか確認したいと思っています。（本当はこれも最初から同時にすすめたらよかった気がする）

このインターフェースだと成立しないよ、みたいのあればコメントいただけると嬉しいです。
