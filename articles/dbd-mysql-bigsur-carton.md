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

# 試した環境

```
$ sw_vers
ProductName:    macOS
ProductVersion: 11.4
BuildVersion:   20F71
$ carton --version
carton v1.0.34
$ perl --version
This is perl 5, version 30, subversion 2 (v5.30.2) built for darwin-thread-multi-2level
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

# 追記（WARNING: xxx/perl5.30.2 is loading libcrypto in an unsafe way が出た場合）

参考: https://github-wiki-see.page/m/kyzn/PRC/wiki/Development-Instructions-(macOS-Apple-Silicon)

```
$ brew --prefix openssl
/usr/local/opt/openssl@3
```

openssl のパスを確認し、以下のように symlink を貼ったらインストールできました。

```
$ sudo ln -s /usr/local/opt/openssl@3/lib/libssl.dylib /usr/local/lib/libssl.dylib
$ sudo ln -s /usr/local/opt/openssl@3/lib/libcrypto.3.dylib /usr/local/lib/libcrypto.dylib
```
