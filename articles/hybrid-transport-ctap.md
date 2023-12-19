---
title: "CTAP2.2 の Hybrid transports"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["authentication", "ctap"]
published: false
---

[Digital Identity技術勉強会 #iddance Advent Calendar 2023](https://qiita.com/advent-calendar/2023/iddance) 21 日目の記事です。

[Client to Authenticator Protocol (CTAP) Review Draft, March 21, 2023 の 11.5. Hybrid transports](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#sctn-hybrid) に書いてある内容を整理しました。

## ざっくり

この仕様はPC などのクライアントプラットフォームに対してスマートフォンなどの別のデバイスを認証器として使うときの仕様で、それぞれ [QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-qr-initiated) と [State-assisted Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-state-assisted) に分かれています。

### QR-initiated Transactions

![](/images/qr-initiated.png)

* 接続したことがないクライアントプラットフォームと認証器の間で QR コードを介して接続するフロー
* クライアントプラットフォームに QR コードを表示し、それを認証器側のデバイスでスキャンする
* スマートフォンは BLE アドバタイズを行い、ここまでにやり取りした情報を使って双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになる
* この接続を通じて CTAP メッセージのやりとりをすることで、クライアントプラットフォームは認証器にクレデンシャルの作成を命令したり、認証器は作成したアテステーションをクライアントプラットフォームに送信したりすることができる

### State-assisted Transactions

![](/images/state-assisted.png)

* QR-initiated Transactions で接続した際に交換した接続情報を使って QR コードを使わずにやり取りするフロー
* QR-initiated Transactions を一度行ったクライアントプラットフォームと認証器の間で再度やりとりをする際に利用する
* 以前の接続で交換済みの接続情報を使って、クライアントプラットフォームは tunnel service にアクセスする
* tunnel service は認証器にアクセスする
* スマートフォンは BLE アドバタイズを行う
* ここまでに取得した情報を使って双方がハンドシェイクを行い、双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになる
* 以降は QR-initiated Transactions と同様

### tunnel service とは

クライアントプラットフォームが認証器とやりとりするために利用するサービスで、認証器を提供するプラットフォーマーによって提供されています。例えば認証器が Android であれば Google に、iPhone であれば Apple に "cable.ua5v.com" や "cable.auth.com" といったホストで提供されています。仕様上 tunnel service は認証器の提供元が自分でホストすることもできるようになっているようです。

プラットフォームと tunnel service の接続はこの仕様で定義されていますが、認証器と tunnel service 間の通信は各プラットフォームの実装依存となります。こちらは[参考になるものも特になさそう](https://groups.google.com/a/fidoalliance.org/g/fido-dev/c/MGEs6AzPK68/m/1MqFZ5uFAgAJ?utm_medium=email&utm_source=footer)です。

## QR-initiated Transactions の流れ

### QR コード

クライアントプラットフォームは認証器が読み取るための QR コードを表示します。この QR コード以下のフィールドを持つ CBOR にエンコードされています。

| key | value | desc |
| --- | ---- | --- |
| 0 | 公開鍵 | トンネル経由でクライアントプラットフォームと認証器がハンドシェイクを行うのに使う |
| 1 | QR secret | BLE advertise の暗号化、復号、認証に使う |
| 2 | クライアントプラットフォームが知ってる tunnel service のドメインの数 ※1 | これが 2 だったらこのプラットフォームは `cable.ua5v.com` と `cable.auth.com` を知っていることになり、後述の BLE advert で 0 を指定すれば前者に、1 を指定すれば後者を利用することが表現できる |
| 3 | 現在時刻 |
| 4 | プラットフォームが state-assisted transactions に対応しているかどうか |
| 5 | 今後行われる操作についてのヒント（認証 or 登録） |

* ※1 なぜこういうフォーマットなのかよくわからない
* ※1 `cable.ua5v.com` だけ知ってる、はできるのに `cable.auth.com` しか知らないプラットフォームを作れないように見えるのがなんか変な気がする
* ※1 wellknown なものは各プラットフォームが全部サポートするのが前提なんだろうか

### 認証器による QR コードの読み取りと BLE advert

クライアントプラットフォームは認証器からの接続を待機し、認証器は BLE advertise を行います。これは認証器がクライアントプラットフォームの近くにあることを担保するために行われます。
これによって攻撃者が自分のページに、あるサイトに対して認証を行うための QR コードを表示してユーザーがそれをスキャンした場合、攻撃者の PC と被害者のスマートフォンが tunnel service を通じて接続されます。このとき、被害者のスマートフォンが発行した RP 向けのアサーションやアテステーションを攻撃者が取得してしまう、といった状況を回避します。

この BLE advert は暗号化されていて、認証、復号のための値は [HKDF](https://www.rfc-editor.org/rfc/rfc5869) を使って QR secret から導出されます。

BLE advert には以下のフィールドがあります。

| value | desc |
| ----- | ---- |
| nonce | |
| routing ID | tunnel service に接続する際にわたす |
| tunnel service identifier | 利用する tunnel service を決定するための値 |

この値によって利用する tunnel service が決定されます。

### tunnel service の決定

tunnel service identifier によって利用する tunnel service が決定されます。
この値が 256 未満の場合、well known な tunnel service を利用することを意味します。
つまり、0 であれば `cable.ua5v.com`、1 であれば `cable.auth.com` が利用されます。

256 以上の場合、この値をあるルール（導出のための Go のコードが仕様に記載されています）でハッシュ化した値がドメインとして利用されます。
例えば 256 であれば `cable.qz2ekwmnd332c.info`、257 であれば `cable.4a6bwmj6hiyyd.net` のようになります。任意の tunnel service を利用する認証器を実装する場合、適当な数字で空いているドメインを取得してホストすればよいのではないでしょうか。

### tunnel service への接続

クライアントプラットフォームが tunnel service へ接続する準備ができたので、接続します。
まず tunnel ID を導出します。これは tunnel service のコネクションを一意に特定する ID のようなものです。この値も QR secret から導出されます。

その後以下のような URL で WebSocket によって tunnel service への接続を行います。

```
wss://cable.example.com/cable/connect/{routing id}/{tunnel id}
```

WebSocket の subprotocol identifier は `fido.cable` となります。このとき、認証器側も tunnel service に接続しに行きますがこちらの接続については具体的な記述がありません。認証器と tunnel service の提供元が同じなので独自の仕組みで認証したりするんでしょうか。

この時点でトンネルが確立し、クライアントプラットフォームと認証器は WebSocket フレーム でのやり取りができるようになります。

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

* s
    * QR コード経由で共有したクライアントプラットフォームの公開鍵
* psk
    * QR secret と復号した BLE advert から導出
* e
    * 双方が動的に生成する鍵

PSK で認証しながら「e 同士」「クライアントプラットフォームの公開鍵と e」 でそれぞれ鍵交換を行い、これらの鍵から通信に利用する鍵を導出します。
以降のメッセージはハンドシェイクで導出した鍵によって暗号化されます。

認証器側からの最初のメッセージは `post handshake` で、これには [getInfo](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorGetInfo) のレスポンス、つまり認証器の情報（aaguid とか対応してるアルゴリズムとかトランスポートとか）が含まれています。

これ以降のメッセージにはメッセージの type が含まれるようになり、CTAP のコマンドは CTAP message type のメッセージによって送信されます。

| key | type | desc |
| --- | ---- | ---- |
| 0 | shutdown | クライアントから認証器にのみ送られる <br> クライアントがこれ以上 CTAP メッセージを送らないことを表す |
| 1 | CTAP | CTAP2 payload |
| 2 | update | 双方から送られるが、現状認証器からクライアントプラットフォームに向けての値のみ定義されている |

クライアントプラットフォームは CTAP2 コマンドを認証器に送って何らかのアクションを送ります。getInfo はすでに受信しているので例えば登録の場合 [authenticatorMakeCredential](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorMakeCredential) を送ると認証器がクレデンシャルを作成してアテステーションをレスポンスしてくるのでそれを使ってよしなに登録処理を進めるといった感じになります。

#### update メッセージ

update メッセージには CBOR で linking information が含まれていて、この linking information は以下となっています。

| key | value | desc |
| ----- | ---- | --- |
| 1 | contact ID | tunnel service が認証器を特定するのに使う <br> State-assisted Transactions で クライアントプラットフォームがこの値を指定して tunnel service にアクセスする <br> Android の場合は FCM Registration token|
| 2 | link ID | この link を認証器が識別する値 |
| 3 | link secret | 共通鍵 |
| 4 | 認証器の公開鍵 | |
| 5 | 認証器の名前 | ユーザーが hybrid を選択したときに選択する `Pixel 6a` みたいなやつ |
| 6 | ハンドシェイクの署名 | linking information に含まれる公開鍵の保持を証明するための署名 |

これらの情報は State-assisted Transactions で利用されます。

## State-assisted Transactions の流れ

State-assisted Transactions では認証器との接続に以前 QR-initiated Transactions で交換した情報を使うので QR の読み取りを利用しません。

### tunnel service への接続

クライアントプラットフォームは認証器と紐づけてドメインと contact id を持っておき、このエンドポイントへの接続を試みます。

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

* s
    * update message で共有した認証器の公開鍵
* psk
    * link secret と復号した BLE advert から導出
* e
    * 双方が動的に生成する鍵

ハンドシェイクが完了したらあとは QR-initiated Transactions と同様に CTAP メッセージのやりとりを行ってアテステーションやアサーションの作成を依頼します。


## シーケンス

値が色々出てくるのでシーケンスをまとめます。通信方法に関わらず矢印は「データを送る側->データを受け取る側」です。

### QR-initiated Transactions

#### Tunnel Service への接続まで

![](https://cdn-0.plantuml.com/plantuml/png/LO-_JiGm38VtF8LrEsI_0Lses1WATo_WImr4IfmgSGwa4mkTU1vMtgOlWccdJZ_xqoV_ELJ18Yr5Cse67qPaWLqN0sds4UKbbxG3hD3rMySrUIFM7YMNnN1RuTIOASAHoYLuMepJqM2Jp2sTgHZJzN1p1mxsyFGCyzVFFFtEqxTnIdMTull71y3XGaLMLmSeVQzrRwt7SwHR-i0qQlgSLc9zPYOlzbfoay2l48PFUvNr6AsDEH0F_o__0G00)

##### 1. QR コード

:::message
Client Platform -> Authenticator
:::

| value | 出どころ |
| ---- | - |
| クライアントプラットフォームの公開鍵 | クライアントプラットフォームが保持 |
| QR secret | クライアントプラットフォームが生成 |
| クライアントプラットフォームが知ってる tunnel service のドメインの数 | クライアントプラットフォームが保持 |

##### 2. BLE advert

:::message
Client Platform <- Authenticator
:::

BLE advert の復号は QR secret から導出した値を使って行う。

| value | 出どころ |
| ----- | - |
| routing ID | 認証器が決定 |
| tunnel service identifier | 認証器が保持 |

##### 3. wss://cable.example.com/cable/connect/{routing id}/{tunnel id}

:::message
Client Platform -> Tunnel Service
:::

| value | 出どころ |
| ----- | - |
| 接続先ホスト | BLE advert で受け取った tunnel service identifier とクライアントプラットフォームが知っている tunnel service のホストの一覧から判定 |
| routing ID | BLE advert で受け取った値 |
| tunnel id | QR secret から導出 |

#### Tunnel Service への接続後

![](https://cdn-0.plantuml.com/plantuml/png/RL7DIiD06BplKtpqh2_GWpJaehU0VO6rsJR1P5EIJNl-DYBIAae5fI2AYgqWL4IAF_LjljhMjt0Z4Yjw-RwTdPbbXgqaYiSg3GFMDDkl-Kqk5PJim1TcEm5NzIWEIy0Ji9tV6YjLdf06SnN5NmgByLH5CWstHCnaf0H4BH63jMAyPPXERZxw1uGZqZkaE--79sOItaCrbL84i2dYbbyJGBetdNG9-wIxZDaEhAw11MLOvz9DFBuj81H9mXk2MJbbE_znqFQL1msXDcGz0iBHV7mqEpzRigHDbwj2_pSkuN6U5Pz9cyCGxAhb5AyJtf7U8xmc792-8Zscx8tq4sL3oXu9Rmcx-NssI_ebdykixYq6E7lGXAU45wGxy_xhudA_WD_LVvedNghSg2sBi8nLX7JDhtq2)

##### 1. Handshake message

:::message
Client Platform -> (Tunnel Service) -> Authenticator
:::

| value | 出どころ |
| ----- | --- |
| PSK | QR secret と復号した BLE から導出 |
| ephemeral key | ランダム生成 |

##### 2. Handshake message with getInfo の結果

:::message
Client Platform <- (Tunnel Service) <- Authenticator
:::

| value | 出どころ |
| ----- | --- |
| ephemeral key | ランダム生成 |
| getInfo の結果 | 認証器が保持 |

この時点で双方が通信に使う鍵が共有される。

##### 3. update message

:::message
Client Platform <- (Tunnel Service) <- Authenticator
:::

このメッセージは仕様上タイミングが定義されていないので CTAP message のあとかも

| key | 出どころ |
| ----- | --- |
| contact ID | 認証器が決定？ |
| link ID | 認証器が決定？ |
| link secret | 認証器が生成？ |
| 認証器の公開鍵 | 認証器が保持 |
| ハンドシェイクの署名 | 認証器の秘密鍵で署名 |

##### 4. shutdown message

:::message
Client Platform -> (Tunnel Service) -> Authenticator
:::

これを受けて認証器は接続を切る。（切らなくてもいい）

### State-assisted Transactions

#### Tunnel Service への接続まで

![](https://cdn-0.plantuml.com/plantuml/png/LO-zQiGm381tFOK8NLll6KfIqws38Na1nL6fmJ_1bjE3uzxzVBbRCdtInwT1Gn7AKeE7hT5Pjr4KxBHtt6WyoM_AeKCggCsv6QlySMmxISf7CPw3kSR87YVEkxDy5FC4G5LIh67X3A0DldysYpt-bz8hPMdn_C4N2bkZJU5fb4rHo8fwkxucTEiDniUrDZr-_NmZhJjd0HWuhksXEm00)

##### 1. wss://cable.example.com/cable/contact/${contact id}

| value | でどころ |
| --- | ---- |
| contact id | 以前の接続時に受信した update message |
| link id | 以前の接続時に受信した update message |
| nonce | ランダムに生成 |

##### 2. BLE advert

BLE advert の復号は link secret と 1 で送信した nonce から導出した鍵によって行う。

| value | でどころ |
| --- | ---- |
| nonce | ランダムに生成 |

#### Tunnel Service への接続後

![](https://cdn-0.plantuml.com/plantuml/png/RL7DIiD06BpdAJvwrXVeGHfoqLl0li3QR9jWiYd9Rdl-DYBIAae5fI2AYgqWL4IAF_LjljhMjt0Z4Yjw-RwTdPbbXgsaaYLIHQ7LD3ke1Kqc99ISCE1cko6JzJY9Ii1ISDpV6bj9dmk3cOoJBuLjCILZeQ8jdbWYbXpY5a_0NZ7UCamdDn_z0y8HwHtIdVV34pC9xw6gXE0XRFAa5TGuaEvD9rt2FkckPJakhAw13MLOPy9BVRRD8U9foXhSidFASV_Ze1r87nfSReXw38IZ-VXeTdwsP55DZyb2_pSkuN6U5Pz9cyCGdAhj5gyJtf7U8xmc792-8Zscx8tq4sL3oXu9Rmcx-NssI_ebdykixYq6E7lGXAU45wGxy_xhudA_WD_LVvgdlf9Ut1P5s0eNdlcpJm00)

##### 1. Handshake message

| value | でどころ |
| ----- | --- |
| PSK | link secret と複合した BLE から導出 |
| ephemeral key | ランダム生成 |

##### 2. Handshake message with getInfo の結果

| value | 出どころ |
| ----- | --- |
| ephemeral key | ランダム生成 |
| getInfo の結果 | 認証器が保持 |

この時点で双方が通信に使う鍵が共有される。


## 参考資料

* [Client to Authenticator Protocol (CTAP) Review Draft, March 21, 2023](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#sctn-hybrid)
* [The Noise Protocol Framework](http://www.noiseprotocol.org/noise.html)
* [携帯電話のPasskeyはもう使えるメモ](https://zenn.dev/okuoku/scraps/35d81e1337262f)

## memo

QR コードの CBOR のエンコード
https://github.com/chromium/chromium/blob/2edb2e5b6179366c0f2153597e9b1a6e4cae1491/device/fido/cable/v2_handshake.cc#L441

https://github.com/chromium/chromium/blob/aaf3c8ca9f31375ac344e46947e02c443ef38a24/chrome/android/features/cablev2_authenticator/java/src/org/chromium/chrome/browser/webauth/authenticator/BLEAdvert.java#L70

QR コードのパース
https://github.com/chromium/chromium/commit/9730b37c518f817df3bab4064fa4e35591d08220

QR コードを読み取る側
https://github.com/chromium/chromium/blob/main/chrome/android/features/cablev2_authenticator/native/cablev2_authenticator_android.cc#L583