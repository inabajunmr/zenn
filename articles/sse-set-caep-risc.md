---
title: "SSE Framework, SET, CAEP, RISC"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openid", "security"]
published: false
---

# Shared Signals and Events

[Shared Signals and Events – A Secure Webhooks Framework](https://openid.net/wg/sse/) では、

* SAML とか OIDC での ID のフェデレーションだとアクセスの妥当性をログイン時に評価される
* なんだけど、セッションは長時間続くことがあるのでセッションが確立してる間にーザーの属性が変わることがある
* ので、ログイン時の古い情報を基にアクセス制御をするとセキュリティ上よくない

といった文脈でステータスの変更を通知するのに [CAEP](https://openid.net/specs/openid-caep-specification-1_0.html) が出てきたり、

* ユーザーは同じメールアドレスを複数サービスに登録したりする
* 攻撃者はこれを利用して、1 つの脆弱なポイントからアカウント乗っ取りを連鎖したりする
    * メールアカウントが乗っ取られたら別サービスのパスワードリカバリ情報がそのメールアカウントに送られる、みたいなケース

といった文脈で [RISC](https://openid.net/specs/openid-risc-profile-specification-1_0-02.html) について記載があります。

ざっくりまとめると、各ユーザーやサービス、その他色々のセキュリティに関連するイベントをリアルタイムにやりとりして評価したいけどどうやる？のための仕様がいろいろある、といった感じです。

これらを実現するためにいくつかのプロトコルが出てくるのですが、本記事では各プロトコルにどのような仕様が定義されているのかをざっくり解説します。

ちなみに CAEP と RISC の立ち位置の差はよくわかってないので、それについては特に何も書いてないです。

# どのようなシーケンスになるのか

# 各仕様ざっくり

* [OpenID Shared Signals and Events Framework Specification 1.0 - draft 01](https://openid.net/specs/openid-sse-framework-1_0-ID1.html)
    * イベントをやり取りするフレームワーク
    * 何に関するどういったイベントをどうやって送信してもらうかをリクエストするエンドポイントの定義とかがある
* [Security Event Token (SET)](https://datatracker.ietf.org/doc/html/rfc8417)
    * セキュリティイベントを表すデータの構造
    * JWT
* [Push-Based Security Event Token (SET) Delivery Using HTTP](https://datatracker.ietf.org/doc/html/draft-ietf-secevent-http-push-10)
    * SET の配送方法（push）
* [Poll-Based Security Event Token (SET) Delivery Using HTTP](https://datatracker.ietf.org/doc/html/draft-ietf-secevent-http-poll)
    * SET の配送方法（polling）
* [OpenID RISC Profile Specification 1.0 - draft 02](https://openid.net/specs/openid-risc-profile-specification-1_0-02.html)
    * イベントがいろいろ定義されてる
        * クレデンシャル漏洩した、とか
* [OpenID Continuous Access Evaluation Profile 1.0 - draft 02](https://openid.net/specs/openid-caep-specification-1_0.html)
    * イベントがいろいろ定義されてる
        * セッションが無効化された、とか

# [OpenID Shared Signals and Events Framework Specification 1.0 - draft 01](https://openid.net/specs/openid-sse-framework-1_0-ID1.html)

この文章では Receiver（受信者） と Transmitter（送信者） 間でストリームを通じてイベントをやり取りするための仕様が定義されています。

大雑把に SSE Framework によるやりとりの流れをまとめると以下のようになります。

1. Receiver は Transmitter の情報を取得
    1. 各エンドポイントの情報が返ってくる
1. Receiver は Transmitter に設定を送る
    1. イベントをどのように通知して欲しいか、など
1. Receiver は Transmitter に Subject の追加リクエストを送る
    1. 例えばユーザー A についてのイベントを通知してくれ、など
1. 指定した Subject のイベントが Transmitter から Receiver に通知される

## Subject とは

SSE Framework では、イベントが**何**に対するイベントなのかを表すものとして Subject という概念があります。例えば「ユーザーがパスワードを変更した」というイベントであれば Subject がユーザーだったり、「セッションが無効化された」イベントであれば Subject はセッションそのものだったりします。

Receiver はある Subject についてのイベント通知を受け取りたい場合、API を通じて Transmitter に Subject の追加リクエストを送ります。

Subject の表現には Simple Subject Member と Complex Subject Member があり、仕様には以下のような例が記載されています。

Simple Subject Member の例。

```json
"transferer": {
  "format": "email",
  "email": "foo@example.com"
}
```

Complex Subject Member の例。

```json
"transferee": {
  "user" : {
    "format": "email",
    "email": "bar@example.com"
  },
  "tenant" : {
    "format": "iss_sub",
    "iss" : "http://example.com/idp1",
    "sub" : "1234"
  }
}
```

## SET の例

いくつか例が紹介されています。

```json
{
  "iss": "https://idp.example.com/",
  "jti": "756E69717565206964656E746966696572",
  "iat": 1520364019,
  "aud": "636C69656E745F6964",
  "events": {
    "https://schemas.openid.net/secevent/risc/event-type/account-enabled": {
      "subject": {
        "format": "email",
        "email": "foo@example.com"
      }
    }
  }
}
```

events フィールドのキーになっている `https://schemas.openid.net/secevent/risc/event-type/account-enabled` は event type を指します。この event type や type ごとのフィールドが後述の RISC や CAEP で定義されています。

この例では foo@example.com というメールアドレスで表される Subject のアカウントが有効化されたことを `https://idp.example.com/` が `636C69656E745F6964` に通知しています。

## 定義されている API

前述の Subject 追加などに利用する API がいくつか定義されています。

### ディスカバリ

ディスカバリのための API が定義されています。この API から他の各 API のエンドポイントなどを取得できます。

仕様では以下のようなリクエストの例が記載されています。

#### リクエスト

```
GET /.well-known/sse-configuration HTTP/1.1
Host: tr.example.com
```

#### レスポンス

各エンドポイントや、サポートしているイベントの通知方法などが返ってきます。

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
  "issuer":
    "https://tr.example.com",
  "jwks_uri":
    "https://tr.example.com/jwks.json",
  "delivery_methods_supported": [
    "https://schemas.openid.net/secevent/risc/delivery-method/push",
    "https://schemas.openid.net/secevent/risc/delivery-method/poll"],
  "configuration_endpoint":
    "https://tr.example.com/sse/mgmt/stream",
  "status_endpoint":
    "https://tr.example.com/sse/mgmt/status",
  "add_subject_endpoint":
    "https://tr.example.com/sse/mgmt/subject:add",
  "remove_subject_endpoint":
    "https://tr.example.com/sse/mgmt/subject:remove",
  "verification_endpoint":
    "https://tr.example.com/sse/mgmt/verification",
  "critical_subject_members": [ "tenant", "user" ]
}
```

### Status Endpoint

ステータスという概念があり、

* enabled（イベントを送信する）
* paused（イベントの送信をしないが、その間保持したイベントを enabled になったら送信する）
* disabled（イベントを送信しない）

が定義されています。これを参照、更新するための API があります。
ストリームに対するステータスの更新だけではなく、ストリームの特定の Subject を指定した操作も定義されています。

### Configuration Endpoint

利用するイベントの通知方法や、対象とするイベントの種類などのストリームの設定を取得、更新、削除ができます。

### Add Subject Endpoint/Remove Subject Endpoint

前述の Subject をストリームに追加するための API が定義されています。

### Verification Endpoint

イベントストリームを流れるイベントは非常に少ない場合があるため、実際にイベントが通知された場合の挙動を検証するための Verification Event が定義されています。これを送信するために Verification Endpoint が定義されています。

## 認証について

Transmitter の API 呼び出しには OAuth 2.0 のアクセストークンを使った認可を行うべき、とあります。

# [Security Event Token (SET)](https://datatracker.ietf.org/doc/html/rfc8417)

SET はデータ構造で、Subject に対するイベントを表現します。JWT なので署名や暗号化が可能です。

SET を受信したら受信者は何らかのアクションを行うことができますが、送信者が受信者に何か操作を命令するものではなく、受信してどうするか、はあくまで受信者側の操作となります。

SET の配送方法やイベントの構造は別の仕様があります。

# [Security Event Token (SET)](https://datatracker.ietf.org/doc/html/rfc8417)

SET はいくつかのクレームが定義された JWT となります。

* events
    * イベントそのものを格納するクレーム
* txn
    * 関連のある JWT が複数発行される場合の紐付けに使う
* toe
    * イベントの発生日時

# [Push-Based Security Event Token (SET) Delivery Using HTTP](https://datatracker.ietf.org/doc/html/draft-ietf-secevent-http-push-10)

SET の配送方法として push と poll があります。こちらは push の仕様となります。この仕様では transmitter が recipient がホストするエンドポイントに SET を POST することで通知を行います。

recipient が SET をパースできるかとか issuer とか audience が想定通りかとか署名検証できるかとかそういう検証をしようね、みたいな話とか、エラー時はこういうレスポンスをしようね、みたいな話が書いてあります。

recipient への POST として以下の例が記載されています。

```
POST /Events HTTP/1.1
Host: notify.rp.example.com
Accept: application/json
Accept-Language: en-US, en;q=0.5
Content-Type: application/secevent+jwt

eyJ0eXAiOiJzZWNldmVudCtqd3QiLCJhbGciOiJIUzI1NiJ9Cg
.
eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLmNvbS8iLCJqdGkiOiI3NTZFNjk
3MTc1NjUyMDY5NjQ2NTZFNzQ2OTY2Njk2NTcyIiwiaWF0IjoxNTA4MTg0ODQ1LC
JhdWQiOiI2MzZDNjk2NTZFNzQ1RjY5NjQiLCJldmVudHMiOnsiaHR0cHM6Ly9zY
2hlbWFzLm9wZW5pZC5uZXQvc2VjZXZlbnQvcmlzYy9ldmVudC10eXBlL2FjY291
bnQtZGlzYWJsZWQiOnsic3ViamVjdCI6eyJzdWJqZWN0X3R5cGUiOiJpc3Mtc3V
iIiwiaXNzIjoiaHR0cHM6Ly9pZHAuZXhhbXBsZS5jb20vIiwic3ViIjoiNzM3NT
YyNkE2NTYzNzQifSwicmVhc29uIjoiaGlqYWNraW5nIn19fQ
.
Y4rXxMD406P2edv00cr9Wf3_XwNtLjB9n-jTqN1_lLc
```

# [Poll-Based Security Event Token (SET) Delivery Using HTTP](https://datatracker.ietf.org/doc/html/draft-ietf-secevent-http-poll)

こちらはポーリングで SET を配送する仕様です。
recipient が transmitter に POST リクエストをすることでイベントを取得します。
イベントの取得だけではなく ack の概念があり、POST リクエストで transmitter に受信が完了したイベントを通知できます。transmitter は ack されたイベントを保持する必要がなくなります。

イベントをリクエストしつつ ack もするケースで以下のようなリクエスト例が記載されています。

```json
POST /Events HTTP/1.1
Host: notify.idp.example.com
Content-Type: application/json

{
"ack": [
    "4d3559ec67504aaba65d40b0363faad8",
    "3d0c3cf797584bd193bd0fb1bd4e7d30"
],
"returnImmediately": false
}
```

ack に指定しているのは SET の jti となります。

レスポンス例は以下のようになります。

```json
HTTP/1.1 200 OK
Content-Type: application/json

{
"sets": {
    "4d3559ec67504aaba65d40b0363faad8":
        "eyJhbGciOiJub25lIn0.
        eyJqdGkiOiI0ZDM1NTllYzY3NTA0YWFiYTY1ZDQwYjAzNjNmYWFkOCIsImlhdCI6MTQ
        1ODQ5NjQwNCwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwiYXVkIjpbIm
        h0dHBzOi8vc2NpbS5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MWZhNWJiYzg3OTU5M
        2I3NzU0IiwiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tL0ZlZWRzLzVkNzYwNDUxNmIx
        ZDA4NjQxZDc2NzZlZTciXSwiZXZlbnRzIjp7InVybjppZXRmOnBhcmFtczpzY2ltOmV
        2ZW50OmNyZWF0ZSI6eyJyZWYiOiJodHRwczovL3NjaW0uZXhhbXBsZS5jb20vVXNlcn
        MvNDRmNjE0MmRmOTZiZDZhYjYxZTc1MjFkOSIsImF0dHJpYnV0ZXMiOlsiaWQiLCJuY
        W1lIiwidXNlck5hbWUiLCJwYXNzd29yZCIsImVtYWlscyJdfX19.",
    "3d0c3cf797584bd193bd0fb1bd4e7d30":
        "eyJhbGciOiJub25lIn0.
        eyJqdGkiOiIzZDBjM2NmNzk3NTg0YmQxOTNiZDBmYjFiZDRlN2QzMCIsImlhdCI6MTQ
        1ODQ5NjAyNSwiaXNzIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tIiwiYXVkIjpbIm
        h0dHBzOi8vamh1Yi5leGFtcGxlLmNvbS9GZWVkcy85OGQ1MjQ2MWZhNWJiYzg3OTU5M
        2I3NzU0IiwiaHR0cHM6Ly9qaHViLmV4YW1wbGUuY29tL0ZlZWRzLzVkNzYwNDUxNmIx
        ZDA4NjQxZDc2NzZlZTciXSwic3ViIjoiaHR0cHM6Ly9zY2ltLmV4YW1wbGUuY29tL1V
        zZXJzLzQ0ZjYxNDJkZjk2YmQ2YWI2MWU3NTIxZDkiLCJldmVudHMiOnsidXJuOmlldG
        Y6cGFyYW1zOnNjaW06ZXZlbnQ6cGFzc3dvcmRSZXNldCI6eyJpZCI6IjQ0ZjYxNDJkZ
        jk2YmQ2YWI2MWU3NTIxZDkifSwiaHR0cHM6Ly9leGFtcGxlLmNvbS9zY2ltL2V2ZW50
        L3Bhc3N3b3JkUmVzZXRFeHQiOnsicmVzZXRBdHRlbXB0cyI6NX19fQ."
    }
}
```

sets フィールドの各キーが jti になります。

# [OpenID RISC Profile Specification 1.0 - draft 02](https://openid.net/specs/openid-risc-profile-specification-1_0-02.html)

RISC では以下のイベントが定義されています。

* クレデンシャルの変更が必要になった
* アカウントが削除された
* アカウントが無効化された
* アカウントが有効化された
* Identifier が変わった
* Identifier が再利用され新しいユーザーに紐付いた
* クレデンシャルが漏洩した
* リカバリのフローが有効化した
* リカバリの情報が変わった
* セッションが無効化された
    * deprecated
    * CAEP の session-revoked を使うこと

ユーザーがイベントの送信をオプトイン/オプトアウトできるようにすべきだという記載があり、これに関連するイベントも定義されています。

* オプトインした
* オプトアウトを開始した
* オプトアウトがキャンセルされた
* オプトアウトした

# [OpenID Continuous Access Evaluation Profile 1.0 - draft 02](https://openid.net/specs/openid-caep-specification-1_0.html)

CAEP では以下のイベントが定義されています。

* セッションが無効化された
* トークンの Claims が変わった
* クレデンシャルが変わった
* AAL が変わった
* デバイスの適格/不適格が変わった
