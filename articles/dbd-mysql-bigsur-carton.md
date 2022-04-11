---
title: "M1 Mac で DBD-mysql を carton からインストールする"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['m1mac', 'perl', 'carton', 'mysql', dbd']
published: true
---

ほとんど以下の記事で紹介いただいている通りです。
https://blog.mitsuto.com/macos-mojave-perl-dbd-mysql

上記記事の通り以下のようにパスを修正して実行すると、

```
PATH="$(brew --prefix mysql-client)/bin:$PATH"
export LIBRARY_PATH=$(brew --prefix openssl)/lib:$LIBRARY_PATH
```

zstd についてエラーがでました。

```
Can't link/include C library 'zstd', aborting.
```

インストールしてこちらも同様にパスを修正したらインストールできました。

```
brew install zstd
export LIBRARY_PATH=$(brew --prefix zstd)/lib:$LIBRARY_PATH
```

# 一連の流れ

インストール時に実行した一連のコマンドは以下のようになりました。

```
PATH="$(brew --prefix mysql-client)/bin:$PATH"
export LIBRARY_PATH=$(brew --prefix openssl)/lib:$LIBRARY_PATH
$ brew install mysql
$ mysql
mysql > create user 'xxx'@'localhost' identified  by 's3kr1t';
mysql > grant all privileges on test.* to 'xxx'@'localhost';
$ brew install zstd
$ export LIBRARY_PATH=$(brew --prefix zstd)/lib:$LIBRARY_PATH
```

xxx は実行環境のユーザー名になると思います。
