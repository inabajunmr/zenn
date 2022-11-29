---
title: "パブリッククライアントでリソースオーナーパスワードクレデンシャルをする際の client_id"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openidconnect"]
published: false
---

リソースオーナーパスワードクレデンシャルは OAuth 2.1 で無くなるとのことですが、

> The Resource Owner Password Credentials grant is omitted from this specification
https://oauth.net/2.1/

OAuth 2.0 の認可サーバーを実装していてちょっと悩んだのでメモしておきます。

認可コードグラントでは、トークンリクエストの仕様に以下のパラメーターが定義されています。

```
client_id
    REQUIRED, if the client is not authenticating with the
    authorization server as described in Section 3.2.1.
```

パブリッククライアントではこのパラメーターによって、認可コードが意図したクライアントに発行されたものかどうかを検証します。

> ensure that the authorization code was issued to the authenticated confidential client, or if the client is public, ensure that the code was issued to "client_id" in the request,

一方リソースオーナーパスワードクレデンシャルグラントでは、このフィールドが定義されておらず、

> If the client type is confidential or the client was issued client credentials (or assigned other authentication requirements), the client MUST authenticate with the authorization server as described in Section 3.2.1.

のようにコンフィデンシャルクライアントの場合は認証を行えば認可サーバーはクライアントを特定できますが、パブリッククライアントの場合これができません。

パブリッククライアントなので偽装し放題だし、そもそもクライアントが特定出来る必要があるのかというと、[JSON Web Token (JWT) Profile for OAuth 2.0 Access Tokens](https://datatracker.ietf.org/doc/html/rfc9068#section-2.2) で client_id が必須だったりするので信用できない前提でも知りたい場合がある気がします。（Introspection は Optional）

が、3.2.1. Client Authentication では以下のように、クライアントが自身が誰なのか知らせるのに client_id をトークンエンドポイントに送っても良い、といった記述があります。

> A client MAY use the "client_id" request parameter to identify itself when sending requests to the token endpoint.  In the "authorization_code" "grant_type" request to the token endpoint, an unauthenticated client MUST send its "client_id" to prevent itself from inadvertently accepting a code intended for a client with a different "client_id".  This protects the client from substitution of the authentication code.  (It provides no additional security for the protected resource.)

クライアント認証を行わず、グラントタイプごとのトークンリクエストに client_id が定義されていない場合も、これを利用すればよさそうです。（値が信用できるかどうかはともかく）

参考までに Auth0 のトークンエンドポイントを見てみたところ、グラントタイプに関わらず clietn_id はすべて REQUIRED となっていました。
https://auth0.com/docs/api/authentication#resource-owner-password
