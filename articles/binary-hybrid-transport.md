---
title: "バイナリで理解する Hybrid Transport のクライアント実装"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["authentication", "ctap"]
published: false
---

# Hybrid Transport

Hybrid Transport の基本的な流れは以下です。
![QR-initiated Transactions の図](/images/qr-initiated.png)

この記事ではこの図の1-3の各フェーズで具体的にどのようなメッセージのやり取りが行われるのかを説明します。

## クライアントが表示する QR コード

QR コードは以下の値を CBOR でエンコードしたものです。([11.5.1. QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#hybrid-qr-initiated))

| key | value                                                                                                                                                                                    | desc                                                                                                                                                                                         |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | 公開鍵                                                                                                                                                                                   | トンネル経由でクライアントプラットフォームと認証器がハンドシェイクを行うのに使う                                                                                                             |
| 1   | QR secret                                                                                                                                                                                | BLE advertise の暗号化、復号、認証に使う                                                                                                                                                     |
| 2   | クライアントプラットフォームが知ってる tunnel service のドメインの数 ※1                                                                                                                  | これが 2 だったらこのプラットフォームは `cable.ua5v.com` と `cable.auth.com` を知っていることになり、後述の BLE advert で 0 を指定すれば前者に、1 を指定すれば後者を利用することが表現できる |
| 3   | 現在時刻                                                                                                                                                                                 |
| 4   | プラットフォームが state-assisted transactions に対応しているかどうか（省略可。この例では省略）                                                                                          |
| 5   | 今後行われる操作についてのヒント（getAssertion (FIDO2) or makeCredential (FIDO2) or credential presentation (Digital Credentials API) or credential issuance (Digital Credentials API)） |
| 6   | トランスポートに使われるチャネル                                                                                                                                                         |

### CBOR の組み立て

以下の値で CBOR を組み立てます。

| 項目           | 値                                                                   |
| -------------- | -------------------------------------------------------------------- |
| QR Secret      | `97aa809b276911764f20b2921b6c7539`                                   |
| 公開鍵         | `021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049` |
| タイムスタンプ | `1780082784`                                                         |
| ヒント         | `ga`                                                                 |
| チャネル       | `0`                                                                  |

公開鍵に対する秘密鍵は以下となります。

```
b85bd584301fd9d938307a0a7d63608ce6272671052023c2e85dad1efc90707f
```

```go
payload := map[uint64]any{
    0: publicKey,
    1: secret[:],
    2: 2,
    3: cfg.Timestamp.Unix(),
    5: "ga",
    6: []uint64{0},
    0xffff: uint64(0),
}
// CBOR にエンコード
encoded, _ := canonicalCBOR.Marshal(payload)
```

CBOR は以下となります。

```
a7005821021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049015097aa809b276911764f20b2921b6c75390202031a6a19e8600562676106810019ffff00
```

この CBOR は以下のように読むことができます。

```
a7                       map(7)
  00                     key 0
  58 21                  byte string(33)
    021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049
                         public key

  01                     key 1
  50                     byte string(16)
    97aa809b276911764f20b2921b6c7539
                         QR Secret

  02                     key 2
  02                     unsigned(2)

  03                     key 3
  1a 6a19e860            timestamp = 1780082784

  05                     key 5
  62 6761                "ga"

  06                     key 6
  81                     array(1)
    00                   WebSocket

  19ffff                 key 65535
  00                     GREASE(Authenticator が未知のキーを無視することをチェックするための値)
```

これを digitEncode します。

1. CBOR bytes を最大 7 bytes ごとに区切る
2. 各 chunk を little-endian uint64 として読む
3. 10 進文字列にする
4. chunk 長ごとの固定桁数まで左を `0` で埋める
5. 連結する

CBOR の先頭 7 bytes は `a7 00 58 21 02 1b bb` となります。
これを 8 bytes buffer にいれて `a7 00 58 21 02 1b bb 00` とし、これを little-endian uint64 として読むと下位バイトから読むことになるので `00 bb 1b 02 21 58 00 a7` となり、これを 10 進数にすると `52665516608192679` となります。

これを CBOR 全体に適用して FIDO:/ を結合すると

```

FIDO:/526655166081926790466861943578209849612861246703166115785136344826622391203681835852636216363878009120090945677715803150062612644112453027276793616139010001418644039521330016776985

```

となります。これが QR コードとなります。

元の値から URL を組み立てるサンプルコードです。

```go
package main

import (
	"encoding/binary"
	"encoding/hex"
	"fmt"
	"strconv"

	"github.com/fxamacker/cbor/v2"
)

func main() {
	publicKey := decodeHex("021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049")
	qrSecret := decodeHex("97aa809b276911764f20b2921b6c7539")

	canonicalCBOR, err := cbor.CanonicalEncOptions().EncMode()
	if err != nil {
		panic(err)
	}

	payload := map[uint64]any{
		0:      publicKey,
		1:      qrSecret,
		2:      uint64(2),
		3:      uint64(1780082784),
		5:      "ga",
		6:      []uint64{0},
		0xffff: uint64(0),
	}

	cborBytes, err := canonicalCBOR.Marshal(payload)
	if err != nil {
		panic(err)
	}

	fmt.Println("FIDO:/" + digitEncode(cborBytes))
}

func decodeHex(s string) []byte {
	out, err := hex.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return out
}
```

digitEncode は [11.5.1. QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#hybrid-qr-initiated) のサンプルコードを参照してください。

## BLE Advert

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

Authenticator が送信する BLE Advert の Service Data は暗号化されています。ここでは受信した Service Data を

```
9f4bc7c7ed77b8a021ba504084f19c55ab2a9e89
```

とします。

前半の

```
9f4bc7c7ed77b8a021ba504084f19c55
```

が暗号文で、

```
ab2a9e89
```

が認証用のタグです。

この暗号文を復号するための鍵を QR コードに埋め込んだ QR Secret から HKDF-SHA256 で導出します。

```
97aa809b276911764f20b2921b6c7539
```

HKDF-SHA256 は以下の手順で行います。

```
# Extract
PRK = HMAC-SHA256(nil, h'97aa809b276911764f20b2921b6c7539')

# Expand
info = 01000000 # Key purpose が 1(keyPurposeEIDKey)
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || info || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || info || 0x02)
...
OKM = T(1) || T(2)
```

Go だと以下のようになります。

```go
qrSecret := decodeHex("97aa809b276911764f20b2921b6c7539")
eidKey, err := derive(qrSecret, nil, keyPurposeEIDKey, 64)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", eidKey)
```

derive は [11.5.1. QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#hybrid-qr-initiated) のサンプルコードを参照してください。

OKM の前半を使って暗号文を復号します。
OKM の後半は HMAC tag の検証に使います。

```
AES-Decrypt(OKM[0:32], 9f4bc7c7ed77b8a021ba504084f19c55)
```

CTAP2 のサンプルコードでは tag の検証と AES 復号が trialDecrypt 関数として記載されています。（[11.5.1. QR-initiated Transactions](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#hybrid-qr-initiated)）

結果が `ab2a9e89` なら、BLE Advert の末尾 4 bytes と一致します。

このように BLE Advert(`9f4bc7c7ed77b8a021ba504084f19c55`) を復号すると

```
006832febb84eab2ff632c4d58880100
```

となります。

これを分割すると nonce、routing ID、tunnel service identifier が取得できます。

| 項目                      | 値                     |
| ------------------------- | ---------------------- |
| nonce                     | `6832febb84eab2ff632c` |
| routing ID                | `4d5888`               |
| tunnel service identifier | `0x0001`               |

1 なので tunnel の domain は　`cable.auth.com` となります。

## Tunnel Service への接続

Tunnel service の URL は以下のようになります。

```
wss://{tunnel domain}/cable/connect/{routing id}/{tunnel id}
```

ここまでに取得した値で Tunnel service の URL を組み立てることができます。

```
wss://cable.auth.com/cable/connect/4d5888/95074c89b06c2eb6bd4d79566d38dbf0
```

tunnel ID の `95074c89b06c2eb6bd4d79566d38dbf0` は QR Secret から導出できます。

keyPurposeTunnelID が 2 なので

```
# Extract
PRK = HMAC-SHA256(nil, h'97aa809b276911764f20b2921b6c7539')
# Expand
info = 02000000 # Key purpose が 2(keyPurposeTunnelID)
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || info || 0x01)
...
OKM = T(1) の先頭 16 bytes が tunnel ID(95074c89b06c2eb6bd4d79566d38dbf0) となる
```

となります。

```go
qrSecret := decodeHex("97aa809b276911764f20b2921b6c7539")
tunnelID, err := derive(qrSecret, nil, keyPurposeTunnelID, 16)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", tunnelID)
```

### Noise handshake

#### Initial message

Noise Protocol Framework の [KNpsk0](https://noiseexplorer.com/patterns/KNpsk0/) は以下のように定義されています。

```text
KNpsk0:
  -> s
  ...
  -> psk, e
  <- e, ee, se
```

| 要素  | 説明                                                                                                                               |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `s`   | QR コードに埋め込んで Authenticator に送信した公開鍵                                                                               |
| `psk` | ハンドシェイクの前に Client / Authenticator で共有する共通鍵。QR Secret と復号した BLE Advert とか導出する。                       |
| `e`   | メッセージの暗号化に使う鍵を交換するために生成する一時的なキーペアの、公開鍵                                                       |
| `ee`  | メッセージの暗号化に使う鍵を導出するために受け取った公開鍵（e）と、自身で生成した秘密鍵から生成した共有秘密                        |
| `se`  | メッセージの暗号化に使う鍵を導出するために受け取った公開鍵（e）と、QR コードに埋め込んだ公開鍵の対になる秘密鍵から生成した共有秘密 |

PSK は QR Secret(`97aa809b276911764f20b2921b6c7539`) と復号した BLE Advert(`006832febb84eab2ff632c4d58880100`)から導出します。

```
# Extract
PRK = HMAC-SHA256(h'006832febb84eab2ff632c4d58880100', h'97aa809b276911764f20b2921b6c7539')

# Expand
info = 03000000 # Key purpose が 3(keyPurposePSK)
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || info || 0x01)
...
PSK = T(1)
```

Go だと以下です。

```go
qrSecret := decodeHex("97aa809b276911764f20b2921b6c7539")
advertPlaintext := decodeHex("006832febb84eab2ff632c4d58880100")

psk, err := derive(qrSecret, advertPlaintext, keyPurposePSK, 32)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", psk)
```

メッセージの暗号化に使う鍵を交換するための一時的なキーペアを生成します。
秘密鍵を以下とします。

```
9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16
```

P-256 の base point と秘密鍵をかけると公開鍵が取得できます。

```
9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16 * P-256 base point
```

公開鍵を X9.62 でエンコードすると e(ephemeral public key) となります。

```
0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653
```

Go だと以下です。

```go
scalar := new(big.Int).SetBytes(decodeHex("9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16"))

x, y := elliptic.P256().ScalarBaseMult(scalar.FillBytes(make([]byte, 32)))
e := elliptic.Marshal(elliptic.P256(), x, y)

fmt.Printf("%x\n", e)
```

Noise は Noise State として以下の値を持ち、handshake やデータ通信の処理に合わせてこれらを更新することでコネクションを維持します。

| 値                         | 説明                                                                                                                                                                   |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `noiseState.chainingKey`   | handshake 中に PSK、ephemeral public key、ECDH の共有秘密などを順番に混ぜて更新する値です。handshake の最後に `writeKey` / `readKey` を導出する元になります。          |
| `noiseState.handshakeHash` | handshake 中に送受信した公開データや暗号文を順番にハッシュして更新する値です。AEAD の associated data として使われ、handshake の transcript を暗号処理に結びつけます。 |
| `noiseState.symmetricKey`  | handshake message の payload を暗号化・復号するための一時的な鍵です。`mixKey` によって更新されます。                                                                   |
| `noiseState.nonce`         | `noiseState.symmetricKey` で AEAD を使うときの nonce です。handshake 中の暗号化・復号ごとに更新されます。                                                              |

chainingKey / handshakehash を初期化します。

```
// 32 Bytes 以下なので 0 padding
noiseState.chainingKey = "Noise_KNpsk0_P256_AESGCM_SHA256" || 00
noiseState.handshakeHash = "Noise_KNpsk0_P256_AESGCM_SHA256" || 00
```

QR initiated の場合、noiseState に 0x01 を混ぜます。

```text
// mixHash(01)
noiseState.handshakeHash = SHA256(noiseState.handshakeHash || 01)
```

noiseState に QR コードで送信した公開鍵を混ぜます。

```
// mixHash(uncompressed client static public key)
x,y = uncompress(h'021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049')
noiseState.handshakeHash = SHA256(noiseState.handshakeHash || 04 || x || y)
```

PSK を noiseState に混ぜます。

```
# MixKeyAndHash(PSK)
## Extract

PRK = HMAC-SHA256(noiseState.chainingKey, psk)

## Expand
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || empty || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || empty || 0x02)
T(3) = HMAC-SHA256(PRK, T(2) || empty || 0x03)

out = T(1) || T(2) || T(3)

noiseState.chainingKey = out[0:32] = T(1)
tempHash = out[32:64] = T(2)
noiseState.symmetricKey = out[64:96] = T(3)

noiseState.nonce = 0

noiseState.handshakeHash = SHA256(noiseState.handshakeHash || tempHash)
```

先ほど生成したキーペアの公開鍵を handshakeHash に混ぜます。

```

// mixHash(e public)
noiseState.handshakeHash = SHA256(noiseState.handshakeHash || h'0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653')
```

キーペアの公開鍵を chaining key にも混ぜます。

```
// mixKey(e public)
PRK = HMAC-SHA256(noiseState.chainingKey, h'0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653')

# Expand

T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || empty || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || empty || 0x02)

out = T(1) || T(2)

noiseState.chainingKey = out[0:32] = T(1)
noiseState.symmetricKey = out[32:64] = T(2)
noiseState.nonce = 0
```

mixKey、mixHash とこのあと出てくる mixKeyAndHash、decryptAndHash の定義は [The Noise Protocol Framework の 5.2. The SymmetricState object](https://noiseprotocol.org/noise.html#the-symmetricstate-object) を参照して下さい。
最初のメッセージにペイロードはないので空の payload を暗号化して AES-GCM の AEAD タグをつけます。

```
// AEAD tag
AES-GCM-Seal(
key = noiseState.symmetricKey,
nonce = noiseState.nonce,
plaintext = empty,
associatedData = noiseState.handshakeHash
)
= f8f93344c8213dd9e0e6c67c29139e
```

payload が空なので暗号文はなしです。
ephemeral public key と AEAD tag を結合して noise の最初のメッセージができます。

```
0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653 || f8f93344c8213dd9e0e6c67c29139e
```

ここまでを Go で書くと以下のようになります。

```go
type noiseState struct {
	chainingKey   [32]byte
	handshakeHash [32]byte
	symmetricKey  [32]byte
	nonce         uint32
}

func newNoise() *noiseState {
	var h [32]byte
	copy(h[:], []byte("Noise_KNpsk0_P256_AESGCM_SHA256"))
	return &noiseState{
		chainingKey:   h,
		handshakeHash: h,
	}
}

func (n *noiseState) mixHash(data []byte) {
	h := sha256.New()
	h.Write(n.handshakeHash[:])
	h.Write(data)
	copy(n.handshakeHash[:], h.Sum(nil))
}

func (n *noiseState) mixHashPoint(pub *ecdsa.PublicKey) {
	n.mixHash(elliptic.Marshal(pub.Curve, pub.X, pub.Y))
}

func (n *noiseState) mixKey(ikm []byte) error {
	out, err := hkdfExpand(ikm, n.chainingKey[:], 64)
	if err != nil {
		return err
	}
	copy(n.chainingKey[:], out[:32])
	copy(n.symmetricKey[:], out[32:64])
	n.nonce = 0
	return nil
}

func (n *noiseState) mixKeyAndHash(ikm []byte) error {
	out, err := hkdfExpand(ikm, n.chainingKey[:], 96)
	if err != nil {
		return err
	}
	copy(n.chainingKey[:], out[:32])
	n.mixHash(out[32:64])
	copy(n.symmetricKey[:], out[64:96])
	n.nonce = 0
	return nil
}

func (n *noiseState) encryptAndHash(plaintext []byte) ([]byte, error) {
	block, err := aes.NewCipher(n.symmetricKey[:])
	if err != nil {
		return nil, err
	}
	aead, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	var nonce [12]byte
	binary.BigEndian.PutUint32(nonce[:4], n.nonce)
	n.nonce++

	ciphertext := aead.Seal(nil, nonce[:], plaintext, n.handshakeHash[:])
	n.mixHash(ciphertext)
	return ciphertext, nil
}

func initialHandshakeMessage(psk *[32]byte, identityKey, ephemeral *ecdsa.PrivateKey) ([]byte, *noiseState, error) {
	noiseState := newNoise()

	noiseState.mixHash([]byte{1})
	noiseState.mixHashPoint(&identityKey.PublicKey)
	if err := noiseState.mixKeyAndHash(psk[:]); err != nil {
		return nil, nil, err
	}

	ephemeralBytes := elliptic.Marshal(ephemeral.Curve, ephemeral.PublicKey.X, ephemeral.PublicKey.Y)
	noiseState.mixHash(ephemeralBytes)
	if err := noiseState.mixKey(ephemeralBytes); err != nil {
		return nil, nil, err
	}

	ciphertext, err := noiseState.encryptAndHash(nil)
	if err != nil {
		return nil, nil, err
	}

	message := append(ephemeralBytes, ciphertext...)
	return message, noiseState, nil
}

func hkdfExpand(secret, salt []byte, outLen int) ([]byte, error) {
	return hkdf.Key(sha256.New, secret, salt, "", outLen)
}

func p256PrivateKey(scalarHex string) *ecdsa.PrivateKey {
	scalar := new(big.Int).SetBytes(decodeHex(scalarHex))
	x, y := elliptic.P256().ScalarBaseMult(scalar.FillBytes(make([]byte, 32)))
	return &ecdsa.PrivateKey{
		PublicKey: ecdsa.PublicKey{Curve: elliptic.P256(), X: x, Y: y},
		D:         scalar,
	}
}
```

このサンプルを先ほどの固定値で呼び出すと、Initial message が得られます。

```go
var pskArray [32]byte
copy(pskArray[:], psk)

identityKey := p256PrivateKey("b85bd584301fd9d938307a0a7d63608ce6272671052023c2e85dad1efc90707f")
ephemeral := p256PrivateKey("9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16")

initial, noiseState, err := initialHandshakeMessage(&pskArray, identityKey, ephemeral)
if err != nil {
	panic(err)
}

fmt.Printf("%x\n", initial)
fmt.Printf("%x\n", noiseState.handshakeHash)
```

#### Response

Initial message に対して Authenticator が返してきたメッセージが

```
043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b5605d73235109111433fade982b0918e17
```

であるとします。

```
<- e, ee, se
```

なのでまず Authenticator の ephemeral public key が来ます。

```
043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b56
```

残りが payload と AEAD のタグですが、AES-GCM の場合 16 bytes なので payload は空です。

```
05d73235109111433fade982b0918e17
```

ee / se を計算します。

```text
ee:
  ECDH(
    initiator e private = 9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16,
    responder e public  = 043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b56
  )

se:
  ECDH(
    initiator s private = b85bd584301fd9d938307a0a7d63608ce6272671052023c2e85dad1efc90707f,
    responder e public  = 043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b56
  )
```

Go だと `ee` と `se` の計算はこうなります。

```go
func ecdhSecret(priv *ecdsa.PrivateKey, pub *ecdsa.PublicKey) ([]byte, error) {
	privECDH, err := priv.ECDH()
	if err != nil {
		return nil, err
	}
	pubECDH, err := pub.ECDH()
	if err != nil {
		return nil, err
	}
	return privECDH.ECDH(pubECDH)
}

responderEPublic := response[:65]
x, y := elliptic.Unmarshal(elliptic.P256(), responderEPublic)
if x == nil || y == nil {
	panic("invalid peer public key")
}
peerPublic := &ecdsa.PublicKey{Curve: elliptic.P256(), X: x, Y: y}

ee, err := ecdhSecret(ephemeral, peerPublic)
if err != nil {
	panic(err)
}

se, err := ecdhSecret(identityKey, peerPublic)
if err != nil {
	panic(err)
}
```

次に、Authenticator から受け取った `e`、`ee`、`se` を順番に Noise State に混ぜます。

`e` は handshakeHash と chaining key の両方に混ぜます。`ee` と `se` は `mixKey` で chaining key に混ぜます。

```text
// MixHash(responder e public)
noiseState.handshakeHash = SHA256(noiseState.handshakeHash || responderEPublic)

// MixKey(responder e public)
noiseState.chainingKey, noiseState.symmetricKey = HKDF(noiseState.chainingKey, responderEPublic)
noiseState.nonce = 0

// MixKey(ee)
noiseState.chainingKey, noiseState.symmetricKey = HKDF(noiseState.chainingKey, ee)
noiseState.nonce = 0

// MixKey(se)
noiseState.chainingKey, noiseState.symmetricKey = HKDF(noiseState.chainingKey, se)
noiseState.nonce = 0
```

Go だとこうなります。

```go
noiseState.mixHash(responderEPublic)
if err := noiseState.mixKey(responderEPublic); err != nil {
	panic(err)
}

if err := noiseState.mixKey(ee); err != nil {
	panic(err)
}

if err := noiseState.mixKey(se); err != nil {
	panic(err)
}
```

その後、response の payload を復号して AEAD tag を検証します。payload は空なので、復号結果が空であることを確認します。

```text
plaintext = AES-GCM-Open(
  key = noiseState.symmetricKey,
  nonce = noiseState.nonce,
  ciphertext = response ciphertext,
  associatedData = noiseState.handshakeHash
)

noiseState.nonce += 1

// DecryptAndHash なので、復号できたら ciphertext も handshakeHash に混ぜる
noiseState.handshakeHash = SHA256(noiseState.handshakeHash || response ciphertext)
```

Go だとこうなります。

```go
func (n *noiseState) decryptAndHash(ciphertext []byte) ([]byte, error) {
	block, err := aes.NewCipher(n.symmetricKey[:])
	if err != nil {
		return nil, err
	}
	aead, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	var nonce [12]byte
	binary.BigEndian.PutUint32(nonce[:4], n.nonce)
	n.nonce++

	plaintext, err := aead.Open(nil, nonce[:], ciphertext, n.handshakeHash[:])
	if err != nil {
		return nil, err
	}
	n.mixHash(ciphertext)
	return plaintext, nil
}

plaintext, err := noiseState.decryptAndHash(ciphertext)
if err != nil {
	panic(err)
}
if len(plaintext) != 0 {
	panic("unexpected handshake response payload")
}
```

最後に `split` して、後続の tunnel message で使う `writeKey` / `readKey` を導出します。

```text
# Split()
out = HKDF(
  secret = empty,
  salt = noiseState.chainingKey,
  length = 64
)

writeKey = out[0:32]
readKey  = out[32:64]
```

Go だとこうなります。

```go
type trafficKeys struct {
	WriteKey [32]byte
	ReadKey  [32]byte
}

func (n *noiseState) split() (trafficKeys, error) {
	var keys trafficKeys
	out, err := hkdfExpand(nil, n.chainingKey[:], 64)
	if err != nil {
		return keys, err
	}
	copy(keys.WriteKey[:], out[:32])
	copy(keys.ReadKey[:], out[32:64])
	return keys, nil
}

keys, err := noiseState.split()
if err != nil {
	panic(err)
}

handshakeHash := noiseState.handshakeHash
```

固定値の response で全体を通すと以下のようになります。

```go
response := decodeHex("043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b5605d73235109111433fade982b0918e17")

responderEPublic := response[:65]
ciphertext := response[65:]

noiseState.mixHash(responderEPublic)
if err := noiseState.mixKey(responderEPublic); err != nil {
	panic(err)
}

x, y := elliptic.Unmarshal(elliptic.P256(), responderEPublic)
if x == nil || y == nil {
	panic("invalid peer public key")
}
peerPublic := &ecdsa.PublicKey{Curve: elliptic.P256(), X: x, Y: y}

ee, err := ecdhSecret(ephemeral, peerPublic)
if err != nil {
	panic(err)
}
if err := noiseState.mixKey(ee); err != nil {
	panic(err)
}

se, err := ecdhSecret(identityKey, peerPublic)
if err != nil {
	panic(err)
}
if err := noiseState.mixKey(se); err != nil {
	panic(err)
}

plaintext, err := noiseState.decryptAndHash(ciphertext)
if err != nil {
	panic(err)
}
if len(plaintext) != 0 {
	panic("unexpected handshake response payload")
}

keys, err := noiseState.split()
if err != nil {
	panic(err)
}
handshakeHash := noiseState.handshakeHash

fmt.Printf("%x\n", handshakeHash[:])
fmt.Printf("%x\n", keys.WriteKey[:])
fmt.Printf("%x\n", keys.ReadKey[:])
```

### Authenticator からの getInfo reply

Hybrid Transport ではこの時点で Authenticator から getInfo の reply として Authenticator の情報が送られてきます。

```
d63919aa1962f73b948ae3a558d4b2d8a3bf069fe74a722647cea87fc0ef23794985ec6f70158ce4dc6965280bc14bc6ec6e59fa9648d74598abe564a5dbc13b651f1bb68aef38d85c9e75e4f4719295aee837d864630d4e76974d1db9251ae3d0947a209e3d03a845bbb3f9542d83f770e5e194d6dfa33b3eb219f99a40e57d8364d0f7020cb65bf506087965d5efd2
```

暗号化されているので復号します。

復号のために Authenticator から受信したデータを復号するための鍵と、Authenticator に送信するデータを暗号化するための鍵を導出します。

```
PRK = HMAC-SHA256(
  key     = final noiseState.chainingKey,
  message = empty
)

# Expand
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || empty || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || empty || 0x02)

transport keys = T(1) || T(2)
writeKey = transport keys[0:32]
readKey  = transport keys[32:64]
```

readKey を使って復号します。最初のメッセージなのでシーケンスが 0 になるため、nonce も 0 です。

```
AES-GCM-Open(
  key = readKey,
  nonce = transport receive nonce = 000000000000000000000000,
  ciphertext = "d63919aa1962f73b948ae3a558d4b2d8a3bf069fe74a722647cea87fc0ef23794985ec6f70158ce4dc6965280bc14bc6ec6e59fa9648d74598abe564a5dbc13b651f1bb68aef38d85c9e75e4f4719295aee837d864630d4e76974d1db9251ae3d0947a209e3d03a845bbb3f9542d83f770e5e194d6dfa33b3eb219f99a40e57d8364d0f7020cb65bf506087965d5efd2",
  associatedData = empty
)
```

復号結果はこうなります。

```
a2015861a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964038264637461706264630000000000000000000000000000000010
```

Go だと以下です。

```go
func decryptTunnelPadded(key []byte, seq uint32, ciphertext []byte) ([]byte, error) {
	block, err := aes.NewCipher(key)
	if err != nil {
		return nil, err
	}
	aead, err := cipher.NewGCM(block)
	if err != nil {
		return nil, err
	}

	var nonce [12]byte
	binary.BigEndian.PutUint32(nonce[8:], seq)

	return aead.Open(nil, nonce[:], ciphertext, nil)
}

getInfoCiphertext := decodeHex("d63919aa1962f73b948ae3a558d4b2d8a3bf069fe74a722647cea87fc0ef23794985ec6f70158ce4dc6965280bc14bc6ec6e59fa9648d74598abe564a5dbc13b651f1bb68aef38d85c9e75e4f4719295aee837d864630d4e76974d1db9251ae3d0947a209e3d03a845bbb3f9542d83f770e5e194d6dfa33b3eb219f99a40e57d8364d0f7020cb65bf506087965d5efd2")

paddedPlaintext, err := decryptTunnelPadded(keys.ReadKey[:], 0, getInfoCiphertext)
if err != nil {
	panic(err)
}

fmt.Printf("%x\n", paddedPlaintext)
```

最後の byte +1 が padding 長になるので padding を外すと

```
a2015861a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c6668796272696403826463746170626463
```

となります。

```go
func removePadding(padded []byte) ([]byte, error) {
	if len(padded) == 0 {
		return nil, errors.New("padded message is empty")
	}
	paddingLen := int(padded[len(padded)-1]) + 1
	if paddingLen > len(padded) {
		return nil, errors.New("invalid padding length")
	}
	return padded[:len(padded)-paddingLen], nil
}

plaintext, err := removePadding(paddedPlaintext)
if err != nil {
	panic(err)
}

fmt.Printf("%x\n", plaintext)
```

これは CBOR なので分解するとこうなります。

```text
a2 # 10100010 で先頭が 101 なので map、00010 なので 2 pairs
  01 # キー。00000001 で unsigned integer 1
  58 61 # 01011000 で先頭が 010 なので byte string、11000 なので長さは次の 1 byte。0x61 = 97 bytes
  a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964
     # キー 1 の値。97 bytes の byte string。この中身は getInfo reply の CBOR

  03 # キー。00000011 で unsigned integer 3
  82 # 10000010 で先頭が 100 なので array、00010 なので 2 elements
    64 63746170 # 01100100 で先頭が 011 なので text string、00100 なので長さ 4 bytes。63746170 は ASCII で "ctap"
    62 6463     # 01100010 で先頭が 011 なので text string、00010 なので長さ 2 bytes。6463 は ASCII で "dc"
```

この CBOR の定義は[11.5.1.2. Data Transfer](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#hybrid-data-transfer)を参照してください。

#### getInfo reply message

キー 1 の値はまた CBOR です。

```text
a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964
```

更にパースするとこうなります。

```text
a5 # 10100101 で先頭が 101 なので map、00101 なので 5 pairs
  01 # キー。unsigned integer 1
  82 # 10000010 で先頭が 100 なので array、00010 なので 2 elements
    68 4649444f5f325f30 # 01101000 で text string、01000 なので長さ 8 bytes。ASCII で "FIDO_2_0"
    68 4649444f5f325f31 # 01101000 で text string、01000 なので長さ 8 bytes。ASCII で "FIDO_2_1"

  02 # キー。unsigned integer 2
  82 # array、2 elements
    69 6c61726765426c6f62 # text string、長さ 9 bytes。ASCII で "largeBlob"
    63 707266             # text string、長さ 3 bytes。ASCII で "prf"

  03 # キー。unsigned integer 3
  50 f24a8e70d0d3f82c293732523cc4de5a
     # 01010000 で byte string、10000 なので長さ 16 bytes。AAGUID

  04 # キー。unsigned integer 4
  a3 # map、3 pairs。options
    62 726b f5
       # key は text string 長さ 2 bytes の "rk"。値 f5 は simple value true
    62 7576 f5
       # key は text string 長さ 2 bytes の "uv"。値 true
    6c 6a736f6e4d65737361676573 f5
       # key は text string 長さ 12 bytes の "jsonMessages"。値 true

  09 # キー。unsigned integer 9
  82 # array、2 elements。transports
    68 696e7465726e616c # text string、長さ 8 bytes。ASCII で "internal"
    66 687962726964     # text string、長さ 6 bytes。ASCII で "hybrid"
```

まとめるとこんな感じです。

```
{
  "1": ["FIDO_2_0", "FIDO_2_1"],
  "2": ["largeBlob", "prf"],
  "3": "f24a8e70d0d3f82c293732523cc4de5a",
  "4": {
    "rk": true,
    "uv": true,
    "jsonMessages": true
  },
  "9": ["internal", "hybrid"]
}
```

authenticatorGetInfo の定義は[6.4. authenticatorGetInfo (0x04)](https://fidoalliance.org/specs/fido-v2.3-ps-20260226/fido-client-to-authenticator-protocol-v2.3-ps-20260226.html#authenticatorGetInfo)を参照してください。

#### getAssertion

authenticatorGetAssertion のリクエストを組み立てます。

今回は rpId、clientDataHash、options からなるリクエストを組み立てます。
まず [clientDataJSON](https://www.w3.org/TR/webauthn-3/#dictdef-collectedclientdata) を組み立てます。

```
{
  "type": "webauthn.get",
  "challenge": "LO8RyAt5P_f3yB4b28mgk5A6dKYsNlvRd4ONXTeEGlM",
  "origin": "https://example.com"
}
```

challenge は以下を Base64URL encode したものです。

```
2cef11c80b793ff7f7c81e1bdbc9a093903a74a62c365bd177838d5d37841a53
```

これを SHA-256 でハッシュすると clientDataHash となります。

```
c6d10de1fc2d9bcffe91bd86800f1901760338fd70ca09c9e0c5b163a6ef0fdd
```

```go
challenge := decodeHex("2cef11c80b793ff7f7c81e1bdbc9a093903a74a62c365bd177838d5d37841a53")
challengeB64 := base64.RawURLEncoding.EncodeToString(challenge)
fmt.Println(challengeB64)
clientDataJSON := []byte(`{"type":"webauthn.get","challenge":"LO8RyAt5P_f3yB4b28mgk5A6dKYsNlvRd4ONXTeEGlM","origin":"https://example.com"}`)
clientDataHash := sha256.Sum256(clientDataJSON)
fmt.Printf("%x\n", clientDataHash[:])
```

```json
{
  "rpId": "example.com",
  "clientDataHash": "c6d10de1fc2d9bcffe91bd86800f1901760338fd70ca09c9e0c5b163a6ef0fdd",
  "options": {
    "uv": true
  }
}
```

こんな感じのリクエストを CBOR で組み立てます。

```text
02 # CTAP command byte。0x02 = authenticatorGetAssertion
a3 # 10100011 で先頭が 101 なので map、00011 なので 3 pairs
  01 # キー。unsigned integer 1。CTAP getAssertion では rpId
  6b 6578616d706c652e636f6d
     # 01101011 で先頭が 011 なので text string、01011 なので長さ 11 bytes。ASCII で "example.com"

  02 # キー。unsigned integer 2。CTAP getAssertion では clientDataHash
  58 20 c6d10de1fc2d9bcffe91bd86800f1901760338fd70ca09c9e0c5b163a6ef0fdd
     # 01011000 で先頭が 010 なので byte string、11000 なので長さは次の 1 byte。0x20 = 32 bytes

  05 # キー。unsigned integer 5。CTAP getAssertion では options
  a1 # 10100001 で先頭が 101 なので map、00001 なので 1 pair
    62 7576 f5
       # key は 01100010 で text string、00010 なので長さ 2 bytes の "uv"。値 f5 は simple value true
```

まとめるとこうなります。

```
02a3016b6578616d706c652e636f6d025820c6d10de1fc2d9bcffe91bd86800f1901760338fd70ca09c9e0c5b163a6ef0fdd05a1627576f5
```

CTAP request に tunnel message type の `01` をつけて padding したものを writeKey で暗号化します。client から送る最初のリクエストなので writeSeq は 0 です。

```go
tunnelPlaintext := append([]byte{0x01}, ctapRequest...)
tunnelPadded := addPadding(tunnelPlaintext)

fmt.Printf("%x\n", tunnelPlaintext)
fmt.Printf("%x\n", tunnelPadded)
```

padding は 32 bytes 境界に揃え、最後の byte に `padding length - 1` を入れます。

```go
func addPadding(plaintext []byte) []byte {
	paddingLen := 32 - (len(plaintext) % 32)
	if paddingLen == 0 {
		paddingLen = 32
	}

	out := make([]byte, len(plaintext)+paddingLen)
	copy(out, plaintext)
	out[len(out)-1] = byte(paddingLen - 1)
	return out
}
```

```
AES-GCM-Seal(
  key = writeKey,
  nonce = transport send nonce = 000000000000000000000000,
  plaintext = tunnel.padded.out,
  associatedData = empty
)=235220c6e1ef1d414abf8a7ce43032f65fabe42c59911018c5ab1a8f71a853a07aa6560bfb50f483e27d8335537e8d2a531e599cd7d8ab9ad279218ab442f5996be8dec747bf3b87613a321481fc26cd
```

となります。最後の16bytesはタグです。これを tunnel に流します。`writeKey` は Noise handshake 後の `split()` で得た client -> authenticator 用の鍵です。

```go
block, err := aes.NewCipher(writeKey)
if err != nil {
	panic(err)
}

aead, err := cipher.NewGCM(block)
if err != nil {
	panic(err)
}

var nonce [12]byte
binary.BigEndian.PutUint32(nonce[8:], 0) // writeSeq = 0

ciphertext := aead.Seal(nil, nonce[:], tunnelPadded, nil)
fmt.Printf("%x\n", ciphertext)
```

## getAssertion Response

Authenticator から来たレスポンスです。

```
860cb6a01646c8d16c644add796b3d80f7d6e2f32f1f7bc866c48725aefab396f5d92b22970e831425e7ed6a5b3b8c878bf4431bde119a8bf0b06261526892d07ac40a70be983ee705ec7216035a60911e85d4d99c842c50977072f2dd03cb6ba4fad021464627056094856fa98323bc71b0bad68607aef0972c5ae6d776cb19e96a9bf1880f2e08e2c2b5f7425551f3e7f71a6781da28008a44e961d958323d96aa9abd5ca58089641731189f905d6fd2e15bfc97c5891b94ba59738dda2117265386f58e756293825cb7c1156cb24e59a80004b19aaed89724fe1f3710ce5a67caf8ac4a4cda6289dd1f403c2bbf83fd9b582969a22cb217b44996350a06da4b7eadfd8d026fe804f01c452b285645
```

暗号化されているので getInfo 同様復号します。2つ目のメッセージなので readSeq = 1 で nonce を作り、readKey で復号します。

```
AES-GCM-Open(
  key = readKey,
  nonce = 000000000000000000000001,
  ciphertext = tunnel.ciphertext.in,
  associatedData = empty
)
```

復号すると CTAP response payload が取得できます。
Go だとこうです。`readKey` は Noise handshake 後の `split()` で得た authenticator -> client 用の鍵です。

```go
inboundCiphertext := decodeHex("860cb6a01646c8d16c644add796b3d80f7d6e2f32f1f7bc866c48725aefab396f5d92b22970e831425e7ed6a5b3b8c878bf4431bde119a8bf0b06261526892d07ac40a70be983ee705ec7216035a60911e85d4d99c842c50977072f2dd03cb6ba4fad021464627056094856fa98323bc71b0bad68607aef0972c5ae6d776cb19e96a9bf1880f2e08e2c2b5f7425551f3e7f71a6781da28008a44e961d958323d96aa9abd5ca58089641731189f905d6fd2e15bfc97c5891b94ba59738dda2117265386f58e756293825cb7c1156cb24e59a80004b19aaed89724fe1f3710ce5a67caf8ac4a4cda6289dd1f403c2bbf83fd9b582969a22cb217b44996350a06da4b7eadfd8d026fe804f01c452b285645")

block, err := aes.NewCipher(readKey)
if err != nil {
	panic(err)
}

aead, err := cipher.NewGCM(block)
if err != nil {
	panic(err)
}

var nonce [12]byte
binary.BigEndian.PutUint32(nonce[8:], 1) // readSeq = 1

paddedPlaintext, err := aead.Open(nil, nonce[:], inboundCiphertext, nil)
if err != nil {
	panic(err)
}

plaintext, err := removePadding(paddedPlaintext)
if err != nil {
	panic(err)
}

fmt.Printf("%x\n", plaintext)
```

padding を外す処理はこうです。

```go
func removePadding(padded []byte) ([]byte, error) {
	if len(padded) == 0 {
		return nil, errors.New("empty plaintext")
	}

	paddingLen := int(padded[len(padded)-1]) + 1
	if paddingLen > len(padded) {
		return nil, errors.New("invalid padding")
	}

	return padded[:len(padded)-paddingLen], nil
}
```

復号するとこうなります。
先頭の `00` は CTAP status byte で、その後ろの `a5...` からが CBOR です。

```
00a501a262696458366f5cd22b68bfcc3b29cd1ee9187d73316b222047cc9d6e75e34d54a3721dc717adba227a54096c31e3e6d2fd83740d468ccfb2dc11ba64747970656a7075626c69632d6b6579025825a379a6f6eeafb9a55e378c118034e2751e682fab9f2d30ab13d2125586ce19471d0000000003584830460221009fade36a5db2bc861bb1e7387fa79362cd8140983eabcb7386b60480c5272d610221009ca697ca15a2b4fbd7dacc7560022de97a148f456f9b6ad3b5580f21d9b8e3a504a362696458208f417f87ef436e26721795a92a22233e9c5a20ca858115db9bcc2d49c3c97982646e616d65606b646973706c61794e616d656006f5
```

アサーションが取得できました。

```text
00 # CTAP status byte。0x00 = CTAP1_ERR_SUCCESS
a5 # 10100101 で先頭が 101 なので map、00101 なので 5 pairs
  01 # キー。unsigned integer 1。CTAP getAssertion response では credential
  a2 # 10100010 で先頭が 101 なので map、00010 なので 2 pairs
    62 6964
       # key は 01100010 で text string、00010 なので長さ 2 bytes の "id"
    58 36 6f5cd22b68bfcc3b29cd1ee9187d73316b222047cc9d6e75e34d54a3721dc717adba227a54096c31e3e6d2fd83740d468ccfb2dc11ba
       # value は 01011000 で byte string、11000 なので長さは次の 1 byte。0x36 = 54 bytes
    64 74797065
       # key は 01100100 で text string、00100 なので長さ 4 bytes の "type"
    6a 7075626c69632d6b6579
       # value は 01101010 で text string、01010 なので長さ 10 bytes。ASCII で "public-key"

  02 # キー。unsigned integer 2。CTAP getAssertion response では authenticatorData
  58 25 a379a6f6eeafb9a55e378c118034e2751e682fab9f2d30ab13d2125586ce19471d00000000
     # 01011000 で byte string、11000 なので長さは次の 1 byte。0x25 = 37 bytes

  03 # キー。unsigned integer 3。CTAP getAssertion response では signature
  58 48 30460221009fade36a5db2bc861bb1e7387fa79362cd8140983eabcb7386b60480c5272d610221009ca697ca15a2b4fbd7dacc7560022de97a148f456f9b6ad3b5580f21d9b8e3a5
     # 01011000 で byte string、11000 なので長さは次の 1 byte。0x48 = 72 bytes

  04 # キー。unsigned integer 4。CTAP getAssertion response では user
  a3 # 10100011 で先頭が 101 なので map、00011 なので 3 pairs
    62 6964
       # key は text string 長さ 2 bytes の "id"
    58 20 8f417f87ef436e26721795a92a22233e9c5a20ca858115db9bcc2d49c3c97982
       # value は byte string、0x20 = 32 bytes
    64 6e616d65
       # key は text string 長さ 4 bytes の "name"
    60
       # value は 01100000 で text string、00000 なので長さ 0 bytes。空文字列 ""
    6b 646973706c61794e616d65
       # key は text string 長さ 11 bytes の "displayName"
    60
       # value は text string 長さ 0 bytes。空文字列 ""

  06 # キー。unsigned integer 6。CTAP getAssertion response では userSelected
  f5 # simple value true
```

CTAP response payload は Go だとこう分けられます。

```go
ctapResponse := decodeHex("00a501a262696458366f5cd22b68bfcc3b29cd1ee9187d73316b222047cc9d6e75e34d54a3721dc717adba227a54096c31e3e6d2fd83740d468ccfb2dc11ba64747970656a7075626c69632d6b6579025825a379a6f6eeafb9a55e378c118034e2751e682fab9f2d30ab13d2125586ce19471d0000000003584830460221009fade36a5db2bc861bb1e7387fa79362cd8140983eabcb7386b60480c5272d610221009ca697ca15a2b4fbd7dacc7560022de97a148f456f9b6ad3b5580f21d9b8e3a504a362696458208f417f87ef436e26721795a92a22233e9c5a20ca858115db9bcc2d49c3c97982646e616d65606b646973706c61794e616d656006f5")

ctapStatus := ctapResponse[0]
ctapCBORBody := ctapResponse[1:]

fmt.Printf("%02x\n", ctapStatus)
fmt.Printf("%x\n", ctapCBORBody)
```

`ctapStatus` が `00` なら成功です。

CBOR body だけを map として読むならこうです。

```go
var body map[uint64]cbor.RawMessage
if err := cbor.Unmarshal(ctapCBORBody, &body); err != nil {
	panic(err)
}

fmt.Println(body[1] != nil) // credential
fmt.Println(body[2] != nil) // authenticatorData
fmt.Println(body[3] != nil) // signature
fmt.Println(body[4] != nil) // user
fmt.Println(body[6] != nil) // userSelected
```
