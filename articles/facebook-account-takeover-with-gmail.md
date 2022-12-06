---
title: "Facebook のフェデレーション先からの認可レスポンスを奪う脆弱性について"
emoji: "💋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["openidconnect", "oauth2", "security"]
published: true
---

[Digital Identity技術勉強会 #iddance Advent Calendar 2022 7日目](https://qiita.com/advent-calendar/2022/iddance)の記事です。

[Multiple bugs chained to takeover Facebook Accounts which uses Gmail](https://ysamm.com/?p=763) で説明されている Facebook がフェデレーションしている Google の認可レスポンスを盗んで他人のアカウントを乗っ取る攻撃についてです。
予測で書いてる部分などあるので詳細はオリジナルの記事を参照してください。

https://ysamm.com/?p=763

## 何の話か

* Facebook は色々な背景で攻撃用のページから Facebook がフェデレーションしている Google の認可レスポンスを盗める脆弱性があった
* これによって盗んだ認可レスポンスを使って、被害者の Facebook アカウントに侵入できた
* 報告によって Facebook の脆弱性はすでに修正済み

## 背景

攻撃を成立させるための背景についてです。

### チェックポイント

* Facebook のアカウントにはアカウントの悪用などで「チェックポイント」という状態になる
* この状態になるとユーザーは Google の Captcha が要求されるページに飛ばされることがある
    * このページはクエリパラメーターで Captcha 完了後のページにリダイレクトするページを指定できる
* このページの Captcha は `www.fbsbx.com` により配信されていて、`www.facebook.co`m から iframe でロードされる
    * `www.fbsbx.com` にはロード元の URL がクエリパラメーターに指定されている

例えばチェックポイント状態のユーザーが `https://www.facebook.com/eg_oauth_callback_endpoint?code=code` にアクセスした場合、

1. `https://www.facebook.com/checkpoint/CHECKPOINT_ID/?next=https%3A%2F%2Fwww.facebook.com%2Feg_oauth_callback_endpoint%3Fcode%3Dcode` にリダイレクト
2. このページには `https://www.fbsbx.com/captcha/recaptcha/iframe/?referer=https://www.facebook.com/checkpoint/CHECKPOINT_ID/?next=https%3A%2F%2Fwww.facebook.com%2Feg_oauth_callback_endpoint%3Fcode%3Dcode` が iframe でロード

という状態になります。

### www.fbsbx.com の XSS

おそらくミニアプリ的なものの開発のためだと思うのですが、Facebook には[開発者ツールから任意の HTML をアップロードする機能](https://developers.facebook.com/blog/post/2019/08/22/introducing-playable-preview-tool-to-validate-playable-assets/?locale=ja_JP)があります。
この HTML は `https://www.fbsbx.com/developer/tools/playable-preview/preview-asset/?handle_str=STR` のように `www.fbsbx.com` から配信できます。

### CSRF によるログインとログアウト


これは解説がなかったので実際に可能なのかどうかなどわかりませんでしたが、攻撃の手順を見た感じ CSRF でユーザーをログアウトさせたり、あるユーザーにログインさせたりできることが前提になっているようです。

## アカウントの乗っ取り

上記の背景にあわせてアカウントの乗っ取りが以下のような手順で行われます。

### 認可レスポンスの取得まで

1. 被害者が `www.fbsbx.com` から配信されている攻撃用のページにアクセス
2. （おそらく）攻撃用のページで CSRF でユーザーをログアウト & CSRF でチェックポイント状態のユーザーにログインさせる
3. 攻撃用のページで `https://accounts.google.com/o/oauth2/auth?redirect_uri=https://www.facebook.com/oauth2/redirect/&response_type=permission%20code%20id_token&scope=email%20profile%20openid&client_id=15057814354-80cg059cn49j6kmhhkjam4b00on1gb2n.apps.googleusercontent.com` を window.open で開く
4. （おそらく）Google のログイン画面や同意画面が開くが、同意済みの場合は即時に `www.facebook.com` のチェックポイントのページ（Captcha が表示されるやつ）にリダイレクトする
5. 3 で開いた Window オブジェクト経由で、チェックポイントのページに埋め込まれた iframe の location.href を取得できる（オリジンが同じため）
6. これによって Google からの認可レスポンスが取得できる


![ここまでのシーケンス](/images/facebook-account-takeover-with-gmail1.png)

<!-- ![](https://www.plantuml.com/plantuml/png/bL9FIm915B_FfnZs47LeFu2748AwroSOrrbTqXtPdMsxxip5Go6960GUYZ2p_qHY5FbX7ceVexSx8HeM9RWClD_x_NdlvKF90XbLAuH5KZ17UljCSYfyOe5kWxu2rGKriEZw1hNYEBRTGbWui1rHfP3SJLglawQUjdgWJq6_WHfQ9E0o2dpY2wHXJY32C4StTtUs47y9kfl11jct2VeMr0EeyrH4r-aA1Ps0GfW6TG-w0-e2z1EM9iCw9AE5zkJQj1kCHKWf454DJNf-KSIsg77V8XH_ypdI6Dj2b2gSvkX7_sZSG9b-AgPwe5vGAMin6MQ0cwpKGHbyBELqcSCBGknbWy6yh2QTCaj7D9iC9CCvxkpvvTXp0rRbT7MUPCruNwArP0qplOLDe0yJB4seZrCEVPRLHcP-FyVxvfFxHPB-_tG0_xHeraMdpuVJrYEU2Aq8XqbF-7Ovu2ucQ8uRxV7x2Vbr9tcqXqVy2G00) -->

### 取得した情報を使って被害者のアカウントでパスワードリカバリ

1. 認可レスポンスに含まれる ID トークンから被害者のメールアドレスを取得
2. 被害者のメールアドレスでパスワードリカバリのフローをすすめる
3. Facebook のパスワードリカバリは Google のアカウント連携で行えるので、選択すると Google に認可リクエストが送られる
4. この認可レスポンスに含まれる state と、「認可レスポンスの取得まで」で取得した認可コードを組み合わせて Facebook に認可レスポンスとしてアクセスすると、被害者のパスワードリカバリを行える

![ここまでのシーケンス](/images/facebook-account-takeover-with-gmail2.png)

<!-- ![](https://www.plantuml.com/plantuml/png/VP5FIiD05CRtFSLSm0kua2v4yHXZ32BM32GJrvtt5595Khke2r4nYrWRQHP1KH0zp4SqtiAP6aLG2bbyl3_VBz-RRgHbgEEzNaV6MabFx-nBxGiLTC1Zy2qe1ps8fHMn9Zr_KwbTiImsLWKu0J-3Id05SmmQrEcuVZc3iTyL-DfrkG1bu07u3lG4SmC-TO9BrtNrAQW03wteXBwEmx_OTQMsUxw5jiAw_5u3ZJxCuzlHfGFtJJCzgTDoNYz1apaiMHJbtmMOcR1tqOusQRAYdbLD_z1Msscgsl-dqVlz0SnhFKn0ASmvJFurqqRc2t1bnNUwG5bJwOKgYYpxnvao5uXoE-pmYm_LzOCj-s2V) -->

## 気になったこと

### 攻撃者による認可コードの注入は nonce で防げるのでは？

被害者が最初に発行した認可リクエスト（および認可コード）の nonce と、攻撃者のパスワードリカバリによる認可リクエストの nonce が別なのでそこで弾ける気がしますが、特に言及はありませんでした。

## 感想

https://ysamm.com/?p=763

についてでした。このケースはいろいろ条件が必要ですが、認可レスポンスはセンシティブな情報でクエリパラメーターが漏れる余地はいろんなところにあるよ、という意味では色々考えないといけないことはあるなと思いました。

もともとこちらの記事を見ていて知ったので、こちらについてもどこかでまとめられればと思います。
https://labs.detectify.com/2022/07/06/account-hijacking-using-dirty-dancing-in-sign-in-oauth-flows/