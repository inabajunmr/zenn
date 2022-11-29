---
title: "Java で認可サーバー/IdP を実装するためのライブラリを実装しながら考えてたこと"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth2", "Hydra", "openidconnect"]
published: false
---

[AzIdP4J](https://github.com/inabajunmr/azidp4j) という Java で認可サーバーや IdP を実装するためのライブラリを書いています。ライブラリを書く上で色々考えていたことについて書きます。

## 認可サーバーや IdP がほしい場合

認可サーバーや IdP がほしい場合、以下のパターンがあるかと思います。

* 全部自前で実装する
* ライブラリを使って実装する
* 連携して認可サーバーや IdP として動く製品を利用する
* スタンドアロンで動く製品を利用する

自前実装の場合当然コードをたくさん書かなければならなかったり、仕様を理解したりする必要があります。個人的には実装はともかく仕様の理解、特にどの仕様までサポートすればよいのかを追いかけたり、ほしい機能がすでに定義されているのかを探したりするのがなかなか大変です。
個人的にはライブラリに認可サーバー/IdP 周りの処理を全部ぶん投げて他はよしなに自分で書きたいと思うことが多かったので、「ライブラリを使って実装する」のためのライブラリを実装することにしました。（Spring Security OAuth が EOL なので Java にしました）

## ポリシーを決める

インターフェースに悩んだときの基準として、大枠で以下のようなライブラリの指針を決めました。

* 特定のフレームワークに依存しない
* ユーザーの体験をできる限り任意に実装できる
* データストアを持たない

### 特定のフレームワークに依存しない

OAuth 2.0 や OpenID Connect はそもそもが Web に依存した仕様であり、クライアントが一切何もしなくても使えるようなものを実装しようとすると HTTP のリクエストを直接受け付ける必要があります。
しかし、Java で Web アプリを作る場合多くの場合フレームワークによってインターフェースがいくつかのパターンがあり、いずれのケースでもなるべく利用できるようにしたいと考えました。
そのため、直接 HTTP リクエストを受け付けるのではなくもう少し抽象的なインターフェースを持つようにしました。

例えばトークンリクエストやトークンレスポンスは以下のようなインターフェースで表現し、HTTP でのやり取りはクライアントがよしなに頑張るような構造にしました。

* https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/request/TokenRequest.java
* https://github.com/inabajunmr/azidp4j/blob/main/azidp4j/src/main/java/org/azidp4j/token/response/TokenResponse.java

これはあまりインターフェースとしては気持ちよくない気がしましたが、[Spring ベースのサンプル実装](https://github.com/inabajunmr/azidp4j/blob/main/azidp4j-spring-security-sample/src/main/java/org/azidp4j/springsecuritysample/handler/TokenEndpointHandler.java#L50)は割とすっきりしたのでまあそんなに悪くもないのかなという気もしています。

逆にプロトコルの範囲内でもフレームワークへ依存する可能性が大きそうな場所はライブラリでは対応せず、クライアントの実装に任せる方針としました。
具体的にはクライアント認証であったり、Bearer トークンによる認可はフレームワーク側で機能を持っている場合が多そうだったのでインターフェースに悩んだ結果サポートしないことにしました。

### ユーザーの体験をできる限り任意に実装できる

認可リクエストはリクエストを受けた後の挙動がプロトコルの範囲で閉じません。例えば prompt=login のリクエストに対してどのような手段や体験で認証を行うかはサービス側でできる限り自由に実装できるようにしたかったため、ライブラリによる処理はプロトコルの範囲で閉じるようにしました。

例えばライブラリでは認証処理を一切持たず、認証が行われていた場合はそのユーザーの ID を受け取り、追加の認証が必要であればその旨だけをレスポンスし、処理自体はクライアント側のコードでよしなにやってもらう、といったインターフェースにしています。

```Java
var response = azIdP.authorize(authzReq);
switch (response.next) {
    case redirect -> {
        // redirect to response.redirect.redirectTo;
    }
    case errorPage -> {
        // Error but can't redirect as authorization response.
        // show error page with response.errorPage.errorType
    }
    case additionalPage -> {
        // When authorization request processing needs additional action by additionalPage.prompt.
        // ex. user authentication or request consent.
    }
}
```

https://github.com/inabajunmr/azidp4j/blob/main/docs/endpoint-implementations.md#authorization-request

このインターフェースは一番悩みました。未だにこれが成立しているのか、これによってユーザー体験が制限されないのか、といったところに全然自信が無いのでなんかあればコメントいただけると嬉しいです。

### データストアを持たない

これは [ory/fosite](https://github.com/ory/fosite) がそんな感じのインターフェースで良さそうだったので真似しました。
このあたりのプロトコルを対応しようとするとどうしてもデータの管理をする必要がでてきますが、インターフェースだけ提供するので任意のデータストアを使って実装してね、という構造にしました。

* https://github.com/inabajunmr/azidp4j/blob/main/docs/config.md#token-stores-configuration

## やることとやらないことを決める

まず最初のマイルストーンとして以下をやることにしました。

* 最低限動くものを作る
* ドキュメントをしっかり書く

これをやるために、当面以下をスコープから外しました。

* スペック全体はカバーしない
    * 例えば Request object をサポートしない、など

これはライブラリの良し悪しよりも個人的なモチベーションのコントロールのためにそうしました。今のところ認識している範囲では後から実装してもなんとかなるような気がしていますが、サポートする仕様を考えるのはなかなか大変そうです。

[Conformance Test](https://openid.net/certification/testing/) はちょくちょく動かしながら実装することにしました。これは進捗が可視化するのでモチベーション的にだいぶプラスになりました。ちょくちょく自分が理解できていなかった仕様もあぶり出されてよかったです。

## 開発の進め方

「特定のフレームワークに依存しない」というポリシーから外れないよう、ライブラリを利用した Identity Provider をライブラリと同時に 2 つ実装することにしました。

* [Spring による実装](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-spring-security-sample)
* [with com.sun.net.httpserver.HTTPServer](https://github.com/inabajunmr/azidp4j/tree/main/azidp4j-httpserver-sample)

このアプローチはインターフェースを決めるのにだいぶ影響があった気がするので良かった気がします。