---
title: "例外はハンドリングする必要がある"
emoji: "💋"
type: "idea"
topics: ["java", "programming", "exception"]
published: true
---

例外をハンドリングしなくてもよい世界観では以下はあてはまらない。

# まとめ

- 例外はハンドリングする必要がある
- 最低限、その例外がスローされたときに何が起きるのか把握する必要がある
- 例外が実際にスローされたと仮定して、問題がおきない、もしくは起きた問題に対して適切に対処できることが担保されている必要がある
- 通知すればOKではない。通知を確実に受け取り処理できることが担保できて初めてOK
- 絶対にスローされない例外に対して何かする必要はないが、おそらくスローされなそう、もしくはスローされるかどうかよくわからない例外は原則ハンドリングしなくてはいけない
- そもそも例外として何がスローされるのかを理解せずにメソッドを呼ぶべきではない

# ここでいうハンドリングとは

例外が実際にスローされたときに、後続の処理がいい感じになるようにすること。
例えば以下のメソッドを呼び出す場合を考える。

```java
	/**
	 * aaa する
	 *
	 * @throws XxxException xxx の場合
	 * @throws YyyException yyy の場合
	 * @throws ZzzException zzz の場合
	 */
	public void excute() {
	}

```

この場合 execute の呼び出し元は、XxxException、YyyException、ZzzException のどれがスローされても問題がない、もしくは問題を最低限に収められるようなコードを書く必要がある。

そもそも絶対にスローされない呼び出し方をする、でも問題ない。
例えば以下の場合、method1 が method2 に渡している value は null にならないため、method2 が NullPointerException をスローする可能性を考慮する必要はない。

```java
	public static void method1(String value) {
		if(value != null) {
			method2(value); // method2 は絶対に NullPointerException をスローしない
		}
	}
	
	/**
	 * aaa する
	 *
	 * @throws NullPointerException value が null の場合
	 */
	public static void method2(String value) {
	}

```

ただし**おそらく**スローされない、という理由でその例外を考慮しない場合、後に問題が起きることがあるのでそのような例外は原則考慮すべきである。

ネットワークがつながらなければ外部サービスへのリクエストは失敗するし、ディスクフルであればファイルへの書き込みは失敗するし、入力は想定通り JSON でない場合がある。
絶対に外部サービスへの接続が成功する、絶対にファイルへの書き込みが失敗しない、絶対に入力が JSON である、という前提が担保されるのであれば、その例外は考慮しなくてもよい。

しかしそれはもうおそらくスローされない例外ではなく、絶対にスローされない例外である。

# いい感じの後続処理とは

まずスローされた後、何が起きるのかを把握しながらコードを書く必要がある。何が起きるのか把握していない状態は何が起きるのかわからないので、問題がある。

全てのメソッド呼び出しがあらゆる例外を直接 catch する必要はない。
しかし、呼び出し元で直接例外を catch しない場合、呼び出し元の呼び出し元が例外を認識してコードを書ける必要がある。
そのため個人的には、全てのメソッドが自身がスローすべき例外を網羅的に Javadoc に書いておくべきだと思っている。
[https://dev.classmethod.jp/articles/javadoc-throws/](https://dev.classmethod.jp/articles/javadoc-throws/)

ただし、グローバルなハンドラでまとめて処理する例外が決まっているような場合、ハンドラに任せても問題ない。
そのケースでも、例外がスローされた場合に何が起きるのか（ハンドラによって例外が処理される）を把握している必要がある。

次に、いざ例外がスローされた後大きな問題が起きないようにすることを考える。
大きな問題の定義は様々だが、例えば以下のような大きな問題がある。

- 顧客が購入していない商品の請求書が送付される
- 情報が漏洩する
- 応募者全員に内定通知が飛ぶ

例えば、請求書送付処理を行った後で商品の引当に失敗した場合、請求書送付処理をロールバックする必要があるが、そのハンドリングが漏れている、といったケースである。（先に引当をしなさい問題については、良い例が思いつかないので無視する）

しかしアプリケーションの特性上どうしても例外処理だけで問題を防ぐことができないケースがある。
そのようなケースでは、問題が起きてしまった後どうするか？を把握し、決める必要がある。

## 問題が起きてしまった後どうするか？

問題が起きたら、大抵の場合リカバーする必要がある。
リカバーの内容はデータの補填であったり、通知であったり、謝罪であったりいろいろあるが、とにかくリカバーのフローが最後まで成立する必要がある。

例えば商品購入メールの送信後、配送処理のバッチで商品の引当に失敗した、というケースを考える。
ここではリカバーの方法は「電話での謝罪」とする。
誰が電話をするのか？など決めるべきことは無数にあるが、まず最初に気にする必要があるのは電話しなければいけない、という事実にどうやって気がつくのか？である。

ここで、以下のようなコードを書いたと仮定する。

```java
public void allocateItem(String itemId) {
    try {
			allocationService.allocate(itemId);
		} catch (AllocationFailureException e) {
			log.error("allocation failed. item:" + itemId, e);
		}
}
```

このコードの例外処理は十分だろうか。エラーログを出しているのでよさそうな気もするし、ログのみで例外が潰されているのでダメそうな気もする。十分かどうかはこのコードだけではわからない。

このコードに対して問題ないと判断するためには、少なくともエラーログを誰かが確実にキャッチする必要がある。このようなコードを書く場合、エラーログが確実に確認されるような運用になっているのかを理解している必要がある。確認されない可能性がある場合、通知方法や運用を再検討する必要がある。

大切なのはアラートを飛ばすことではなく、飛んだアラートが確実に処理されることである。

# 絶対にスローされないチェック例外について

非チェック例外であればなかったことにすればよいが、チェック例外はどこかで catch する必要がある。catch して何もしないコードを見るとギョッとするので、コメントに何か書いてあるか、AssertionError を再度スローするようになっていると個人的には嬉しい。