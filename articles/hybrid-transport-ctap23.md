---
title: "CTAP 2.3 Review Draft の Hybrid Transports"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["authentication", "ctap"]
published: false
---

[Digital Identity 技術勉強会 #iddance Advent Calendar 2025](https://qiita.com/advent-calendar/2025/iddance) 23 日目の記事です。

先日の[FIDO 東京セミナー](https://fidoalliance.org/event/fido-tokyo-seminar/?lang=ja)で Hybrid Transport の内容が結構変わっているとを教えてもらったので改めて整理してみます。

以下の内容は [Client to Authenticator Protocol (CTAP)
Review Draft, October 23, 2025](https://fidoalliance.org/specs/fido-v2.3-rd-20251023/fido-client-to-authenticator-protocol-v2.3-rd-20251023.html)を参考にしています。

## [Client to Authenticator Protocol (CTAP) Review Draft, March 21, 2023](https://zenn.dev/inabajunmr/articles/hybrid-transport-ctap) との変更点

- Digital Credential API を前提とした記載が追加された
  - CTAP メッセージだけではなく Hybrid Transport のチャネルで Digital Credential のメッセージもやり取りできる
- メッセージ自体をやり取りする経路として、BLE での接続が追加された
  - Tunnel Service で WebSocket を使って、をせずに BLE だけで全部のやり取りをしてしまうパターンができた
  - 通信経路として 'tunnel service, or use local communication (e.g. Bluetooth Low Energy (BLE), Ultra-wideband (UWB), etc.' とあるので今後なにか増えるのかもしれないが現時点で仕様としては Tunnel Service と BLE のみ記載されている

## ざっくり

この仕様は PC などのクライアントプラットフォームに対してスマートフォンなどの別のデバイスを認証器として使うときの仕様で、それぞれ [QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-qr-initiated) と [State-assisted Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-state-assisted) に分かれています。

### QR-initiated Transactions

![QR-initiated Transactions の図 流れは後述](/images/hybrid23-1.jpg)

- 接続したことがないクライアントプラットフォームと認証器の間で QR コードを介して接続するフロー
- クライアントプラットフォームに QR コードを表示し、それを認証器側のデバイスでスキャンする
- スマートフォンは BLE アドバタイズを行い、ここまでにやり取りした情報を使って双方のデバイスがメッセージのやりとりできるようになる
  - このときの接続は BLE でクライアントプラットフォームとスマートフォンが直接接続するか、tunnel service 経由で WebSocket によって接続する
- この接続を通じて CTAP メッセージのやりとりをすることで、クライアントプラットフォームは認証器にクレデンシャルの作成を命令したり、認証器は作成したアテステーションをクライアントプラットフォームに送信したりすることができる

### State-assisted Transactions

![State-assisted Transactions の図 流れは後述](/images/hybrid23-2.jpg)

- QR-initiated Transactions で接続した際に交換した接続情報を使って QR コードを使わずにやり取りするフロー
- QR-initiated Transactions を一度行ったクライアントプラットフォームと認証器の間で再度やりとりをする際に利用する
- 以前の接続で交換済みの接続情報を使って、クライアントプラットフォームは tunnel service にアクセスする
- tunnel service は認証器にアクセスする
- スマートフォンは BLE アドバタイズを行う
- ここまでに取得した情報を使って双方がハンドシェイクを行い、双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになる
- 以降は QR-initiated Transactions と同様

### tunnel service とは

クライアントプラットフォームが認証器とやりとりするために利用するサービスで、認証器を提供するプラットフォーマーによって提供されています。例えば認証器が Android であれば Google に、iPhone であれば Apple に "cable.ua5v.com" や "cable.auth.com" といったホストで提供されています。仕様上 tunnel service は認証器の提供元が自分でホストすることもできるようになっているようです。

プラットフォームと tunnel service の接続はこの仕様で定義されていますが、認証器と tunnel service 間の通信は各プラットフォームの実装依存となります。こちらは[参考になるものも特になさそう](https://groups.google.com/a/fidoalliance.org/g/fido-dev/c/MGEs6AzPK68/m/1MqFZ5uFAgAJ?utm_medium=email&utm_source=footer)です。

## QR-initiated Transactions の流れ

### QR コード

クライアントプラットフォームは認証器が読み取るための QR コードを表示します。この QR コード以下のフィールドを持つ CBOR にエンコードされています。

| key | value                                                                   | desc                                                                                                                                                                                         |
| --- | ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | 公開鍵                                                                  | トンネル経由でクライアントプラットフォームと認証器がハンドシェイクを行うのに使う                                                                                                             |
| 1   | QR secret                                                               | BLE advertise の暗号化、復号、認証に使う                                                                                                                                                     |
| 2   | クライアントプラットフォームが知ってる tunnel service のドメインの数 ※1 | これが 2 だったらこのプラットフォームは `cable.ua5v.com` と `cable.auth.com` を知っていることになり、後述の BLE advert で 0 を指定すれば前者に、1 を指定すれば後者を利用することが表現できる |
| 3   | 現在時刻                                                                |
| 4   | プラットフォームが state-assisted transactions に対応しているかどうか   |
| 5   | 今後行われる操作についてのヒント                                        |                                                                                                                                                                                              |
| 6   | トランスポートに使われるチャネル                                        | WebSocket or Bluetooth Low Energy                                                                                                                                                            |

ヒントには以下のものがあります。
| key | value |
| --- | --- |
| ga | getAssertion (FIDO2) |
| mc | makeCredential (FIDO2) |
| dcp | credential presentation (Digital Credentials API) |
| dci | credential issuance (Digital Credentials API) |

### 認証器による QR コードの読み取りと BLE advert

クライアントプラットフォームは認証器からの接続を待機し、認証器は BLE advertise を行います。これは認証器がクライアントプラットフォームの近くにあることを担保するために行われます。
これによって攻撃者が自分のページに、あるサイトに対して認証を行うための QR コードを表示してユーザーがそれをスキャンした場合、攻撃者の PC と被害者のスマートフォンが tunnel service を通じて接続されます。このとき、被害者のスマートフォンが発行した RP 向けのアサーションやアテステーションを攻撃者が取得してしまう、といった状況を回避します。

この BLE advert は暗号化されていて、認証、復号のための値は [HKDF](https://www.rfc-editor.org/rfc/rfc5869) を使って QR secret から導出されます。

BLE advert には以下のフィールドがあります。

| value                     | desc                                       |
| ------------------------- | ------------------------------------------ |
| nonce                     |                                            |
| routing ID                | tunnel service に接続する際にわたす        |
| tunnel service identifier | 利用する tunnel service を決定するための値 |
| advertisement suffix      | 追加情報を CBOR で格納できる               |

チャネルが BLE の場合、ペイロードの末尾に advertisement suffix という CBOR が入ります。この CBOR は BLE の場合以下となります。

| key                             | value                     | desc                                              |
| ------------------------------- | ------------------------- | ------------------------------------------------- |
| 1(transport_channel_identifier) | server PSM(channel_extra) | client platform が Authenticator に接続ために使う |

これら値によって通信経路の接続に必要な情報が交換されます。

### データ送信チャネルの決定

QR コードにはトランスポートに利用できるチャネルの情報が含まれており、認証器は自身がサポートしているチャネルと QR コードで通知されたチャネルで共通のものの中から適切なものを選択します。
[11.5.1.1. Data transfer channel](https://fidoalliance.org/specs/fido-v2.3-rd-20251023/fido-client-to-authenticator-protocol-v2.3-rd-20251023.html#hybrid-data-transfer-channel) には WebSocket(Tunnel Service) はフォールバックもしくは後方互換性のためにサポートすべきである、という記述があるため、今後は BLE が主流になっていくのかもしれません。

### WebSocket による接続

#### tunnel service の決定

tunnel service identifier によって利用する tunnel service が決定されます。
この値が 256 未満の場合、well known な tunnel service を利用することを意味します。
つまり、0 であれば `cable.ua5v.com`、1 であれば `cable.auth.com` が利用されます。

256 以上の場合、この値をあるルール（導出のための Go のコードが仕様に記載されています）でハッシュ化した値がドメインとして利用されます。
例えば 256 であれば `cable.qz2ekwmnd332c.info`、257 であれば `cable.4a6bwmj6hiyyd.net` のようになります。任意の tunnel service を利用する認証器を実装する場合、適当な数字で空いているドメインを取得してホストすればよいのではないでしょうか。

#### tunnel service への接続

クライアントプラットフォームが tunnel service へ接続する準備ができたので、接続します。
まず tunnel ID を導出します。これは tunnel service のコネクションを一意に特定する ID のようなものです。この値も QR secret から導出されます。

その後以下のような URL で WebSocket によって tunnel service への接続を行います。

```
wss://cable.example.com/cable/connect/{routing id}/{tunnel id}
```

WebSocket の subprotocol identifier は `fido.cable` となります。このとき、認証器側も tunnel service に接続しに行きますがこちらの接続については具体的な記述がありません。認証器と tunnel service の提供元が同じなので独自の仕組みで認証したりするんでしょうか。

この時点でトンネルが確立し、クライアントプラットフォームと認証器は WebSocket フレーム でのやり取りができるようになります。

### BLE による接続

QR コード経由で BLE がトランスポートに利用可能なことが通知された場合、認証器は insecure L2CAP Connection-oriented Channel (CoC) Bluetooth server socket を作ります。
ここでこのチャンネルの識別子である server PSM を生成し、advertisement suffix に追加して Client Platform に L2CAP の接続ができることを通知します。
client platform が BLE advert から server PSM をパースして利用することでソケットに接続し、以降このチャネル上でメッセージのやり取りを行うことができます。

この BLE の接続は optional であり、フォールバックのために WebSocket での通信もサポートすべきとのことです。

> Bluetooth Low Energy L2CAP and extended advertising are optional features and may not be available on all devices. Implementations SHOULD always include websockets as a fallback.

### ハンドシェイク

トンネルを通じてクライアントプラットフォームと認証器は Noise KNpsk0 暗号のハンドシェイクを行います。
このとき、クライアントプラットフォームが BLE advert を受信したことを証明するためまず最初にハンドシェイクのメッセージを送信し、認証器はそれにレスポンスを返します。

[KNpsk0](http://www.noiseprotocol.org/noise.html#pattern-modifiers) は以下の定義になっています。

```
-> s
...
-> psk, e
<- e, ee, se
```

- s
  - QR コード経由で共有したクライアントプラットフォームの公開鍵
- psk
  - QR secret と復号した BLE advert から導出
- e
  - 双方が動的に生成する鍵

PSK で認証しながら「e 同士」「クライアントプラットフォームの公開鍵と e」 でそれぞれ鍵交換を行い、これらの鍵から通信に利用する鍵を導出します。
以降のメッセージはハンドシェイクで導出した鍵によって暗号化されます。

認証器側からの最初のメッセージは `post handshake` で、これには [getInfo](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorGetInfo) のレスポンス、つまり認証器の情報（aaguid とか対応してるアルゴリズムとかトランスポートとか）と、認証器がサポートしている機能（ctap / dc）が含まれています。dc は Digital Credential のサポートを表します。

これ以降のメッセージにはメッセージの type が含まれるようになり、CTAP のコマンドは CTAP message type のメッセージによって送信されます。

| key | type               | desc                                                                                                 |
| --- | ------------------ | ---------------------------------------------------------------------------------------------------- |
| 0   | shutdown           | クライアントから認証器にのみ送られる <br> クライアントがこれ以上 CTAP メッセージを送らないことを表す |
| 1   | CTAP               | CTAP2 payload                                                                                        |
| 2   | update             | 双方から送られるが、現状認証器からクライアントプラットフォームに向けての値のみ定義されている         |
| 3   | JSON-based message | Digital Credential                                                                                   |

クライアントプラットフォームは CTAP2 コマンドを認証器に送って何らかのアクションを送ります。getInfo はすでに受信しているので例えば登録の場合 [authenticatorMakeCredential](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorMakeCredential) を送ると認証器がクレデンシャルを作成してアテステーションをレスポンスしてくるのでそれを使ってよしなに登録処理を進めるといった感じになります。

#### update メッセージ

update メッセージには CBOR で linking information が含まれていて、この linking information は以下となっています。

| key | value                | desc                                                                                                                                                                                                    |
| --- | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | contact ID           | tunnel service が認証器を特定するのに使う <br> State-assisted Transactions で クライアントプラットフォームがこの値を指定して tunnel service にアクセスする <br> Android の場合は FCM Registration token |
| 2   | link ID              | この link を認証器が識別する値                                                                                                                                                                          |
| 3   | link secret          | 共通鍵                                                                                                                                                                                                  |
| 4   | 認証器の公開鍵       |                                                                                                                                                                                                         |
| 5   | 認証器の名前         | ユーザーが hybrid を選択したときに選択する `Pixel 6a` みたいなやつ                                                                                                                                      |
| 6   | ハンドシェイクの署名 | linking information に含まれる公開鍵の保持を証明するための署名                                                                                                                                          |

これらの情報は State-assisted Transactions で利用されます。

## State-assisted Transactions の流れ

State-assisted Transactions では認証器との接続に以前 QR-initiated Transactions で交換した情報を使うので QR の読み取りを利用しません。

仕様を見たところ WebSocket を前提としたやり取りしか記載されていなかったので、BLE の場合毎回 QR コードのスキャンが必要になるのかもしれませんが Draft なのでまだよくわかりません。
この場合 BLE advert をするきっかけがないので Authenticator が常時 advert し続ける必要な気がしますがそれが難しいとかそういう話なのでしょうか。（BLE のことがわかっていないので滅茶苦茶なことを言っているかもしれません）

### tunnel service への接続

クライアントプラットフォームは認証器と紐づけてドメインと contact id を持っておき、このエンドポイント経由での認証器への接続を試みます。

```
wss://cable.example.com/cable/contact/${contact id}
```

一方認証器はどのクライアントプラットフォームと接続するかを知るための link ID とクライアントプラットフォームによる nonce を必要とします。これらの値は `X-caBLE-Client-Payload` ヘッダ経由でクライアントプラットフォームから送信されます。ただしこちらについても認証器側の具体的な接続手順は仕様外です。

トンネルに接続できたら認証器はハンドシェイクメッセージを送信します。

### 認証器による BLE advert

この後認証器は近接性の検証のため BLE advert を行います。この BLE advert は link secret と tunnel service 経由で認証器に送信した nonce によって復号できます。

### ハンドシェイク

クライアントプラットフォームは link secret と BLE advert に含まれる nonce からハンドシェイクに使う PSK を導出します。
PSK および linking information に含まれる公開鍵を使ってハンドシェイクを行います。この際のハンドシェイクは [NKpsk0](http://www.noiseprotocol.org/noise.html#pattern-modifiers) となります。

```
<- s
...
-> psk, e, es
<- e, ee
```

- s
  - update message で共有した認証器の公開鍵
- psk
  - link secret と復号した BLE advert から導出
- e
  - 双方が動的に生成する鍵

ハンドシェイクが完了したらあとは QR-initiated Transactions と同様に CTAP メッセージのやりとりを行ってアテステーションやアサーションの作成を依頼します。

## シーケンス

値が色々出てくるのでシーケンスをまとめます。通信方法に関わらず矢印は「データを送る側->データを受け取る側」です。

### QR-initiated Transactions

#### メッセージの通信経路の確立まで

![QR-initiated Transactions のシーケンス1](/images/hybrid23-sequence1.png)

<!-- ![](//www.plantuml.com/plantuml/png/LP2nJiCm48RtUufJT_3U0JL4R20578dvwXnWuTYHVGv85GkTU1vMtYOlmj4Cg8j_s_f-tQVR5Q4iTGmmQNhd9ug2cpPurkm2oLFAumQfODkTCqsL5uxw9advH3JdG5zZv82My-mTduZU0bL9vCJF98mf0hGTNbnXWrkyVy3bytiv_Yp7ByWiDSSjNj_U80qpPm4AWe-yjycziW3YskojLjzAsHhZQ1_uajzfd3HT6jSVuvAQE367dAhu-8n--307MVtq3XmA_qq2n7-TekASiRDtm760dHwwg5y0) -->

##### 1. QR コード

**Client Platform -> Authenticator**

| value                                                                | 出どころ                           |
| -------------------------------------------------------------------- | ---------------------------------- |
| クライアントプラットフォームの公開鍵                                 | クライアントプラットフォームが保持 |
| QR secret                                                            | クライアントプラットフォームが生成 |
| クライアントプラットフォームが知ってる tunnel service のドメインの数 | クライアントプラットフォームが保持 |
| クライアントプラットフォームがサポートしているメッセージの通信経路   | クライアントプラットフォームが保持 |

##### 2. BLE advert

**Client Platform <- Authenticator**

BLE advert の復号は QR secret から導出した値を使って行う。

| value                     | 出どころ     |
| ------------------------- | ------------ |
| routing ID                | 認証器が決定 |
| tunnel service identifier | 認証器が保持 |
| server PSM                | 認証器が生成 |

##### 3-A. Tunnel Service への接続(WebSocket の場合)

**Client Platform -> Tunnel Service**

| value        | 出どころ                                                                                                                           |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| 接続先ホスト | BLE advert で受け取った tunnel service identifier とクライアントプラットフォームが知っている tunnel service のホストの一覧から判定 |
| routing ID   | BLE advert で受け取った値                                                                                                          |
| tunnel id    | QR secret から導出                                                                                                                 |

##### 3-B. L2CAP CoC の確立(BLE の場合)

**Client Platform -> Authenticator**

| value      | 出どころ                           |
| ---------- | ---------------------------------- |
| server PSM | BLE advert で受け取った server PSM |

#### WebSocket / L2CAP CoC の確立後

![QR-initiated Transactions のシーケンス2](/images/hybrid23-sequence2.png)

<!-- ![](//www.plantuml.com/plantuml/png/RL5DIyD04BtdLmmzwyLZ3jAGYuA7WFq3IxDjWkbkIJRjrTs8Y5Kg5PI2A2gsWg285AlrtqnjwxzmuaUXwENDphwtZtcpnKInMAMroAfJ3SjXdGa51JSAELKlOgeYure1M0AkjwXnKXLmGJrJClvIE1PBbMHb5JQOuY259MHU6pm6PuaCR1YFRZwwXwZlqUoWsNzldn2YVe1IAWIFO9F7ZR3C0T1qngCMwYuQmXPkwuqLr_70bIwCX_IaFxiyGDD6GzYoBDU3vLLmm8Or9lmaO5iSQZn9M9LRCLTfBFwZe1cg0AfShOkA11fhEZYQd9zPJcv6bZQaTP-fkVvl8DJ7UHezH4E7FhWbn_jL4tc7PW_rDUeUcYiq0ypUq3nZriVi2VKj9SllirqqBuGBhEdCUdhoQ52VehwWDKsxhvHvtdxlvYUS3KcGAbysnNpK8XS-VUmd) -->

##### 1. Handshake message

**Client Platform -> (Tunnel Service) -> Authenticator**

| value         | 出どころ                                 |
| ------------- | ---------------------------------------- |
| PSK           | QR secret と復号した BLE advert から導出 |
| ephemeral key | ランダム生成                             |

##### 2. Handshake message with getInfo の結果

**Client Platform <- (Tunnel Service or L2CAP CoC) <- Authenticator**

| value          | 出どころ     |
| -------------- | ------------ |
| ephemeral key  | ランダム生成 |
| getInfo の結果 | 認証器が保持 |

この時点で双方が通信に使う鍵が共有される。

##### 3. update message

**Client Platform <- (Tunnel Service or L2CAP CoC) <- Authenticator**

このメッセージは仕様上タイミングが定義されていないので CTAP message のあとかも

| key                  | 出どころ             |
| -------------------- | -------------------- |
| contact ID           | 認証器が決定？       |
| link ID              | 認証器が決定？       |
| link secret          | 認証器が生成？       |
| 認証器の公開鍵       | 認証器が保持         |
| ハンドシェイクの署名 | 認証器の秘密鍵で署名 |

##### 4. shutdown message

**Client Platform -> (Tunnel Service) -> Authenticator**

これを受けて認証器は接続を切る。（切らなくてもいい）

### State-assisted Transactions

#### Tunnel Service への接続まで

![State-assisted Transactions のシーケンス1](/images/hybrid3.png)

<!-- ![](https://cdn-0.plantuml.com/plantuml/png/LO-zQiGm381tFOK8NLll6KfIqws38Na1nL6fmJ_1bjE3uzxzVBbRCdtInwT1Gn7AKeE7hT5Pjr4KxBHtt6WyoM_AeKCggCsv6QlySMmxISf7CPw3kSR87YVEkxDy5FC4G5LIh67X3A0DldysYpt-bz8hPMdn_C4N2bkZJU5fb4rHo8fwkxucTEiDniUrDZr-_NmZhJjd0HWuhksXEm00) -->

##### 1. wss://cable.example.com/cable/contact/${contact id}

**Client Platform -> Tunnel Service**

| value      | でどころ                              |
| ---------- | ------------------------------------- |
| contact id | 以前の接続時に受信した update message |
| link id    | 以前の接続時に受信した update message |
| nonce      | ランダムに生成                        |

##### 2. BLE advert

**Authenticator Client Platform**

BLE advert の復号は link secret と 1 で送信した nonce から導出した鍵によって行う。

| value | でどころ       |
| ----- | -------------- |
| nonce | ランダムに生成 |

#### Tunnel Service への接続後

![State-assisted Transactions のシーケンス4](/images/hybrid4.png)

<!-- ![](https://cdn-0.plantuml.com/plantuml/png/RL7DIiD06BpdAJvwrXVeGHfoqLl0li3QR9jWiYd9Rdl-DYBIAae5fI2AYgqWL4IAF_LjljhMjt0Z4Yjw-RwTdPbbXgsaaYLIHQ7LD3ke1Kqc99ISCE1cko6JzJY9Ii1ISDpV6bj9dmk3cOoJBuLjCILZeQ8jdbWYbXpY5a_0NZ7UCamdDn_z0y8HwHtIdVV34pC9xw6gXE0XRFAa5TGuaEvD9rt2FkckPJakhAw13MLOPy9BVRRD8U9foXhSidFASV_Ze1r87nfSReXw38IZ-VXeTdwsP55DZyb2_pSkuN6U5Pz9cyCGdAhj5gyJtf7U8xmc792-8Zscx8tq4sL3oXu9Rmcx-NssI_ebdykixYq6E7lGXAU45wGxy_xhudA_WD_LVvgdlf9Ut1P5s0eNdlcpJm00) -->

##### 1. Handshake message

**Client Platform -> Authenticator**

| value         | でどころ                            |
| ------------- | ----------------------------------- |
| PSK           | link secret と複合した BLE から導出 |
| ephemeral key | ランダム生成                        |

##### 2. Handshake message with getInfo の結果

**Client Platform -> Authenticator**

| value          | 出どころ     |
| -------------- | ------------ |
| ephemeral key  | ランダム生成 |
| getInfo の結果 | 認証器が保持 |

この時点で双方が通信に使う鍵が共有される。

## 参考資料

- [Client to Authenticator Protocol (CTAP) Review Draft, March 21, 2023](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#sctn-hybrid)
- [Client to Authenticator Protocol (CTAP) Review Draft, October 23, 2025](https://fidoalliance.org/specs/fido-v2.3-rd-20251023/fido-client-to-authenticator-protocol-v2.3-rd-20251023.html)
- [The Noise Protocol Framework](http://www.noiseprotocol.org/noise.html)
- [携帯電話の Passkey はもう使えるメモ](https://zenn.dev/okuoku/scraps/35d81e1337262f)
