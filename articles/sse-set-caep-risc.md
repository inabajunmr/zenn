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

といった文脈でステータスの変更を通知するのに [CAEP]() が出てきたり、

* ユーザーは同じメールアドレスを複数サービスに登録したりする
* 攻撃者はこれを利用して、1 つの脆弱なポイントからアカウント乗っ取りを連鎖したりする
    * メールアカウントが乗っ取られたら別サービスのパスワードリカバリ情報がそのメールアカウントに送られる、みたいなケース

といった文脈で [RISC]() について記載があります。

ざっくりまとめると、各ユーザーやサービス、その他色々のセキュリティに関連するイベントをリアルタイムにやりとりして評価したいけどどうやる？のための仕様がいろいろある、といった感じです。

これらを実現するためにいくつかのプロトコルが出てくるのですが、本記事では各プロトコルにどのような仕様が定義されているのかをざっくり解説します。

ちなみに CAEP と RISC の立ち位置の差はよくわかってないので、それについては特に何も書いてないです。

# どのようなシーケンスになるのか

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

仕様では以下のようなリクエストの例が記載されています。

#### ステータス参照のリクエスト

```
GET /sse/status HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer zzzz
```

#### ステータス参照のレスポンス

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "status": "enabled"
}
```

#### Subject を指定したステータス参照のリクエスト

Subject ごとのステータスがあるようで、特定の Subject のステータスを取得する場合以下のようになります。

```
GET /sse/status?subject=<url-encoded-subject> HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
```

#### Subject を指定したステータス参照のレスポンス

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "status": "enabled",
  "subject": {
    "tenant" : {
      "format" : "iss_sub",
      "iss" : "http://example.com/idp1",
      "sub" : "1234"
    }
  }
}
```

#### ステータス更新のリクエスト

```
POST /sse/status HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "status": "paused"
}
```

#### Subject を指定したステータス参照のリクエスト

特定の Subject を指定してステータスを更新することもできます。他に reason フィールドが定義されています。

```
POST /sse/status HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "status": "paused",
  "subject": {
    "tenant" : {
      "format" : "iss_sub",
      "iss" : "http://example.com/idp1",
      "sub" : "1234"
    }
  },
  "reason": "Disabled by administrator action."
}
```

#### ステータス更新のレスポンス

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "status": "paused"
}
```

### Configuration Endpoint

利用するイベントの通知方法や、対象とするイベントの種類などのストリームの設定を取得、更新できます。

#### 設定参照のリクエスト

```
GET /sse/stream HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
```

#### 設定参照のレスポンス

ストリームで利用するイベントの通知方法や、どのようなイベントが通知されるのか、などの情報が返ってきます。

```json
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "iss": "https://tr.example.com",
  "aud": [
      "http://receiver.example.com/web",
      "http://receiver.example.com/mobile"
    ],
  "delivery": {
    "delivery_method":
      "https://schemas.openid.net/secevent/risc/delivery-method/push",
      "url": "https://receiver.example.com/events"
  },
  "events_supported": [
    "urn:example:secevent:events:type_1",
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3"
  ],
  "events_requested": [
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3",
    "urn:example:secevent:events:type_4"
  ],
  "events_delivered": [
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3"
  ]
}
```

#### 設定更新のリクエスト

```
POST /sse/stream HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "iss": "https://tr.example.com",
  "aud": [
    "http://receiver.example.com/web",
    "http://receiver.example.com/mobile"
  ],
  "delivery": {
    "delivery_method":
      "https://schemas.openid.net/secevent/risc/delivery-method/push",
      "url": "https://receiver.example.com/events"
  },
  "events_requested": [
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3",
    "urn:example:secevent:events:type_4"
  ]
}
```

#### 設定参照のレスポンス

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "iss": "https://tr.example.com",
  "aud": [
    "http://receiver.example.com/web",
    "http://receiver.example.com/mobile"
  ],
  "delivery": {
    "delivery_method":
      "https://schemas.openid.net/secevent/risc/delivery-method/push",
      "url": "https://receiver.example.com/events"
  },
  "events_supported": [
    "urn:example:secevent:events:type_1",
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3"
  ],
  "events_requested": [
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3",
    "urn:example:secevent:events:type_4"
  ],
  "events_delivered": [
    "urn:example:secevent:events:type_2",
    "urn:example:secevent:events:type_3"
  ]
}
```

### 設定削除のリクエスト

```
DELETE /sse/stream HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=
```

### Add Subject Endpoint/Remove Subject Endpoint

前述の Subject をストリームに追加するための API が定義されています。

#### Subject 追加のリクエスト

```
POST /sse/subjects:add HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "subject": {
    "format": "email",
    "email": "example.user@example.com"
  },
  "verified": true
}
```

#### Subject 追加のレスポンス

```
HTTP/1.1 200 OK
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache
```

#### Subject 削除のリクエスト

```
POST /sse/subjects:remove HTTP/1.1
Host: transmitter.example.com
Authorization: Bearer eyJ0b2tlbiI6ImV4YW1wbGUifQo=

{
  "subject": {
    "format": "phone",
    "phone_number": "+1 206 555 0123"
  }
}
```

#### Subject 削除のレスポンス

```
HTTP/1.1 204 No Content
Server: transmitter.example.com
Cache-Control: no-store
Pragma: no-cache
```

### Verification Endpoint

イベントストリームを流れるイベントは非常に少ない場合があるため、実際にイベントが通知された場合の挙動を検証するための Verification Event が定義されています。これを送信するために Verification Endpoint が定義されています。

## 認証について

Transmitter の API 呼び出しには OAuth 2.0 のアクセストークンを使った認可を行うべき、とあります。

# [Security Event Token (SET)](https://datatracker.ietf.org/doc/html/rfc8417)

TODO