---
title: "セキュリティ上重要なクッキー属性を読んでみる"
emoji: "🍪"
type: "tech"
topics:
  - "rails"
published: true
published_at: "2022-11-27 15:23"
---

[体系的に学ぶ 安全なWebアプリケーションの作り方](https://www.sbcr.jp/product/4797393163/)という書籍を読んでいて、クッキーの**属性**についてこれまで意識したことがなかったな、と思い、セキュリティ上重要なクッキーの属性についてまとめました。

また、他人のクッキーからセッションIDを盗み取り、他人になりすましてしまう**セッションハイジャック**という攻撃をRailsで作成されたアプリケーション上で試してみました。

# セキュリティ上重要なクッキー属性

書籍では以下をセキュリティ上重要な属性としてあげています。

- Domain属性
- Secure属性
- HttpOnly属性
- SameSite属性

## Domain属性

Domain属性では、クッキーを受信することができるホストを指定します。デフォルトではクッキーを設定したサーバにのみ送信されます。

## Secure属性

Secure属性をつけたクッキーはHTTPS通信であるときのみサーバに送信されます。
通信が暗号化されていないHTTP通信を盗聴され、クッキー情報を取得されてしまうことを防ぐことができます。

## HttpOnly属性

HttpOnly属性をつけたクッキーは、JavaScriptからアクセスできなくなります。これによりXSS攻撃などの実行を難しくすることができます。

## SameSite属性

SameSite属性にて、他のサイトへ遷移する際にクッキーも一緒に送るかどうかを指定することができます。

SameSite属性は3つの値を指定できます。

- Strict
- Lax
- None

### Strict

クッキーは、クッキーを発行したサイトにだけ送信されます。

### Lax

外部サイトからリンクをたどり、クッキーを発行したサイトに遷移した場合でもクッキーが送信されます。ただし外部サイトからのアクセスがGETリクエストのときだけです。

Google Chrome 80以降、SameSite属性を指定しない場合は`Lax`がデフォルトになりました。

### None

外部からのどんなリクエストに対しても、クッキーが送信されます。

Google Chrome 80以降、SameSite属性を`None`にする場合はSecure属性を付与することが必須になっています。

## ブラウザでクッキー属性を確認する

クッキーの属性はデベロッパーツールなどから確認することができます。以下のスクリーンショットはGoogle ChromeでZennのクッキーを確認した画面です。

![](https://storage.googleapis.com/zenn-user-upload/008cd2639154-20221127.png)

# セッションハイジャックを試す

なんらかの原因でクッキーに保存されたセッションIDを盗み取られた場合、セッションハイジャックなどの攻撃に利用されてしまう可能性があります。
以下では、攻撃者であるユーザが自身のセッションを、別ユーザのセッションに書き換えることで別ユーザとしてアクセスできてしまう例を試しました。

左に被害者の画面、右に攻撃者の画面を表示しています。またそれぞれの画面の下部にはブラウザに保存されているセッションIDを表示させています。

![](https://storage.googleapis.com/zenn-user-upload/ccdc0de2306f-20221124.png)

このとき、攻撃者が何らかの方法で被害者のセッションIDを入手し、セッションIDを書き換えたとします。

![](https://storage.googleapis.com/zenn-user-upload/387d6ec6b56f-20221125.png)

その後、攻撃者の画面をリロードすると被害者のアカウントでログインしていることにできてしまいます。

![](https://storage.googleapis.com/zenn-user-upload/36afb2beebc2-20221125.png)

# 参考文献

https://www.sbcr.jp/product/4797393163/
https://developer.mozilla.org/ja/docs/Web/HTTP/Cookies
https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Set-Cookie/SameSite
https://qiita.com/ahera/items/0c8276da6b0bed2b580c