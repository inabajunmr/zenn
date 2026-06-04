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

QR コードには以下の値が含まれています。

| key | value                                                                                           | desc                                                                                                                                                                                         |
| --- | ----------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 0   | 公開鍵                                                                                          | トンネル経由でクライアントプラットフォームと認証器がハンドシェイクを行うのに使う                                                                                                             |
| 1   | QR secret                                                                                       | BLE advertise の暗号化、復号、認証に使う                                                                                                                                                     |
| 2   | クライアントプラットフォームが知ってる tunnel service のドメインの数 ※1                         | これが 2 だったらこのプラットフォームは `cable.ua5v.com` と `cable.auth.com` を知っていることになり、後述の BLE advert で 0 を指定すれば前者に、1 を指定すれば後者を利用することが表現できる |
| 3   | 現在時刻                                                                                        |
| 4   | プラットフォームが state-assisted transactions に対応しているかどうか（省略可。この例では省略） |
| 5   | 今後行われる操作についてのヒント（認証 or 登録）                                                |
| 6   | トランスポートに使われるチャネル                                                                |

### CBOR の組み立て

以下の値で CBOR を組み立てます。

| 項目           | 値                                                                   |
| -------------- | -------------------------------------------------------------------- |
| QR Secret      | `97aa809b276911764f20b2921b6c7539`                                   |
| 秘密鍵         | `b85bd584301fd9d938307a0a7d63608ce6272671052023c2e85dad1efc90707f`   |
| 公開鍵         | `021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049` |
| タイムスタンプ | `1780082784`                                                         |
| ヒント         | `ga`                                                                 |
| チャネル       | `0`                                                                  |

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
  00                     GREASE
```

これを digitEncode します。

1. CBOR bytes を最大 7 bytes ごとに区切る
2. 各 chunk を little-endian uint64 として読む
3. 10 進文字列にする
4. chunk 長ごとの固定桁数まで左を `0` で埋める
5. 連結する

CBOR の先頭 7 bytes は
a7 00 58 21 02 1b bb
となります。
これを 8 bytes buffer にいれて
a7 00 58 21 02 1b bb 00
とし、これを little-endian uint64 として読むと下位バイトから読むことになるので
00 bb 1b 02 21 58 00 a7
となり、これを 10 進数にすると
52665516608192679
となります。

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
	publicKey := mustHex("021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049")
	qrSecret := mustHex("97aa809b276911764f20b2921b6c7539")

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

	fmt.Printf("%x\n", cborBytes)
	fmt.Println("FIDO:/" + digitEncode(cborBytes))
}

func mustHex(s string) []byte {
	out, err := hex.DecodeString(s)
	if err != nil {
		panic(err)
	}
	return out
}

func digitEncode(d []byte) string {
	const chunkSize = 7
	const chunkDigits = 17
	const zeros = "00000000000000000"

	var ret string
	for len(d) >= chunkSize {
		var chunk [8]byte
		copy(chunk[:], d[:chunkSize])
		v := strconv.FormatUint(binary.LittleEndian.Uint64(chunk[:]), 10)
		ret += zeros[:chunkDigits-len(v)]
		ret += v

		d = d[chunkSize:]
	}

	if len(d) != 0 {
		// partialChunkDigits is the number of digits needed to encode
		// each length of trailing data from 6 bytes down to zero. I.e.
		// it's 15, 13, 10, 8, 5, 3, 0 written in hex.
		const partialChunkDigits = 0x0fda8530

		digits := 15 & (partialChunkDigits >> (4 * len(d)))
		var chunk [8]byte
		copy(chunk[:], d)
		v := strconv.FormatUint(binary.LittleEndian.Uint64(chunk[:]), 10)
		ret += zeros[:digits-len(v)]
		ret += v
	}

	return ret
}
```

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
qrSecret := mustHex("97aa809b276911764f20b2921b6c7539")
eidKey, err := derive(qrSecret, nil, keyPurposeEIDKey, 64)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", eidKey)
```

`derive` は CTAP2 のサンプルコードそのままです。

```go
type keyPurpose int

const (
	keyPurposeEIDKey keyPurpose = 1
	keyPurposeTunnelID keyPurpose = 2
	keyPurposePSK keyPurpose = 3
)

