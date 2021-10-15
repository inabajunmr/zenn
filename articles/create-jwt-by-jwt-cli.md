---
title: "ターミナルから jwt-cli で JWT を作成したりデコードしたりする"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jwt"]
published: true
---


# [mike-engel/jwt-cli](https://github.com/mike-engel/jwt-cli)

Rust 製の JWT を作ったりデコードしたりするやつです。

## 記事を書いたときのバージョン

```
$ jwt --version
jwt 4.0.0
```

## インストール

この辺を参照ください。

https://github.com/mike-engel/jwt-cli#homebrew

## 使い方

### デコード

```
$ jwt decode -h
jwt-decode 
Decode a JWT

USAGE:
    jwt decode [FLAGS] [OPTIONS] <jwt>

FLAGS:
    -h, --help       Prints help information
        --iso8601    display unix timestamps as ISO 8601 dates
    -j, --json       render decoded JWT as JSON
    -V, --version    Prints version information

OPTIONS:
    -A, --alg <algorithm>    the algorithm to use for signing the JWT [default: HS256]  [possible values: HS256, HS384,
                             HS512, RS256, RS384, RS512, ES256, ES384]
    -S, --secret <secret>    the secret to validate the JWT with. Can be prefixed with @ to read from a binary file
                             [default: ]

ARGS:
    <jwt>    the jwt to decode
```

### エンコード

```
$ jwt encode -h
jwt-encode 
Encode new JWTs

USAGE:
    jwt encode [FLAGS] [OPTIONS] --secret <secret> [--] [json]

FLAGS:
    -h, --help       Prints help information
        --no-iat     prevent an iat claim from being automatically added
    -V, --version    Prints version information

OPTIONS:
    -A, --alg <algorithm>         the algorithm to use for signing the JWT [default: HS256]  [possible values: HS256,
                                  HS384, HS512, RS256, RS384, RS512, ES256, ES384]
    -a, --aud <audience>          the audience of the token
    -e, --exp <expires>           the time the token should expire, in seconds or systemd.time string [default: +30 min]
    -i, --iss <issuer>            the issuer of the token
        --jti <jwt_id>            the jwt id of the token
    -k, --kid <kid>               the kid to place in the header
    -n, --nbf <not_before>        the time the JWT should become valid, in seconds or systemd.time string
    -P, --payload <payload>...    a key=value pair to add to the payload
    -S, --secret <secret>         the secret to sign the JWT with. Can be prefixed with @ to read from a binary file
    -s, --sub <subject>           the subject of the token
    -t, --typ <type>              the type of token being encoded [possible values: JWT]

ARGS:
    <json>    the json payload to encode
```

## やってみる

### エンコード

```
$ jwt encode -P 'hello=jwt' --secret "secret"
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJoZWxsbyI6Imp3dCIsImlhdCI6MTYzNDI5MTE3MX0.5JXMhypffjaZ6tYzNu_XFi3lFxHCXSZGKX7jzVeEAPE
```

### デコード

```
$ jwt decode eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJoZWxsbyI6Imp3dCIsImlhdCI6MTYzNDI5MTE3MX0.5JXMhypffjaZ6tYzNu_XFi3lFxHCXSZGKX7jzVeEAPE

Token header
------------
{
  "typ": "JWT",
  "alg": "HS256"
}

Token claims
------------
{
  "hello": "jwt",
  "iat": 1634291171
}
```
