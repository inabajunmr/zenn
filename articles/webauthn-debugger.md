---
title: "ブラウザの拡張で各サイトが WebAuthn の API を叩くときのパラメーターを確認する"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["webauthn"]
published: true
---

WebAuthn に対応したサイトを見るときに navigator.credentials.get() や navigator.credentials.create() をどのようなパラメーターで叩いているのか気になることがありますが、これらの API をラップして実行ごとに毎回ログにパラメーターを出力してくれる Chrome 拡張を紹介します。

[SMD2/webauthn-debugger](https://github.com/SMD2/webauthn-debugger)

# 使い方

リポジトリを clone して拡張として読み込んでください。

Chrome の場合 https://developer.chrome.com/docs/extensions/mv3/getstarted/#unpacked の手順で読み込めます。

debug レベルでログが出力されるので Chrome の場合 verbose ログが出力されるようにします。

![Chrome拡張のログレベルをverboseに変更](/images/chromeextension-loglevel.png)

適当に WebAuthn に対応しているサイトで操作するとログが出力されました。

![ログ](/images/webauthn-debugger-log.png)
