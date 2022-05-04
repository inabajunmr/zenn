---
title: "WebAuthn の get() と create() のパラメーター一覧（Level 3）"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['webauthn']
published: true
---

WebAuthn で get()、create() する時のパラメーターがそれぞれなんなのかをまとめました。

[Web Authentication: An API for accessing Public Key Credentials Level 3 Editor’s Draft, 6 April 2022](https://w3c.github.io/webauthn/) の [5. Web Authentication API](https://w3c.github.io/webauthn/#sctn-api) を参考にしています。
説明は要点を絞っているので正確な内容は [仕様](https://w3c.github.io/webauthn/#sctn-api) を参照ください。

Extension はよくわからないので型について記載していません。

## PublicKeyCredentialCreationOptions

navigator.credentials.create() のオプション。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| rp | [PublicKeyCredentialRpEntity](#publickeycredentialrpentity) | 必須 | Relying Party の名前と識別子。name は必須。id は RP ID となる。 id が未指定の場合、origin の有効ドメインが設定される。 |
| user | [PublicKeyCredentialUserEntity](#publickeycredentialuserentity) | 必須 | 登録を行うユーザーアカウントの名前と識別子。name、displayName、id は必須。id は userHandle となる。name と displayName はユーザーが選択のヒントとして使う。 |
| challenge | BufferSource | 必須 | 認証器が署名するチャレンジ。 |
| pubKeyCredParams | sequence<[PublicKeyCredentialParameters](#publickeycredentialparameters)> | 必須 | Relying Party がサポートする鍵のタイプと署名アルゴリズムの羅列。望ましいものを前に並べる。クライアント、認証器にとって利用可能なものがなければエラーになる。 |
| timeout | unsigned long |  | API 呼び出しの待機時間。 |
| excludeCredentials | sequence<[PublicKeyCredentialDescriptor](#publickeycredentialdescriptor)>       |  | ここに指定したクレデンシャルは認証器によって新しく作成されない。Relying Party が保持している、user.id に紐づくクレデンシャルをここに指定すると、同じユーザーアカウントが同じ認証器で複数のクレデンシャルを作成するのを防げる。 |
| authenticatorSelection | [AuthenticatorSelectionCriteria](#authenticatorselectioncriteria) |  | 認証器が満たさなければいけない機能もしくは設定。プラットフォーム認証器じゃないと駄目とか、user verification できないと駄目とか、discoverable credential を期待する、みたいなものを指定できる。 |
| attestation | DOMString |  | アテステーションを Relying Party にどう渡してほしいかを設定する。アテステーションなし、アテステーションステートメントの発行はクライアントに任せる（🍙）、認証器が発行したアテステーション、などを選べる。 |
| extensions | AuthenticationExtensionsClientInputs |  | クライアント、認証器に追加の処理を要求する場合に使う拡張領域。 |

## PublicKeyCredentialRequestOptions

navigator.credentials.get() のオプション。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| challenge | BufferSource | 必須 | 認証器が署名するチャレンジ。 |
| timeout | unsigned long |  | API 呼び出しの待機時間。 |
| rpId | USVString |  | Relying Party の RP ID。 |
| allowCredentials | sequence<[PublicKeyCredentialDescriptor](#publickeycredentialdescriptor)> |  | ユーザーアカウントが既に登録済みのクレデンシャルのリスト。 |
| userVerification | DOMString |  | get() で user verification をどれくらい望むかを指定する。必須なら "required"、必須でないが望むなら "preferred"、不要なら "discouraged" を指定する。 |
| extensions | AuthenticationExtensionsClientInputs |  | クライアント、認証器に追加の処理を要求する場合に使う拡張領域。 |

## PublicKeyCredentialRpEntity

Relying Party を表現するディクショナリー。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| name | DOMString | 必須 | 人間が覚えたり再入力できるようなエンティティーの名前。 |
| id | DOMString |  | Relying Party のユニークな識別子。RP ID となる。 |

## PublicKeyCredentialUserEntity

ユーザーアカウントを表現するディクショナリー。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| name | DOMString | 必須 | 人間が覚えたり再入力できるようなエンティティーの名前。 |
| id | BufferSource | 必須 | ユーザーアカウントのユーザーハンドル（ユーザーアカウントの識別子）。最大 64 バイトのバイトシーケンスで、ユーザーに表示されることを意図していない。ユーザーの識別はこの値を元に行う必要がある（displayName や name ではなく）。空は許容されない。メールアドレスなど、ユーザーの個人を特定する情報を含んではいけない。 |
| displayName | DOMString | 必須 | 人間が覚えたり再入力できるようなエンティティーの名前。 |

## PublicKeyCredentialParameters

新しいクレデンシャルの作成時に渡す追加のパラメーター。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| type | DOMString | 必須 | public-key" を指定する。
| alg | COSEAlgorithmIdentifier | 必須 | 暗号アルゴリズムの識別番号を指定する。番号は [COSE Algorithms](https://www.iana.org/assignments/cose/cose.xhtml#algorithms) を参照すること。"RS256" なら -257 を指定する。 |

## AuthenticatorSelectionCriteria

認証器の属性に関する要求を指定するディクショナリー。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| authenticatorAttachment | DOMString |  | platform" か "cross-platform" を指定する。クライアントデバイスから外せないような認証器を使わせたい場合、"platform" を指定する。 |
| residentKey | DOMString | | discoverable credential の作成をどれくらい望むかを指定する。"discouraged" なら望まない、"preferred" なら望むが必須でない、"required" なら必須となる。 |discoverable credential とは、認証時に Relying Party がユーザーアカウントを事前に識別できなくても利用可能なクレデンシャルのこと。 |
| requireResidentKey | boolean | | WebAuthn Level 1 の後方互換性のために歴史的な理由で残っているフィールド。residentKey が "required" なら true にする。 |
| userVerification | DOMString | | create() で user verification をどれくらい望むかを指定する。必須なら "required"、必須でないが望むなら "preferred"、不要なら "discouraged" を指定する。 |

## PublicKeyCredentialDescriptor

公開鍵クレデンシャルを表すディクショナリー。

| フィールド名 | 型 | 必須 | 説明 |
| --- | --- | --- | --- |
| type | DOMString | 必須 | public-key" を指定する。
| id | BufferSource | 必須 | 公開鍵クレデンシャルの credential ID を指定する。
| transports | sequence<DOMString> | クレデンシャルに紐づく認証器がクライアントとどのようにやり取りするか。"usb"、"nfc"、"ble"、"internal" のいずれかを指定する。 |
