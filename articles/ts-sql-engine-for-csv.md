---
title: "TypeScript で CSV に SQL を発行するやつを作る"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["sql", "database", "typescript"]
published: true
---

# 作ったやつ

こんな感じで動くやつを作りました。（作っています）

![](/images/swrql.gif)

# この記事で言いたいこと

https://link.springer.com/book/10.1007/978-3-030-33836-7

[Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) は面白いのでおすすめです。

# まとめ

* 便利なツールを作ったのでみんな使ってくれ的な記事ではないです
* [Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) が面白かったので何か作りたくなった
* TypeScript で CSV に SQL を発行するやつを作っている
  * GROUP BY したり JOIN したりできる
  * 作ってるやつ
    * https://github.com/inabajunmr/swrql
* 実践で使えるようなものではないのでそういうツールが欲しい場合は以下のようなものを使うと良いと思います
  * http://harelba.github.io/q/
  * https://github.com/mithrandie/csvq
* [Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) は面白いのでおすすめです
  * そんなに前提知識なくても読めます（なんとなく SELECT のクエリが書ける、で十分だと思う）
    * ボリュームは結構あって、英語

# TypeScript で CSV に SQL を発行するやつを作る

## [Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) が面白かった

こちらの記事を読んで楽しそうだったのでとりあえず本を読みました。

https://tarovel4842.hatenablog.com/entry/2021/12/20/084413

内容については記事を見ていただくと良いと思います。なんとなく SQL 書けます、みたいなレベルでも十分読める内容でした。一通り読んだ結果、わからないことが増えました。(なんで商用のデータベースだとあれができるんだ？みたいな疑問がいっぱいでてくる)

## 読み終わったのでなんか作ってみたくなった

RDBMS をスクラッチで作ろう、的な書籍なんですが、一通り読んでなんか作ってみたくなったので TypeScript で CSV に SQL を発行するやつを作ることにしました。

https://github.com/inabajunmr/swrql

## なんとなく動くようになった

TypeScript なのはブラウザで動いたら楽しい気がしたからです。
以下で実際に動かすことができます。

https://inabajunmr.github.io/swrql/pages/public/

こんな感じで動きます。

![](/images/swrql.gif)

## 書いたコードについて

* 実行計画はない
  * 状況に関わらず常に同じ流れでクエリが実行される
    * [書籍](https://link.springer.com/book/10.1007/978-3-030-33836-7)だと最適化の基準が基本的にブロックへのアクセス数なのでインメモリだとこの辺の考え方が変わってくる気がするけど、よくわかってない
* インメモリ
* マージジョインとか B-Tree とか実装してない
  * 直積に対して条件が一致するレコードを抽出するだけで JOIN している
  * [書籍](https://link.springer.com/book/10.1007/978-3-030-33836-7)でいう 8 章のパイプライン処理を安直に実装していった
* 複雑な WHERE 句のパースがいまいちわからなかった（書籍だと完全一致を AND でつなぐ仕様）
  * ので[操車場アルゴリズム](https://github.com/inabajunmr/swrql/blob/main/swrql/src/sql/parser.ts#L218)で逆ポーランド記法にして、レコードに対してマッチするか毎回チェックするようにした
    * `name='inaba' AND age=32` だったら、`name,inaba,=,age,32,AND` になる
      * クエリに合わせて実行計画を最適化するような場合、めちゃくちゃ相性が悪いような気もする
        * そもそも複雑な条件式で実行計画をどうすればよいのか、よくわかってない
* 外部結合は軸になるテーブルに対してマッチするレコードを全探索している
  * めちゃくちゃ効率が悪い
    * 結合条件が単純なら結合のキーで先にソートすればよいが、複雑な場合にどうするといい感じなのかよくわからない
* いろいろ動かない
  * 副問合せとか
  * SQL、何が動けば自分の中で完成とみなせるのかよくわからない
  * パーサーの異常系の処理がめちゃくちゃ甘いのでブラウザで触ってるとたまにハングする

# まとめ

* 便利なツールを作ったのでみんな使ってくれ的な記事ではないです
* [Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) が面白かったので何か作りたくなった
* TypeScript で CSV に SQL を発行するやつを作っている
  * GROUP BY したり JOIN したりできる
  * 作ってるやつ
    * https://github.com/inabajunmr/swrql
* 実践で使えるようなものではないのでそういうツールが欲しい場合は以下のようなものを使うと良いと思います
  * http://harelba.github.io/q/
  * https://github.com/mithrandie/csvq
* [Database Design and Implementation(Second Edition)](https://link.springer.com/book/10.1007/978-3-030-33836-7) は面白いのでおすすめです
  * そんなに前提知識なくても読めます（なんとなく SELECT のクエリが書ける、で十分だと思う）
    * ボリュームは結構あって、英語

https://link.springer.com/book/10.1007/978-3-030-33836-7

次は CSV ではなく書籍のように永続化部分から RDBMS を実装してみたいと思いました。
