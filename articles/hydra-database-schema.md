---
title: "Hydra のデータベースを構築して定義を見てみる"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [oauth, oidc, hydra, database]
published: true
---

# Hydra

https://github.com/ory/hydra

以下 README.md の引用です。

> ORY Hydra is a hardened, OpenID Certified OAuth 2.0 Server and OpenID Connect Provider optimized for low-latency, high throughput, and low resource consumption. ORY Hydra is not an identity provider (user sign up, user login, password reset flow), but connects to your existing identity provider through a login and consent app. Implementing the login and consent app in a different language is easy, and exemplary consent apps (Node) and SDKs for all common languages are provided.

本記事では Hydra のデータベース定義を見てみます。

## Hydra のバージョン

```
hydra version
Version:    v1.10.3
Git Hash:   ea931581eb54ab5dc142ea1f81357f25b8e4156a
Build Time: 2021-07-14T14:42:23Z
```

## データベースの構築

構築したデータベースの定義を見るため、データーベースを構築します。
Hydra では PostgreSQL、MySQL、SQLite をサポートしています。（ベータ版として CockroachDB もサポートしています）
ここでは PostgreSQL を利用します。

Docker で PostgreSQL を立てて Hydra 用のテーブルを作成します。

```
docker run -p 5432:5432 --name hydra-postgres -e POSTGRES_PASSWORD=pass -d postgres
psql -h localhost -p 5432 -U postgres
create database hydra;
```

Hydra の CLI を[インストール](https://www.ory.sh/hydra/docs/install#macos)します。

```
brew tap ory/hydra
brew install ory/hydra/hydra
```

CLI でテーブルを作成します。

```
hydra migrate sql postgres://postgres:pass@localhost:5432/hydra
```

テーブルが作成されました。

```
psql -h localhost -p 5432 -U postgres -d hydra
Password for user postgres: 
psql (13.3)
Type "help" for help.

hydra=# \dt
                             List of relations
 Schema |                      Name                      | Type  |  Owner   
--------+------------------------------------------------+-------+----------
 public | hydra_client                                   | table | postgres
 public | hydra_jwk                                      | table | postgres
 public | hydra_oauth2_access                            | table | postgres
 public | hydra_oauth2_authentication_request            | table | postgres
 public | hydra_oauth2_authentication_request_handled    | table | postgres
 public | hydra_oauth2_authentication_session            | table | postgres
 public | hydra_oauth2_code                              | table | postgres
 public | hydra_oauth2_consent_request                   | table | postgres
 public | hydra_oauth2_consent_request_handled           | table | postgres
 public | hydra_oauth2_jti_blacklist                     | table | postgres
 public | hydra_oauth2_logout_request                    | table | postgres
 public | hydra_oauth2_obfuscated_authentication_session | table | postgres
 public | hydra_oauth2_oidc                              | table | postgres
 public | hydra_oauth2_pkce                              | table | postgres
 public | hydra_oauth2_refresh                           | table | postgres
 public | schema_migration                               | table | postgres
(16 rows)
```

SchemaSpy でスキーマを出力します。

```
docker run -v "$PWD/schema:/output" --net="host" schemaspy/schemaspy:snapshot -t pgsql -host localhost:5432 -db hydra -u postgres -p pass
```


## テーブル定義

画像はすべて SchemaSpy によって出力されたものです。

:::message
コメントはテーブルやカラム名から適当に想像したり実際に動かしたときの挙動からなんとなく推測しているだけで、実装は確認してません。
:::

### hydra_client

作成したクライアントの設定情報？

![](/images/hydra_client.1degree.png)

### hydra_jwk

Hydra 自体が提供する JWK？

![](/images/hydra_jwk.1degree.png)

### hydra_oauth2_access

アクセストークン？（トークン発行したり Revoke したりするとレコード数が増減する）

![](/images/hydra_oauth2_access.1degree.png)

### hydra_oauth2_authentication_request

hydra への認可リクエスト？（ログイン処理で hydra に通知される認証結果の検証に使ってる気がする）

![](/images/hydra_oauth2_authentication_req_15aeee6c.1degree.png)

### hydra_oauth2_authentication_request_handled

認可リクエストに対するログイン処理の結果？

![](/images/hydra_oauth2_authentication_req_15aeee6c.1degree.png)

### hydra_oauth2_authentication_session

![](/images/hydra_oauth2_authentication_session.1degree.png)

### hydra_oauth2_code

認可コード？

![](/images/hydra_oauth2_code.1degree.png)

### hydra_oauth2_consent_request

hydra から同意画面に遷移する際のリクエストを管理？（同意処理で hydra に通知される同意結果の検証に使ってる気がする）

![](/images/hydra_oauth2_consent_request.1degree.png)

### hydra_oauth2_consent_request_handled

同意リクエストに対する処理の結果？

![](/images/hydra_oauth2_consent_request_handled.1degree.png)

### hydra_oauth2_jti_blacklist

金された JWT を管理？

![](/images/hydra_oauth2_jti_blacklist.1degree.png)

### hydra_oauth2_logout_request

![](/images/hydra_oauth2_logout_request.1degree.png)

### hydra_oauth2_obfuscated_authentication_session

![](/images/hydra_oauth2_obfuscated_authent_1c0b8da3.1degree.png)

### hydra_oauth2_oidc

![](/images/hydra_oauth2_oidc.1degree.png)

### hydra_oauth2_pkce

[PKCE](https://datatracker.ietf.org/doc/html/rfc7636) 関連の何か？（PKCE ありの認可リクエストをするとレコードが登録される）

![](/images/hydra_oauth2_pkce.1degree.png)

### hydra_oauth2_refresh

リフレッシュトークン？

![](/images/hydra_oauth2_refresh.1degree.png)

### schema_migration

マイグレーションの管理用？

![](/images/schema_migration.1degree.png)

## 感想

* 大雑把に以下を管理するためのテーブルがあるっぽい
  * 認可リクエストからの一連のトランザクション
  * 発行したトークン類
  * JWT 関連
  * クライアント