func derive(ikm, salt []byte, purpose keyPurpose, n int) ([]byte, error) {
	var purpose32 [4]byte
	purpose32[0] = byte(purpose)

	reader := hkdf.New(sha256.New, ikm, salt, purpose32[:])
	ret := make([]byte, n)
	_, err := io.ReadFull(reader, ret)
	return ret, err
}
```

OKM の前半を使って暗号文を復号します。
OKM の後半は HMAC tag の検証に使います。

```
AES-Decrypt(OKM[0:32], 9f4bc7c7ed77b8a021ba504084f19c55)
```

CTAP2 のサンプルコードでは tag の検証と AES 復号がこのように記載されています。

```go
func trialDecrypt(eidKey *[64]byte, ciphertext []byte) []byte {
	if len(ciphertext) != 20 {
		return nil
	}

	mac := hmac.New(sha256.New, eidKey[32:])
	mac.Write(ciphertext[:16])
	if expectedMAC := mac.Sum(nil); !bytes.Equal(expectedMAC[:4], ciphertext[16:]) {
		return nil
	}

	block, err := aes.NewCipher(eidKey[:32])
	if err != nil {
		panic(err)
	}

	var ret [16]byte
	block.Decrypt(ret[:], ciphertext[:16])
	if ret[0] != 0 {
		return nil
	}

	return ret[:]
}
```

結果が `ab2a9e89` なら、BLE Advert の末尾 4 bytes と一致します。

BLE Advert を復号すると

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

TODO

## Tunnel Service への接続

ここまでに取得した値で Tunnel service の URL を組み立てることができます。

```
wss://cable.auth.com/cable/connect/4d5888/95074c89b06c2eb6bd4d79566d38dbf0
```

95074c89b06c2eb6bd4d79566d38dbf0 は QR Secret から導出できます。

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

Go だと以下です。

```go
qrSecret := mustHex("97aa809b276911764f20b2921b6c7539")
tunnelID, err := derive(qrSecret, nil, keyPurposeTunnelID, 16)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", tunnelID)
```

## Noise handshake

### Initial message

Noise Protocol Framework の [KNpsk0](https://noiseexplorer.com/patterns/KNpsk0/) は以下のように定義されています。

```text
KNpsk0:
  -> s
  ...
  -> psk, e
  <- e, ee, se
```

ここで s はすでに Client(Initiator) から Authenticator(Responder) に QR コードで通知した公開鍵です。

PSK は QR Secret と復号した BLE Advert

```
006832febb84eab2ff632c4d58880100
```

から導出します。

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
qrSecret := mustHex("97aa809b276911764f20b2921b6c7539")
advertPlaintext := mustHex("006832febb84eab2ff632c4d58880100")

psk, err := derive(qrSecret, advertPlaintext, keyPurposePSK, 32)
if err != nil {
	panic(err)
}
fmt.Printf("%x\n", psk)
```

ephemeral なキーペアを生成します。
秘密鍵

```
ephemeral private scalar = 9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16
```

P-256 の base point と ephemeral private scalar をかけると公開鍵が取得できます。

```
ephemeral public point = 9b4d1b8f9de3f44af2c3e9b41377fa23deb7e9582f3cbab6d9dbd3ac4c66cb16 * P-256 base point
```

ephemeral public point を X9.62 でエンコードすると e(ephemeral public key) がでます。

```
0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653
```

最初のメッセージにペイロードはないので空の payload を暗号化して AES-GCM の AEAD タグをつけます。

Noise State

```
// 32 Bytes 以下なので 0 padding
chainingKey = "Noise_KNpsk0_P256_AESGCM_SHA256" || 00
handshakeHash = "Noise_KNpsk0_P256_AESGCM_SHA256" || 00
```

```text
// mixHash(01) QR initiated の場合
handshakeHash = SHA256(handshakeHash || 01)

// mixHash(uncompressed client static public key)
x,y = uncompress(h'021bbbd2af0fab1596103e86e66aa242b0e6d935c8982229fe106aecdef3542049')
handshakeHash = SHA256(handshakeHash || 04 || x || y)

```

```
# Extract
PRK = HMAC-SHA256(chainingKey, psk)

# Expand
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || empty || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || empty || 0x02)
T(3) = HMAC-SHA256(PRK, T(2) || empty || 0x03)

out = T(1) || T(2) || T(3)

chainingKey  = out[0:32]   = T(1)
tempHash     = out[32:64]  = T(2)
symmetricKey = out[64:96]  = T(3)

nonce = 0

handshakeHash = SHA256(handshakeHash || tempHash)
```

```
// mixHash(e public)
handshakeHash = SHA256(handshakeHash || h'0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653')

// mixKey(e public)
PRK = HMAC-SHA256(chainingKey, h'0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653')

# Expand
T(0) = empty
T(1) = HMAC-SHA256(PRK, T(0) || empty || 0x01)
T(2) = HMAC-SHA256(PRK, T(1) || empty || 0x02)

out = T(1) || T(2)

state.chainingKey  = out[0:32]  = T(1)
state.symmetricKey = out[32:64] = T(2)
state.nonce        = 0
```

