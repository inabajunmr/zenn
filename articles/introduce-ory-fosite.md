---
title: "Go 製の認可サーバー、IdP 実装用ライブラリ Fosite"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth2", "Hydra", "openidconnect", "ory", "go"]
published: false
---

本記事は [Digital Identity技術勉強会 #iddanceのカレンダー](https://qiita.com/advent-calendar/2021/iddance) 20日目の記事です。

# ory/fosite について

[https://github.com/ory/fosite](https://github.com/ory/fosite)

OAuth 2.0/OIDC の 認可サーバー、IdP 実装のための Go のライブラリです。
エンドポイントそのものは直接実装せず、プロトコルに関連するリクエストを http のハンドラー経由でライブラリに渡してあげるとパースしたり必要なものを永続化したりレスポンスを生成してくれます。

[Motivation]([https://github.com/ory/fosite#motivation](https://github.com/ory/fosite#motivation)) によると、そもそも Hydra 用のライブラリとして開発されたようです。
[Hydra](https://github.com/ory/hydra) については以前 [Hydra による OAuth 2.0 の認可サーバー/ OIDC の IdP の実装イメージ](https://zenn.dev/inabajunmr/articles/7f5f5a979f20b6)で紹介させていただきました。


# 検証環境

```go
$ go version
go version go1.17.2 darwin/arm64
$ cat go.mod
github.com/ory/fosite v0.41.0
```

# とりあえずクイックスタートを見ながら使ってみる

とりあえず[クイックスタート]([https://github.com/ory/fosite#quickstart](https://github.com/ory/fosite#quickstart))と[サンプル実装](https://github.com/ory/fosite-example)を見ながら認可エンドポイント、トークンエンドポイント、イントロスペクションを実装して動かしてみます。

## 初期設定

```go
package oauth2

import (
	"crypto/rand"
	"crypto/rsa"

	"github.com/ory/fosite/compose"
	"github.com/ory/fosite/storage"
)

var secret = []byte("my super secret signing password")
var privateKey, _ = rsa.GenerateKey(rand.Reader, 2048)
var config = &compose.Config{}
var oauth2Provider = compose.ComposeAllEnabled(config, storage.NewExampleStore(), secret, privateKey)
```

## 認可エンドポイントを定義

ログイン処理は自前で実装します。今回はクエリパラメーターに指定したユーザーがログインしているものとします。

```go
package oauth2

import (
	"net/http"

	"github.com/ory/fosite"
)

func AuthorizationEndpoint(rw http.ResponseWriter, req *http.Request) {

	ctx := req.Context()
	ar, err := oauth2Provider.NewAuthorizeRequest(ctx, req)
	if err != nil {
		oauth2Provider.WriteAuthorizeError(rw, ar, err)
		return
	}

	if req.URL.Query().Get("username") == "" {
		rw.Write([]byte(`set username as query parameter.`))
		return
	}

	mySessionData := &fosite.DefaultSession{
		Username: req.Form.Get("username"),
	}

	response, err := oauth2Provider.NewAuthorizeResponse(ctx, ar, mySessionData)
	if err != nil {
		oauth2Provider.WriteAuthorizeError(rw, ar, err)
		return
	}

	oauth2Provider.WriteAuthorizeResponse(rw, ar, response)
}
```

## トークンエンドポイントを定義

```go
package oauth2

import (
	"net/http"

	"github.com/ory/fosite"
)

func TokenEndpoint(rw http.ResponseWriter, req *http.Request) {
	ctx := req.Context()
	mySessionData := new(fosite.DefaultSession)
	accessRequest, err := oauth2Provider.NewAccessRequest(ctx, req, mySessionData)
	if err != nil {
		oauth2Provider.WriteAccessError(rw, accessRequest, err)
		return
	}
	response, err := oauth2Provider.NewAccessResponse(ctx, accessRequest)
	if err != nil {
		oauth2Provider.WriteAccessError(rw, accessRequest, err)
		return
	}

	oauth2Provider.WriteAccessResponse(rw, accessRequest, response)
}
```

## イントロスペクションエンドポイントを定義

```go
package oauth2

import (
	"log"
	"net/http"

	"github.com/ory/fosite"
)

func IntrospectionEndpoint(rw http.ResponseWriter, req *http.Request) {
	ctx := req.Context()
	ir, err := oauth2Provider.NewIntrospectionRequest(ctx, req, new(fosite.DefaultSession))
	if err != nil {
		log.Printf("Error occurred in NewIntrospectionRequest: %+v", err)
		oauth2Provider.WriteIntrospectionError(rw, err)
		return
	}
	oauth2Provider.WriteIntrospectionResponse(rw, ir)
}
```

## ハンドラーを定義

```go
package main

import (
	"log"
	"net/http"

	"github.com/inabajunmr/fosite-oauth-server-sample/oauth2"
)

func main() {
	http.HandleFunc("/oauth2/auth", oauth2.AuthorizationEndpoint)
	http.HandleFunc("/oauth2/token", oauth2.TokenEndpoint)
	http.HandleFunc("/oauth2/introspect", oauth2.IntrospectionEndpoint)
	log.Fatal(http.ListenAndServe(":3846", nil))
}
```

## 動かしてみる

認可リクエストしてみます。

```go
http://localhost:3846/oauth2/auth?client_id=my-client&response_type=code&state=64aa6f2d-52d1-ec96-04b7-832f8720e7a7&username=inaba&redirect_uri=http://localhost:3846/callback
```

認可レスポンスが返ってきました。

```go
http://localhost:3846/callback?code=NbAK7SwO-zzwOD-u_Qr8_hlVJfwelvMtq4287QvBtoI.CHnA8zT8occbmyux3Fo9iyL_sP_-EeJLmr16u6REm-E&scope=&state=64aa6f2d-52d1-ec96-04b7-832f8720e7a7
```

トークンリクエストしてみます。

```go
curl -d 'grant_type=authorization_code' -d 'redirect_uri=http://localhost:3846/callback' -d 'code=NbAK7SwO-zzwOD-u_Qr8_hlVJfwelvMtq4287QvBtoI.CHnA8zT8occbmyux3Fo9iyL_sP_-EeJLmr16u6REm-E' -u 'my-client:foobar' localhost:3846/oauth2/token
```

トークンレスポンスが返ってきました。

```go
{"access_token":"KSQutKDWU0kL1gZW2kDUQoPDeGjuJOgCUkdeUs8orkc.IEyAMdYP2H9gPpTaZXLQ1KSU-_WrcMbB7SMq1ktjDFs","expires_in":3600,"scope":"","token_type":"bearer"}
```

イントロスペクションしてみます。

```go
curl -d 'token=KSQutKDWU0kL1gZW2kDUQoPDeGjuJOgCUkdeUs8orkc.IEyAMdYP2H9gPpTaZXLQ1KSU-_WrcMbB7SMq1ktjDFs' -u 'my-client:foobar' localhost:3846/oauth2/introspect
```

イントロスペクションできました。

```go
{"active":true,"client_id":"my-client","exp":1639206763,"iat":1639203162,"username":"inaba"}
```

# ストレージをサンプルから自前の実装に切り替える

さきほどのサンプルでは以下のように初期化を行っていました。

```go
var secret = []byte("my super secret signing password")
var privateKey, _ = rsa.GenerateKey(rand.Reader, 2048)
var config = &compose.Config{}
var oauth2Provider = compose.ComposeAllEnabled(config, storage.NewExampleStore(), secret, privateKey)
```

ここでは [compose.ComposeAllEnabled](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go#L99) で全機能を有効可してかつ永続化層の実装にサンプル実装を利用しています。
全機能というのはつまり「特定のグラントタイプだけ提供する」とか「イントロスペクションを提供する」とかそういったものを選べるのですが、それらを全部有効にする、という意味です。
永続化層の実装にサンプル実装は [MemoryStore](https://github.com/ory/fosite/blob/4e2c03d3f6dcb3a3b50e7ea245128edde7ebf959/storage/memory.go#L108) で、ここではインメモリの実装 & デフォルトで設定されたクライアントを利用、といった構造になっています。

実際に利用する場合はおそらくインメモリではない実装が必要になるのでこれを自前で用意して上げる必要がありますが、これをやってみます。（ただしここでの実装は結局インメモリです）
全機能有効だと実装量が多くて大変なので「クライアントクレデンシャルでのアクセストークン発行」と「イントロスペクション」だけ有効にして実装します。

実際に書いたコードは [こちら](https://github.com/inabajunmr/fosite-oauth-server-sample) を参照ください。

## 何を実装すればよいのか？

[compose.Compose](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go) を利用して初期化する場合、こんな感じになります。構造は後述しますが、最後の引数に渡している XxxFactory によって有効にする機能が決まります。

```go
var oauth2Provider = compose.Compose(
	config,
	storage.NewInMemoryStorage(defaultClient()),
	&compose.CommonStrategy{
		CoreStrategy:               compose.NewOAuth2HMACStrategy(config, secret, nil),
		OpenIDConnectTokenStrategy: compose.NewOpenIDConnectStrategy(config, privateKey),
		JWTStrategy: &jwt.RS256JWTStrategy{
			PrivateKey: privateKey,
		},
	},
	nil,

	compose.OAuth2ClientCredentialsGrantFactory,
	compose.OAuth2TokenIntrospectionFactory,
)
```

この場合、「クライアントクレデンシャルでのアクセストークン発行」と「イントロスペクション」だけ有効となります。

[OAuth2ClientCredentialsGrantFactory](https://github.com/ory/fosite/blob/a6bfb921ebc746ba7a1215e32fb42a2c0530a2bf/compose/compose_oauth2.go#L50) の実装を追ってみると以下のようになっています。

```go
func OAuth2ClientCredentialsGrantFactory(config *Config, storage interface{}, strategy interface{}) interface{} {
	return &oauth2.ClientCredentialsGrantHandler{
		HandleHelper: &oauth2.HandleHelper{
			AccessTokenStrategy: strategy.(oauth2.AccessTokenStrategy),
			AccessTokenStorage:  storage.(oauth2.AccessTokenStorage),
			AccessTokenLifespan: config.GetAccessTokenLifespan(),
		},
		ScopeStrategy:            config.GetScopeStrategy(),
		AudienceMatchingStrategy: config.GetAudienceStrategy(),
	}
}
```

ここからクライアントクレデンシャルのためには [AccessTokenStorage](https://github.com/ory/fosite/blob/master/handler/oauth2/storage.go#L54) を実装する必要があることがわかります。

```go
type AccessTokenStorage interface {
	CreateAccessTokenSession(ctx context.Context, signature string, request fosite.Requester) (err error)

	GetAccessTokenSession(ctx context.Context, signature string, session fosite.Session) (request fosite.Requester, err error)

	DeleteAccessTokenSession(ctx context.Context, signature string) (err error)
}
```

イントロスペクションも同様に追いかけると [CoreStorage](https://github.com/ory/fosite/blob/72bff7f33ee8c3a4a8806cc266ca7299ff1785d4/handler/oauth2/storage.go#L30) のインターフェースを実装する必要があるようです。

```go
type CoreStorage interface {
	AuthorizeCodeStorage
	AccessTokenStorage
	RefreshTokenStorage
}

// AuthorizeCodeStorage handles storage requests related to authorization codes.
type AuthorizeCodeStorage interface {
	// GetAuthorizeCodeSession stores the authorization request for a given authorization code.
	CreateAuthorizeCodeSession(ctx context.Context, code string, request fosite.Requester) (err error)

	// GetAuthorizeCodeSession hydrates the session based on the given code and returns the authorization request.
	// If the authorization code has been invalidated with `InvalidateAuthorizeCodeSession`, this
	// method should return the ErrInvalidatedAuthorizeCode error.
	//
	// Make sure to also return the fosite.Requester value when returning the fosite.ErrInvalidatedAuthorizeCode error!
	GetAuthorizeCodeSession(ctx context.Context, code string, session fosite.Session) (request fosite.Requester, err error)

	// InvalidateAuthorizeCodeSession is called when an authorize code is being used. The state of the authorization
	// code should be set to invalid and consecutive requests to GetAuthorizeCodeSession should return the
	// ErrInvalidatedAuthorizeCode error.
	InvalidateAuthorizeCodeSession(ctx context.Context, code string) (err error)
}

type AccessTokenStorage interface {
	CreateAccessTokenSession(ctx context.Context, signature string, request fosite.Requester) (err error)

	GetAccessTokenSession(ctx context.Context, signature string, session fosite.Session) (request fosite.Requester, err error)

	DeleteAccessTokenSession(ctx context.Context, signature string) (err error)
}

type RefreshTokenStorage interface {
	CreateRefreshTokenSession(ctx context.Context, signature string, request fosite.Requester) (err error)

	GetRefreshTokenSession(ctx context.Context, signature string, session fosite.Session) (request fosite.Requester, err error)

	DeleteRefreshTokenSession(ctx context.Context, signature string) (err error)
}
```

[サンプル](https://github.com/ory/fosite/blob/4e2c03d3f6dcb3a3b50e7ea245128edde7ebf959/storage/memory.go)とあんまり変わりがないですが、インメモリで全部安直に実装するとこんな感じになりました。認可コード周りはインターフェース的には必要なんですが、今回は多分いらない気がしたので実装していません。

```go
package storage

import (
	"context"
	"sync"
	"time"

	"github.com/ory/fosite"
)

type InMemoryStorage struct {
	clients       sync.Map
	accessTokens  sync.Map
	refreshTokens sync.Map
}

func NewInMemoryStorage(defaultClient fosite.Client) *InMemoryStorage {
	s := InMemoryStorage{
		clients:       sync.Map{},
		accessTokens:  sync.Map{},
		refreshTokens: sync.Map{},
	}
	s.CreateClient(context.TODO(), defaultClient)
	return &s
}

func (s *InMemoryStorage) CreateClient(_ context.Context, client fosite.Client) {
	s.clients.Store(client.GetID(), client)
}

func (s *InMemoryStorage) GetClient(_ context.Context, id string) (fosite.Client, error) {
	client, ok := s.clients.Load(id)
	if ok {
		return client.(fosite.Client), nil
	}

	return nil, fosite.ErrNotFound
}

func (s *InMemoryStorage) ClientAssertionJWTValid(_ context.Context, jti string) error {
	return nil
}

func (s *InMemoryStorage) SetClientAssertionJWT(_ context.Context, jti string, exp time.Time) error {
	return nil
}

func (s *InMemoryStorage) CreateAccessTokenSession(ctx context.Context, signature string, request fosite.Requester) (err error) {
	s.accessTokens.Store(signature, request)
	return nil
}

func (s *InMemoryStorage) GetAccessTokenSession(ctx context.Context, signature string, session fosite.Session) (request fosite.Requester, err error) {
	at, ok := s.accessTokens.Load(signature)
	if ok {
		return at.(fosite.Requester), nil
	}

	return nil, fosite.ErrNotFound
}

func (s *InMemoryStorage) DeleteAccessTokenSession(ctx context.Context, signature string) (err error) {
	s.accessTokens.Delete(signature)
	return nil
}

func (s *InMemoryStorage) CreateAuthorizeCodeSession(ctx context.Context, code string, request fosite.Requester) (err error) {
	return nil
}

func (s *InMemoryStorage) GetAuthorizeCodeSession(ctx context.Context, code string, session fosite.Session) (request fosite.Requester, err error) {
	return nil, nil
}

func (s *InMemoryStorage) InvalidateAuthorizeCodeSession(ctx context.Context, code string) (err error) {
	return nil
}

func (s *InMemoryStorage) CreateRefreshTokenSession(ctx context.Context, signature string, request fosite.Requester) (err error) {
	s.refreshTokens.Store(signature, request)
	return nil
}

func (s *InMemoryStorage) GetRefreshTokenSession(ctx context.Context, signature string, session fosite.Session) (request fosite.Requester, err error) {
	at, ok := s.refreshTokens.Load(signature)
	if ok {
		return at.(fosite.Requester), nil
	}

	return nil, fosite.ErrNotFound
}

func (s *InMemoryStorage) DeleteRefreshTokenSession(ctx context.Context, signature string) (err error) {
	s.refreshTokens.Delete(signature)
	return nil
}
```

## 実装したコードを動かしてみる

アクセストークンを発行します。

```
curl -d 'grant_type=client_credentials' -u 'default-client:secret' localhost:3846/oauth2/toke
```

発行できました。

```
{"access_token":"uxrxObB9G49rHV6bPfviWHMuiLgimbOt9E9B3MQi6Nc.m1ySRCqfv3ZOxoT0WNlLQ6UGmQmuZEZ7bo8p-6yqruA","expires_in":3599,"scope":"","token_type":"bearer"}%
```

イントロスペクションします。

```
curl -d 'token=uxrxObB9G49rHV6bPfviWHMuiLgimbOt9E9B3MQi6Nc.m1ySRCqfv3ZOxoT0WNlLQ6UGmQmuZEZ7bo8p-6yqruA' -u 'default-client:secret' localhost:3846/oauth2/introspect
```

イントロスペクションできました。

```
{"active":true,"client_id":"default-client","exp":1639224068,"iat":1639220468}
```

# コードを眺めてみる

## 認可エンドポイント

fosite を利用して認可エンドポイントを実装する場合、いろいろはしょると以下のようになります。

```go
ar, err := oauth2Provider.NewAuthorizeRequest(ctx, req)
mySessionData := &fosite.DefaultSession{
	Username: "ログインしているユーザー",
}
response, err := oauth2Provider.NewAuthorizeResponse(ctx, ar, mySessionData)
oauth2Provider.WriteAuthorizeResponse(rw, ar, response)
```

リクエストをまるごと [NewAuthorizeRequest](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/authorize_request_handler.go#L278) に渡すと、fosite はリクエストをパースし OAuth2.0 や OIDC に関連するパラメーターを抽出しながら各パラメーターのバリデーションを行います。

一方 [NewAuthorizeResponse](https://github.com/ory/fosite/blob/2f96bb8a2623fe7b4abb31db870582b555df6db8/authorize_response_writer.go#L43) ではほとんどの処理をハンドラーに任せています。先程初期化で compose に XxxFactory を渡していましたが、ファクトリが生成するのがこのハンドラーとなります。

例えば [OAuth2AuthorizeImplicitFactory](https://github.com/ory/fosite/blob/a6bfb921ebc746ba7a1215e32fb42a2c0530a2bf/compose/compose_oauth2.go#L79) を渡すとここで [AuthorizeImplicitGrantTypeHandler](https://github.com/ory/fosite/blob/620d4c148307f7be7b2674fe420141b33aef6075/handler/oauth2/flow_authorize_implicit.go#L37) が実行される、といった具合になります。つまり、ここで初めてグラントタイプやその他（PKCE の利用や OIDC の有無など）の差が出てきます。

ハンドラーではハンドラー特有のバリデーションと必要な情報の永続化を行っているようです。

例えば PKCE 有りの認可コードフローの場合、 [OAuth2AuthorizeExplicitFactory](https://github.com/ory/fosite/blob/a6bfb921ebc746ba7a1215e32fb42a2c0530a2bf/compose/compose_oauth2.go#L31)  と [OAuth2PKCEFactory](https://github.com/ory/fosite/blob/a6bfb921ebc746ba7a1215e32fb42a2c0530a2bf/compose/compose_pkce.go#L30) を利用しますが、前者のハンドラーで認可コードの発行と永続化、後者のハンドラーで `code_challenge` と `code_challenge_method` を永続化したりしています。PKCE 用のパラメーターは認可コードと紐付けて永続化していますが、ハンドラーの順番は自動で制御されないようなので [compose.Compose](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go) に Factory を渡す順番には気をつける必要があります。おそらく [compose.ComposeAllEnabled](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go#L99) の順番を参考にすれば問題ないかと思います。

ざっと見たところ、認可エンドポイントに関わらず「NewXxxRequest でバリデーションなどの前処理」を行って「NewXxxResponse で永続化」というパターンになっています。ただし、ハンドラーによる固有の処理がどこにあるのか、はエンドポイントごとに差があります。

## トークンエンドポイント

トークンエンドポイントも利用時のコードは似たような感じです。

```go
accessRequest, err := oauth2Provider.NewAccessRequest(ctx, req, new(fosite.DefaultSession))
response, err := oauth2Provider.NewAccessResponse(ctx, accessRequest)
oauth2Provider.WriteAccessResponse(rw, accessRequest, response)
```

[NewAccessRequest](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/access_request_handler.go#L60) で、リクエストのパース、バリデーション、クライアント認証などの処理を行います。認可リクエストと違ってこちらでは共通的な処理の後ハンドラーごとの処理も行います。例えば [AuthorizeExplicitGrantHandler](https://github.com/ory/fosite/blob/6ad92642f0f01ff4d3662f3680a825db22594366/handler/oauth2/flow_authorize_code_auth.go#L78) であれば、認可コードから認可リクエストを取得したりリダイレクト URI の検証はここで行います。

一方 [NewAccessResponse](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/access_response_writer.go#L32) の処理はほとんどハンドラーが行っていて、主にアクセストークンの発行と永続化をしています。

## 初期設定について

```go
var oauth2Provider = compose.ComposeAllEnabled(config, storage.NewExampleStore(), secret, privateKey)
```

で初期化される oauthProvider の実態は各種ハンドラーが設定された [fosite.Fosite](https://github.com/ory/fosite/blob/master/fosite.go#L89) です。
[compose.Compose](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go) による [fosite.Fosite](https://github.com/ory/fosite/blob/master/fosite.go#L89) の初期化は以下のようになっています。

```go
func Compose(config *Config, storage interface{}, strategy interface{}, hasher fosite.Hasher, factories ...Factory) fosite.OAuth2Provider {
	if hasher == nil {
		hasher = &fosite.BCrypt{WorkFactor: config.GetHashCost()}
	}

	f := &fosite.Fosite{
		Store:                        storage.(fosite.Storage),
		AuthorizeEndpointHandlers:    fosite.AuthorizeEndpointHandlers{},
		TokenEndpointHandlers:        fosite.TokenEndpointHandlers{},
		TokenIntrospectionHandlers:   fosite.TokenIntrospectionHandlers{},
		RevocationHandlers:           fosite.RevocationHandlers{},
		Hasher:                       hasher,
		ScopeStrategy:                config.GetScopeStrategy(),
		AudienceMatchingStrategy:     config.GetAudienceStrategy(),
		SendDebugMessagesToClients:   config.SendDebugMessagesToClients,
		TokenURL:                     config.TokenURL,
		JWKSFetcherStrategy:          config.GetJWKSFetcherStrategy(),
		MinParameterEntropy:          config.GetMinParameterEntropy(),
		UseLegacyErrorFormat:         config.UseLegacyErrorFormat,
		ClientAuthenticationStrategy: config.GetClientAuthenticationStrategy(),
		ResponseModeHandlerExtension: config.ResponseModeHandlerExtension,
		MessageCatalog:               config.MessageCatalog,
	}

	for _, factory := range factories {
		res := factory(config, storage, strategy)
		if ah, ok := res.(fosite.AuthorizeEndpointHandler); ok {
			f.AuthorizeEndpointHandlers.Append(ah)
		}
		if th, ok := res.(fosite.TokenEndpointHandler); ok {
			f.TokenEndpointHandlers.Append(th)
		}
		if tv, ok := res.(fosite.TokenIntrospector); ok {
			f.TokenIntrospectionHandlers.Append(tv)
		}
		if rh, ok := res.(fosite.RevocationHandler); ok {
			f.RevocationHandlers.Append(rh)
		}
	}

	return f
}
```

config はいわゆる設定、storage は先程実装した `InMemoryStorage` のような、永続化層の実装となり、これらを使って[fosite.Fosite](https://github.com/ory/fosite/blob/master/fosite.go#L89)  を組み立てた後、渡したファクトリーによってハンドラーを生成しています。

[Factory](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go#L31) は以下のようなインターフェースになっています。

```go
type Factory func(config *Config, storage interface{}, strategy interface{}) interface{}
```

例えば[クライアントクレデンシャルの処理を行うハンドラー](https://github.com/ory/fosite/blob/0a48821b156f4a5dffa0f7149d30d5cf02636f37/compose/compose_oauth2.go#L50)を生成するファクトリーは以下のようになっています。

```go
func OAuth2ClientCredentialsGrantFactory(config *Config, storage interface{}, strategy interface{}) interface{} {
	return &oauth2.ClientCredentialsGrantHandler{
		HandleHelper: &oauth2.HandleHelper{
			AccessTokenStrategy: strategy.(oauth2.AccessTokenStrategy),
			AccessTokenStorage:  storage.(oauth2.AccessTokenStorage),
			AccessTokenLifespan: config.GetAccessTokenLifespan(),
		},
		ScopeStrategy:            config.GetScopeStrategy(),
		AudienceMatchingStrategy: config.GetAudienceStrategy(),
	}
}
```

### Strategy と Config について

有効化する機能は [compose.Compose](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go) にわたすファクトリーによってコントロール可能ですが、さらに細かい挙動は Strategy と Config によって設定します。厳密な基準はよくわからないのですが、内部的に実装の差し替えを行うものは Strategy、分岐や数値が変わるだけのものは [Config](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/config.go#L32) で設定できるようになっているようです。
例えば Strategy ではアクセストークンの形式（乱数 or JWT）や、IDトークンの署名アルゴリズムを選択できて、Config ではトークンの有効期間や PKCE を強制するかどうかなどの設定ができます。
可能な設定の一覧は見たところドキュメントがなさそうなので実装を見る必要がありそうです。

# ドキュメントについて

[https://github.com/ory/fosite/issues/566](https://github.com/ory/fosite/issues/566) にもあるのですが、ドキュメントだけ読んで利用できるかというとちょっと厳しいような気がします。
現時点で実際に利用する場合、少なくとも [compose.Compose](https://github.com/ory/fosite/blob/cf02af977681fd667b33f8e131891f6746d0b9da/compose/compose.go) 周りやストレージのサンプル実装などを見る必要がありそうです。
[Storage の実装で返すエラーが暗黙的に指定](https://github.com/ory/fosite/blob/a6bfb921ebc746ba7a1215e32fb42a2c0530a2bf/handler/pkce/handler.go#L143)されていたりするところがあるので、利用するエンドポイントとハンドラーのコードは一通り確認してから利用したほうが無難な気がします。