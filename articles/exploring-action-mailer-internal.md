---
title: "Action Mailerの仕組みを調べてみた"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rails, ActionMailer]
published: true
---

Railsでメーラーを実装していて、ふと不思議に思ったことはありませんか？
「メーラークラスでは**インスタンスメソッド**として定義しているのに、呼び出す時はなぜか**クラスメソッド**として使っている……。これ、どういう仕組みなんだろう？」

今回は、Action Mailerがこの「違和感」をどう解決しているのか、その裏側にある仕組みをまとめました。


### 1. 混乱のポイント：定義と呼び出しの「ズレ」

まずは、一般的なメーラーの実装を見てみましょう。

```ruby
# app/mailers/user_mailer.rb
class UserMailer < ApplicationMailer
  def welcome_email(user)
    @user = user
    mail(to: @user.email, subject: 'ご登録ありがとうございます！')
  end
end

```

ここでは `welcome_email` を**インスタンスメソッド**として定義しています。
しかし、コントローラーなどで呼び出す際は以下のようになります。

```ruby
# app/controllers/users_controller.rb
def create
  # インスタンス化（UserMailer.new）せずに、クラスメソッドとして呼んでいる！
  UserMailer.welcome_email(@user).deliver_now
end

```

通常、Rubyでは定義されていないクラスメソッドを呼べば `NoMethodError` になります。なぜAction Mailerではこれが許されるのでしょうか？

---

### 2. 正体は `method_missing` による委譲

この魔法の正体は、`ActionMailer::Base`（`ApplicationMailer` の継承元）に定義されている **`method_missing`** です。

Rubyには、呼び出されたメソッドが見つからない場合に実行される `method_missing` という特殊なメソッドがあります。Action Mailerのクラスメソッド側でこれがどう動いているかというと、以下のようなフローを辿っています。

1. `UserMailer.welcome_email` が呼ばれる（しかしクラスメソッドには定義されていない）。
2. `ActionMailer::Base` のクラスレベルの `method_missing` が発動。
3. 内部で `UserMailer` の**新しいインスタンスを生成**。
4. そのインスタンスに対して、本来の `welcome_email` メソッドを実行。
5. 最終的に `ActionMailer::MessageDelivery` オブジェクトを返す。

つまり、**「ないはずのクラスメソッドを呼ぶことで、あえてエラーを発生させ、それをフックにしてインスタンス化を自動で行っている」** のです。

---

### 3. なぜこのような設計になっているのか？

なぜわざわざ、こんなトリッキーな仕組みを採用しているのでしょうか。
主な理由は3つあると考えられます。

#### ① Viewへのデータ渡しをシンプルにするため（コントローラーの模倣）

Mailerの内部では、`@user = user` のようにインスタンス変数をセットするだけで、自動的にメールテンプレート（View）でその変数が使えるようになります。
これはRailsのコントローラーと同じ仕組みです。この「インスタンス変数をViewと共有する」という魔法を実現するためには、クラスメソッドではなく、**インスタンスのコンテキスト**で実行される必要があったのです。

#### ② 「いつ送るか」を呼び出し側で決めるため

もし `UserMailer.new.send_mail` を直接実行してしまうと、その場でメール生成処理が終わってしまいます。
Railsは `method_missing` を挟むことで、メール本体ではなく **「送信指示待ちの代理人オブジェクト（ActionMailer::MessageDelivery）」** を返しています。
これにより、呼び出し元で以下のように送信タイミングを柔軟に選べるようになります。

```ruby
# すぐ送る
UserMailer.welcome_email(@user).deliver_now

# Active Jobで後で送る
UserMailer.welcome_email(@user).deliver_later

```

#### ③ コードを「自然言語」に近づけるため（直感的なAPI）

RubyやRailsには「書くのが楽しくなる美しいコード」を良しとする文化があります。
メールを送るたびに `UserMailer.new.welcome_email...` と書くのは、実装の都合が見えすぎていて少し無粋です。
`UserMailer.welcome_email` と書けるようにすることで、 **「誰へ・どのメールを送るか」** という意図だけを記述でき、英語の文章のように自然に読めるコード（DSL）を実現しています。

---

### まとめ

今回のポイントを整理すると以下の通りです。

- **仕組みの正体：** クラスレベルの method_missing が呼び出しをキャッチし、内部でインスタンスを生成して実行している。

- **返り値の工夫：** メールそのものではなく `ActionMailer::MessageDelivery`（代理人）を返すことで、`deliver_later` などの柔軟な運用を可能にしている。

- **設計の意図：** コントローラーに近い開発体験（インスタンス変数の共有）と、直感的で読みやすいコード（DSL）を両立させるため。

仕組みを理解することで、適切なメーラーの実装を行うことができるようになるかと思います。

---