```
// AEAD tag
AES-GCM-Seal(
  key = current symmetricKey,
  nonce = 000000000000000000000000,
  plaintext = empty,
  associatedData = current handshakeHash
)
= f8f93344c8213dd9e0e6c67c29139e
```

payload が空なので暗号文はなしです。
ephemeral public key と AEAD tag を結合して noise の最初のメッセージができます。

0469361c21990613e68103b62480f865b2fac4e9101a7bc83a0aa6bbb7d17988ccae1833a49ae91cca2200ed650d0ee1ac0030e7eabb1b7128bb0884362da2236653 || f8f93344c8213dd9e0e6c67c29139e

### Response

Initial message に対して Authenticator が返してきたメッセージが

```
043e506f57ca06d713b770fb99560f6db4d9ff8cff7c55f96220f48dbe635ced8dd95cae1a8c6d8f273a297e7c00ac6872badfa377f872dc8c076c21679ce19b5605d73235109111433fade982b0918e17
```

となります。

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

### Authenticator からの getInfo reply

Hybrid Transport ではこの時点で Authenticator から getInfo の reply として Authenticator の情報が送られてきます。

```
d63919aa1962f73b948ae3a558d4b2d8a3bf069fe74a722647cea87fc0ef23794985ec6f70158ce4dc6965280bc14bc6ec6e59fa9648d74598abe564a5dbc13b651f1bb68aef38d85c9e75e4f4719295aee837d864630d4e76974d1db9251ae3d0947a209e3d03a845bbb3f9542d83f770e5e194d6dfa33b3eb219f99a40e57d8364d0f7020cb65bf506087965d5efd2
```

暗号化されているので復号します。

復号のために Authenticator から受信したデータを復号するための鍵と、Authenticator に送信するデータを暗号化するための鍵を導出します。

```
PRK = HMAC-SHA256(
  key     = final chainingKey,
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

最後の byte +1 が padding 長になるので padding を外すと

```
a2015861a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c6668796272696403826463746170626463
```

となります。これは CBOR なので分解するとこうなります。

CBOR 分解:

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

先頭 `a2` は map 2 pairs です。

### post-handshake key 1: getInfo reply

```text
01
58 61
a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964
```

読み方:

```text
01      key 1
58 61   byte string, length = 0x61 = 97
...     getInfo reply CBOR body
```

値:

```text
getInfo reply:
a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964
```

### post-handshake key 3: features

```text
03
82 6463746170 626463
```

読み方:

```text
03          key 3
82          array 2 elements
6463746170  text string length 4: "ctap"
626463      text string length 2: "dc"
```

値:

```text
features = ["ctap", "dc"]
```

ctap と dc のメッセージが喋れるようです。
dc は Digital Credentials 用の JSON message です。

### CBOR の getInfo reply message

ログ:

```text
Post-handshake getInfo: a50182684649444f5f325f30684649444f5f325f310282696c61726765426c6f62637072660350f24a8e70d0d3f82c293732523cc4de5a04a362726bf5627576f56c6a736f6e4d65737361676573f5098268696e7465726e616c66687962726964
```

CBOR:

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

先頭 `a5` は map 5 pairs です。

### getInfo key 1: versions

```text
01
82
  684649444f5f325f30
  684649444f5f325f31
```

値:

```text
versions = ["FIDO_2_0", "FIDO_2_1"]
```

### getInfo key 2: extensions

```text
02
82
  696c61726765426c6f62
  63707266
```

値:

```text
extensions = ["largeBlob", "prf"]
```

### getInfo key 3: AAGUID

```text
03
50 f24a8e70d0d3f82c293732523cc4de5a
```

値:

```text
aaguid = f24a8e70d0d3f82c293732523cc4de5a
```

### getInfo key 4: options

```text
04
a3
  62726b f5
  627576 f5
  6c6a736f6e4d65737361676573 f5
```

値:

```text
options = {
  "rk": true,
  "uv": true,
  "jsonMessages": true
}
```

### getInfo key 9: transports

```text
09
82
  68696e7465726e616c
  66787962726964
