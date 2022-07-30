---
title: ""
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

WebAuthn で get()、create() したときに返ってくる PublicKeyCredential についてまとめてみました。

[Web Authentication: An API for accessing Public Key Credentials Level 3 Editor’s Draft, 27 July 2022](https://w3c.github.io/webauthn/) の [5. Web Authentication API](https://w3c.github.io/webauthn/#sctn-api) を参考にしています。
説明は要点を絞っているので正確な内容は [仕様](https://w3c.github.io/webauthn/#sctn-api) を参照ください。

# [PublicKeyCredential](https://w3c.github.io/webauthn/#publickeycredential)

[PublicKeyCredential](https://w3c.github.io/webauthn/#publickeycredential) は [Credential](https://w3c.github.io/webappsec-credential-management/#credential) を継承しています。


| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| id | USVString | Credential をオーバーライドしている。この PublicKeyCredential の識別子。 |
| type | DOMString | 'public-key' |
| response | AuthenticatorResponse | 公開鍵クレデンシャルの作成 / 認証アサーションの生成リクエストに対する認証器のレスポンス。create() の場合 AuthenticatorAttestationResponse に、get() の場合 AuthenticatorAssertionResponse となる。|
| authenticatorAttachment | USVString | 'plat-form' / 'cross-platform' |
| AuthenticationExtensionsClientOutputs getClientExtensionResults() | method | Relying Party にリクエストされた Client Extension を処理した結果。 |
| static Promise<boolean> isConditionalMediationAvailable() | method | Credential をオーバーライドしている。conditional user mediation が利用可能かどうかを返す。options.meditation に conditional を指定する場合、事前にこのメソッドを使って利用可能か確認すべき。このメソッドが存在しない場合、conditional user mediation は利用できない。(参考: https://github.com/w3c/webauthn/wiki/Explainer:-WebAuthn-Conditional-UI#api-layer) |
| PublicKeyCredentialJSON toJSON() | method | RegistrationResponseJSON か AuthenticationResponseJSON を返す。これらは PublicKeyCredential の JSON 表現となり、Relying Party のサーバーに application/json ペイロードで送信するのに適している。 |

# AuthenticatorAttestationResponse

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | ArrayBuffer | クライアントから認証器に渡された client data の JSON 表現。JSON のフィールドは CollectedClientData を参照。 |
| attestationObject | ArrayBuffer | attestation object。認証器そのものを担保するための情報、ユーザーが認証に使う公開鍵、ユーザーが認証器とどのようなインタラクションをしたか、といった情報が含まれる。詳細なフォーマットは [attestation object](https://w3c.github.io/webauthn/#attestation-object) を参照。 |
TODO

# AuthenticatorAssertionResponse

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | ArrayBuffer |  |

# CollectedClientData

https://w3c.github.io/webauthn/#dictdef-collectedclientdata

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| type | DOMString | 'webauthn.create' / 'webauthn.get'  |
| challenge | DOMString | Base64 エンコードされた Relying Party に渡されたチャレンジ |
| origin | DOMString | クライアントが認証器に渡したオリジン。 |
| crossOrigin | boolean |  |



# RegistrationResponseJSON

# AuthenticationResponseJSON