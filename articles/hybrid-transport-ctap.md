---
title: "CTAP2.2 の Hybrid transports"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["authentication", "ctap"]
published: false
---

[Client to Authenticator Protocol (CTAP) Review Draft, March 21, 2023 の 11.5. Hybrid transports](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#sctn-hybrid) に書いてある内容を整理しました。

## ざっくり

この仕様はPC などのクライアントプラットフォームに対してスマートフォンなどの別のデバイスを認証器として使うときの仕様で、それぞれ [QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-qr-initiated) と [State-assisted Transactions](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#hybrid-state-assisted) に分かれています。

### QR-initiated Transactions

![](/images/qr-initiated.png)

* クライアントプラットフォームに QR コードを表示し、それを認証器側のデバイスでスキャンします
* スマートフォンは BLE アドバタイズを行い、ここまでにやり取りした情報を使って双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになります
* この接続を通じて CTAP メッセージのやりとりをすることで、クライアントプラットフォームは認証器にクレデンシャルの作成を命令したり、認証器は作成したアテステーションをクライアントプラットフォームに送信したりすることができます

### State-assisted Transactions

![](/images/state-assisted.png)

* QR-initiated Transactions を一度行ったクライアントプラットフォーム・認証機間で再度やりとりをする際に利用します
* 以前の接続で交換済みの接続情報を使って、クライアントプラットフォームは tunnel service にアクセスします
* tunnel service は認証器にアクセスします
* スマートフォンは BLE アドバタイズを行います
* ここまでに取得した情報を使って双方がハンドシェイクを行い、双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになります
* 以降は QR-initiated Transactions

### tunnel service とは

クライアントプラットフォームが認証器とやりとりするために利用するサービスで、認証器を提供するプラットフォーマーによって提供されています。例えば認証器が Android であれば Google に、iPhone であれば Apple に "cable.ua5v.com" や "cable.auth.com" といったホストで提供されています。使用上 tunnel service は認証器の提供元が自分でホストすることもできるようになっているようです。

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

* ※1 なぜ数なのだろうか
* ※1 `cable.ua5v.com` だけ知ってる、はできるのに `cable.auth.com` しか知らないプラットフォームを作れないように見えるのがなんか変な気がする
* ※1 wellknown なものは全部サポートするのが前提なんだろうか

### 認証器による QR コードの読み取りと BLE advert

クライアントプラットフォームは認証器からの接続を待機し、認証器は BLE advertise を行います。これは認証器がクライアントプラットフォームの近くにあることを担保するために行われます。
これによって攻撃者が自分のページに、あるサイトに対して認証を行うための QR コードを表示してユーザーがそれをスキャンした場合、攻撃者の PC と被害者のスマートフォンが tunnel service を通じて接続されます。このとき、被害者のスマートフォンが発行した RP 向けのアサーションやアテステーションを攻撃者が取得してしまう、といった状況を回避します。

この BLE Advert は暗号化されて、認証、復号のための値は HKDF を使って QR secret から導出されます。

BLE advert には以下のフィールドがあります。

| value | desc |
| ----- | ---- |
| nonce | TODO |
| routing ID | TODO |
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

この時点でトンネルが確立し、クライアントプラットフォームと認証器は WebScoket フレーム でのやり取りができるようになります。

### ハンドシェイク

トンネルを通じてクライアントプラットフォームと認証器は Noise KNpsk0 暗号のハンドシェイクを行います。Noise プロトコルについてはよくわかってないのであいまいですが、 ここまでに交換した情報を使って秘密鍵を共有したりする感じだと思います。

具体的にはこのアルゴリズムでは事前に公開鍵と交換し、対象鍵を共通化しないといけないとあります。前者は QR コードに埋め込まれた公開鍵、後者は QR シークレットと復号した BLE advert から導出します。// TODO Noise が理解できたら書き直すかもしれない // TODO この辺のサンプルコードをもう少し真面目に読む

このとき、クライアントプラットフォームが BLE advert を受信したことを証明するためまず最初にハンドシェイクのメッセージを送信し、認証器はそれにレスポンスを返します。

これ以降のメッセージはハンドシェイクで導出した鍵によって暗号化されます。

認証器側からの最初のメッセージは `post handshake` で、これには [getInfo](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorGetInfo) のレスポンス、つまり認証器の情報（aaguid とか対応してるアルゴリズムとかトランスポートとか）が含まれています。

これ以降のメッセージにはメッセージの type が含まれるようになり、CTAP のコマンドは CTAP message type のメッセージによって送信されます。

// TODO update message については State-assisted Transactions について書いたあとで書く

クライアントプラットフォームは CTAP2 コマンドを認証器に送って何らかのアクションを送ります。getInfo はすでに受信しているので例えば登録の場合 [authenticatorMakeCredential](https://fidoalliance.org/specs/fido-v2.2-rd-20230321/fido-client-to-authenticator-protocol-v2.2-rd-20230321.html#authenticatorMakeCredential) を送ると認証器がクレデンシャルを作成してアテステーションをレスポンスしてくるのでそれを使ってよしなに登録処理を進めるといった感じになります。

## State-assisted Transactions の流れ

State-assisted Transactions では認証器との接続に以前 QR-initiated Transactions で交換した情報を使うので QR の読み取りを利用しません。

### tunnel service への接続

クライアントプラットフォームは認証器と紐づけてドメインと contact id を持っておき、このエンドポイントへの接続を試みます。

```
wss://"cable.example.com/cable/contact/${contact id}
```

一方認証器はどのクライアントプラットフォームと接続するかを知るための link ID とクライアントプラットフォームによる nonce を必要とします。ただしこちらについても認証器側の具体的な手順は仕様外です。

トンネルに接続できたら認証器はハンドシェイクメッセージを送信します。

### 認証器による BLE advert

この後認証器は近接性の検証のため BLE advert を行います。この BLE advert によって取得した nonce とクライアントプラットフォームが link ID と紐づけて保持しているシークレットを利用してハンドシェイク PSK を計算します。

### ハンドシェイク

これによって、クライアントプラットフォームは認証器にハンドシェイクのレスポンスをすることができます。
ハンドシェイクが完了したらあとは QR-initiated Transactions と同様に CTAP メッセージのやりとりを行ってアテステーションやアサーションの作成を依頼します。

// TODO contact id の取得を update message にからめてかく

## memo

QR コードの CBOR のエンコード
https://github.com/chromium/chromium/blob/2edb2e5b6179366c0f2153597e9b1a6e4cae1491/device/fido/cable/v2_handshake.cc#L441

https://github.com/chromium/chromium/blob/aaf3c8ca9f31375ac344e46947e02c443ef38a24/chrome/android/features/cablev2_authenticator/java/src/org/chromium/chrome/browser/webauth/authenticator/BLEAdvert.java#L70

QR コードのパース
https://github.com/chromium/chromium/commit/9730b37c518f817df3bab4064fa4e35591d08220

QR コードを読み取る側
https://github.com/chromium/chromium/blob/main/chrome/android/features/cablev2_authenticator/native/cablev2_authenticator_android.cc#L583