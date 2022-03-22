---
title: "convert-junit4-to-junit5 で JUnit 4 のプロジェクトを JUnit 5 に一括置換する"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['junit', 'junit4', 'junit5']
published: true
---

置換が結構めんどくさかったので [junit-pioneer/convert-junit4-to-junit5](https://github.com/junit-pioneer/convert-junit4-to-junit5) を試してみました。

# 検証環境

```
$ java --version
openjdk 11.0.2 2019-01-15
OpenJDK Runtime Environment 18.9 (build 11.0.2+9)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.2+9, mixed mode)
$ sw_vers
ProductName:	macOS
ProductVersion:	11.4
BuildVersion:	20F71
```

# やってみた
Gradle でビルドします。
```
$ ./gradlew build
```

build/libs にビルドされた jar がいるので起動します。

```
$ cd build/libs
$ java -jar convert-junit4-to-junit5-fat.jar -w 配下にテストクラスが存在するディレクトリ
```


[junit-pioneer/convert-junit4-to-junit5](https://github.com/junit-pioneer/convert-junit4-to-junit5#running-the-update-from-the-command-line)

# 結果
spring-security-oauth2 に試した結果以下のような差分になりました。
https://github.com/inabajunmr/spring-security-oauth/compare/main...inabajunmr:apply-convert-junit4-to-junit5

単純なアノテーションクラスの置き換えだけではなく、expected を assertThrows に書き換えたりもしてくれています。
Rule とかは駄目なので自力で書き換える必要がありますが、それでもだいぶ楽なのではないかと思いました。
