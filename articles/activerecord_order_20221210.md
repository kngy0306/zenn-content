---
title: "ActiveRecord、orderメソッドの実装を追ってRubyにも詳しくなる"
emoji: "🍭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rails"]
published: false
---

この記事は　　アドベントカレンダーです。

Ruby / Railsともに学び始めたばかりで、日々新しい発見があります。この記事ではorderメソッドの実装を読んだ結果、orderの引数はどのように指定できるのかを知ると同時に、Rubyに関する知識も発見があったので紹介します。

# orderメソッドの使用例

DBから取得した値を並び替えることができます。

```rb

```

# ActiveRecord::QueryMethods#order

https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L409
