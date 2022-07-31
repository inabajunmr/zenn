---
title: "WebAuthn の get() と create() のレスポンス一覧（Level 3）"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['webauthn','authentication']
published: true
---

WebAuthn で get()、create() したときに返ってくる PublicKeyCredential についてまとめてみました。

[Web Authentication: An API for accessing Public Key Credentials Level 3 Editor’s Draft, 27 July 2022](https://w3c.github.io/webauthn/) の [5. Web Authentication API](https://w3c.github.io/webauthn/#sctn-api) を参考にしています。
説明は要点を絞っているので正確な内容は [仕様](https://w3c.github.io/webauthn/#sctn-api) を参照ください。

# [PublicKeyCredential](https://w3c.github.io/webauthn/#publickeycredential)

[PublicKeyCredential](https://w3c.github.io/webauthn/#publickeycredential) は [Credential](https://w3c.github.io/webappsec-credential-management/#credential) を継承しています。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| id | USVString | Credential をオーバーライドしている。この PublicKeyCredential の識別子。rawId を base64url encoding したもの。 |
| rawId | ArrayBuffer | 認証器によって選択された [credential ID](https://w3c.github.io/webauthn/#credential-id)。 |
| type | DOMString | 'public-key' |
| response | AuthenticatorResponse | 公開鍵クレデンシャルの作成 / 認証アサーションの生成リクエストに対する認証器のレスポンス。create() の場合 [AuthenticatorAttestationResponse](#authenticatorattestationresponse) に、get() の場合 [AuthenticatorAssertionResponse](#authenticatorassertionresponse) となる。|
| authenticatorAttachment | USVString | 'plat-form' / 'cross-platform' |
| AuthenticationExtensionsClientOutputs getClientExtensionResults() | method | Relying Party にリクエストされた [Client Extension](https://w3c.github.io/webauthn/#client-extension-processing) を処理した結果。 |
| static Promise<boolean> isConditionalMediationAvailable() | method | Credential をオーバーライドしている。[conditional user mediation](https://w3c.github.io/webappsec-credential-management/#dom-credentialmediationrequirement-conditional) が利用可能かどうかを返す。options.meditation に conditional を指定する場合、事前にこのメソッドを使って利用可能か確認すべき。このメソッドが存在しない場合、conditional user mediation は利用できない。(参考: https://github.com/w3c/webauthn/wiki/Explainer:-WebAuthn-Conditional-UI#api-layer) |
| PublicKeyCredentialJSON toJSON() | method | [RegistrationResponseJSON](#registrationresponsejson) か [AuthenticationResponseJSON](#authenticationresponsejson) を返す。これらは PublicKeyCredential の JSON 表現となり、Relying Party のサーバーに application/json ペイロードで送信するのに適している。 |

# [AuthenticatorAttestationResponse](https://w3c.github.io/webauthn/#authenticatorattestationresponse)

[AuthenticatorAttestationResponse](https://w3c.github.io/webauthn/#authenticatorattestationresponse) は [AuthenticatorResponse](https://w3c.github.io/webauthn/#authenticatorresponse) を継承しています。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | ArrayBuffer | クライアントから認証器に渡された client data の JSON 表現。JSON のフィールドは [CollectedClientData](#collectedclientdata) を参照。 |
| attestationObject | ArrayBuffer | attestation object。認証器そのものを担保するための情報、ユーザーが認証に使う公開鍵、ユーザーが認証器とどのようなインタラクションをしたか、といった情報が含まれる。詳細なフォーマットは [attestation object](https://w3c.github.io/webauthn/#attestation-object) を参照。 |
| sequence<DOMString> getTransports() | method | 認証器がサポートしている transport のリスト。この情報がが利用できない場合空になる。[AuthenticatorTransport](https://w3c.github.io/webauthn/#enumdef-authenticatortransport) のメンバーであるべきだが、そうでない場合も Relying Party は受け入れるべきである。 |
| ArrayBuffer getAuthenticatorData() | method | attestationObject 内の [authenticator data](https://w3c.github.io/webauthn/#authenticator-data) を返す。 詳細は[5.2.1.1. Easily accessing credential data](https://w3c.github.io/webauthn/#sctn-public-key-easy) を参照。|
| ArrayBuffer? getPublicKey() | method | [SubjectPublicKeyInfo](https://tools.ietf.org/html/rfc5280#section-4.1.2.7) として、[credential public key](https://w3c.github.io/webauthn/#credential-public-key) を返す。詳細は[5.2.1.1. Easily accessing credential data](https://w3c.github.io/webauthn/#sctn-public-key-easy) を参照。 |
| COSEAlgorithmIdentifier  getPublicKeyAlgorithm() | method | 新しいクレデンシャルの [COSEAlgorithmIdentifier](https://w3c.github.io/webauthn/#typedefdef-cosealgorithmidentifier) を返す。詳細は[5.2.1.1. Easily accessing credential data](https://w3c.github.io/webauthn/#sctn-public-key-easy) を参照。 |

# [AuthenticatorAssertionResponse](https://w3c.github.io/webauthn/#authenticatorassertionresponse)

[AuthenticatorAssertionResponse](https://w3c.github.io/webauthn/#authenticatorassertionresponse) は [AuthenticatorResponse](https://w3c.github.io/webauthn/#authenticatorresponse) を継承しています。


| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | ArrayBuffer | クライアントから認証器に渡された client data の JSON 表現。JSON のフィールドは CollectedClientData を参照。 |
| authenticatorData | ArrayBuffer | 認証器が返した [authenticator data](https://w3c.github.io/webauthn/#authenticator-data)。[https://w3c.github.io/webauthn/#sctn-authenticator-data](https://w3c.github.io/webauthn/#sctn-authenticator-data) を参照。 |
| signature | ArrayBuffer | 認証器が返した署名。[§ 6.3.3 The authenticatorGetAssertion Operation](https://w3c.github.io/webauthn/#sctn-op-get-assertion) を参照。 |
| userHandle | ArrayBuffer? | 認証器が返した [user handle](https://w3c.github.io/webauthn/#user-handle)。[§ 6.3.3 The authenticatorGetAssertion Operation](https://w3c.github.io/webauthn/#sctn-op-get-assertion) を参照。 |

# [CollectedClientData](https://w3c.github.io/webauthn/#dictdef-collectedclientdata)

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| type | DOMString | 'webauthn.create' / 'webauthn.get'  |
| challenge | DOMString | Base64 エンコードされた Relying Party に渡されたチャレンジ |
| origin | DOMString | クライアントが認証器に渡したオリジン。 |
| crossOrigin | boolean |  |

# [RegistrationResponseJSON](https://w3c.github.io/webauthn/#dictdef-registrationresponsejson)

各フィールドの説明は [PublicKeyCredential](#publickeycredential) を参照。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| id | Base64URLString |  |
| rawId | Base64URLString |  |
| reseponse | [AuthenticatorAttestationResponseJSON](#authenticatorattestationresponsejson) |  |
| authenticatorAttachment | DOMString? |  |
| clientExtensionResults | [AuthenticationExtensionsClientOutputsJSON](https://w3c.github.io/webauthn/#dictdef-authenticationextensionsclientoutputsjson) |  |
| type | DOMString |  |

# [AuthenticatorAttestationResponseJSON](https://w3c.github.io/webauthn/#dom-authenticatorattestationresponsejson)

各フィールドの説明は [AuthenticatorAttestationResponse](#authenticatorattestationresponse) を参照。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | Base64URLString |  |
| attestationObject | Base64URLString |  |
| transports | sequence<DOMString> |  |

# [AuthenticationResponseJSON](https://w3c.github.io/webauthn/#dictdef-authenticationresponsejson)

各フィールドの説明は PublicKeyCredential を参照。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| id | Base64URLString |  |
| rawId | Base64URLString |  |
| reseponse | [AuthenticatorAssertionResponseJSON](#authenticatorassertionresponsejson) |  |
| authenticatorAttachment | DOMString? |  |
| clientExtensionResults | [AuthenticationExtensionsClientOutputsJSON](https://w3c.github.io/webauthn/#dictdef-authenticationextensionsclientoutputsjson) |  |
| type | DOMString |  |

# [AuthenticatorAssertionResponseJSON](https://w3c.github.io/webauthn/#dictdef-authenticatorassertionresponsejson)

各フィールドの説明は [AuthenticatorAssertionResponse](#authenticatorassertionresponse) を参照。

| フィールド名/メソッド定義 | 型 | 説明 |
| --- | --- | --- |
| clientDataJSON | Base64URLString |  |
| authenticatorData | Base64URLString |  |
| signature | Base64URLString |  |
| userHandle | Base64URLString |  |
