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

// TODO 図

* クライアントプラットフォームに QR コードを表示し、それを認証器側のデバイスでスキャンします
* スマートフォンは BLE アドバタイズを行い、ここまでにやり取りした情報を使って双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになります
* この接続を通じて CTAP メッセージのやりとりをすることで、クライアントプラットフォームは認証器にクレデンシャルの作成を命令したり、認証器は作成したアテステーションをクライアントプラットフォームに送信したりすることができます

### State-assisted Transactions

// TODO 図

* QR-initiated Transactions を一度行ったクライアントプラットフォーム・認証機間で再度やりとりをする際に利用します
* 以前の接続で交換済みの接続情報を使って、クライアントプラットフォームは tunnel service にアクセスします
* tunnel service は認証器にアクセスします
* スマートフォンは BLE アドバタイズを行います
* ここまでに取得した情報を使って双方がハンドシェイクを行い、双方のデバイスが tunnel service 経由で WebSocket によってやりとりできるようになります
* 以降は QR-initiated Transactions

### tunnel service とは

クライアントプラットフォームが認証器とやりとりするために利用するサービスで、認証器を提供するプラットフォーマーによって提供されています。例えば認証器が Android であれば Google に、iPhone であれば Apple に "cable.ua5v.com" や "cable.auth.com" といったホストで提供されています。使用上 tunnel service は認証器の提供元が自分でホストすることもできるようになっているようです。