```

値:

```text
transports = ["internal", "hybrid"]
```

まとめるとこんな感じ

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

```
{
  "versions": [
    "FIDO_2_0",
    "FIDO_2_1"
  ],
  "extensions": [
    "largeBlob",
    "prf"
  ],
  "aaguid": "f24a8e70d0d3f82c293732523cc4de5a",
  "options": {
    "rk": true,
    "uv": true,
    "jsonMessages": true
  },
  "transports": [
    "internal",
    "hybrid"
  ]
}
```

## getAssertion

authenticatorGetAssertion のリクエストを組み立てます。
https://fidoalliance.org/specs/fido-v2.1-rd-20191217/fido-client-to-authenticator-protocol-v2.1-rd-20191217.html#authenticatorGetAssertion

今回は rpId、clientDataHash、options からなるリクエストを組み立てます。
まず clientDataJSON を組み立てます。

```
{
  "type": "webauthn.get",
  "challenge": "LO8RyAt5P_f3yB4b28mgk5A6dKYsNlvRd4ONXTeEGlM",
  "origin": "https://example.com"
}
```

これを SHA-256 でハッシュすると clientDataHash となります。

```
c6d10de1fc2d9bcffe91bd86800f1901760338fd70ca09c9e0c5b163a6ef0fdd
```

challenge bytes から base64url にするならこうです。

```go
challenge := mustHex("2cef11c80b793ff7f7c81e1bdbc9a093903a74a62c365bd177838d5d37841a53")
challengeB64 := base64.RawURLEncoding.EncodeToString(challenge)
fmt.Println(challengeB64)
```

clientDataHash はこう確認できます。

```go
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

CBOR 部分を Go で作るとこうです。

```go
canonicalCBOR, err := cbor.CanonicalEncOptions().EncMode()
if err != nil {
	panic(err)
}

ctapMap := map[uint64]any{
	1: "example.com",
	2: clientDataHash[:],
	5: map[string]bool{"uv": true},
}

ctapCBOR, err := canonicalCBOR.Marshal(ctapMap)
if err != nil {
	panic(err)
}

ctapRequest := append([]byte{0x02}, ctapCBOR...)
fmt.Printf("%x\n", ctapRequest)
```

CTAP request に tunnel message type の `01` をつけて padding したものを writeKey で暗号化します。client から送る最初のリクエストなので writeSeq は 0 で、

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

となります。最後の16bytesはタグです。これを tunnel に流します。

Go だとこうです。`writeKey` は Noise handshake 後の `split()` で得た client -> authenticator 用の鍵です。

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

復号すると CTAP response payload が取得できます。先頭の `00` は CTAP status byte で、その後ろの `a5...` からが CBOR です。

Go だとこうです。`readKey` は Noise handshake 後の `split()` で得た authenticator -> client 用の鍵です。

```go
inboundCiphertext := mustHex("860cb6a01646c8d16c644add796b3d80f7d6e2f32f1f7bc866c48725aefab396f5d92b22970e831425e7ed6a5b3b8c878bf4431bde119a8bf0b06261526892d07ac40a70be983ee705ec7216035a60911e85d4d99c842c50977072f2dd03cb6ba4fad021464627056094856fa98323bc71b0bad68607aef0972c5ae6d776cb19e96a9bf1880f2e08e2c2b5f7425551f3e7f71a6781da28008a44e961d958323d96aa9abd5ca58089641731189f905d6fd2e15bfc97c5891b94ba59738dda2117265386f58e756293825cb7c1156cb24e59a80004b19aaed89724fe1f3710ce5a67caf8ac4a4cda6289dd1f403c2bbf83fd9b582969a22cb217b44996350a06da4b7eadfd8d026fe804f01c452b285645")

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

```
00a501a262696458366f5cd22b68bfcc3b29cd1ee9187d73316b222047cc9d6e75e34d54a3721dc717adba227a54096c31e3e6d2fd83740d468ccfb2dc11ba64747970656a7075626c69632d6b6579025825a379a6f6eeafb9a55e378c118034e2751e682fab9f2d30ab13d2125586ce19471d0000000003584830460221009fade36a5db2bc861bb1e7387fa79362cd8140983eabcb7386b60480c5272d610221009ca697ca15a2b4fbd7dacc7560022de97a148f456f9b6ad3b5580f21d9b8e3a504a362696458208f417f87ef436e26721795a92a22233e9c5a20ca858115db9bcc2d49c3c97982646e616d65606b646973706c61794e616d656006f5
```

CTAP response payload は Go だとこう分けられます。

```go
ctapResponse := mustHex("00a501a262696458366f5cd22b68bfcc3b29cd1ee9187d73316b222047cc9d6e75e34d54a3721dc717adba227a54096c31e3e6d2fd83740d468ccfb2dc11ba64747970656a7075626c69632d6b6579025825a379a6f6eeafb9a55e378c118034e2751e682fab9f2d30ab13d2125586ce19471d0000000003584830460221009fade36a5db2bc861bb1e7387fa79362cd8140983eabcb7386b60480c5272d610221009ca697ca15a2b4fbd7dacc7560022de97a148f456f9b6ad3b5580f21d9b8e3a504a362696458208f417f87ef436e26721795a92a22233e9c5a20ca858115db9bcc2d49c3c97982646e616d65606b646973706c61794e616d656006f5")

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
