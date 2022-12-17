---
title: "ActiveRecord::QueryMethods#orderの実装を追ってRubyにも詳しくなる"
emoji: "🍭"
type: "tech"
topics:
  - "rails"
published: true
published_at: "2022-12-17 23:46"
---

株式会社iCAREのこなゆうです！

今年の10月からRuby、Railsを学び始めたため、業務を行う中で日々新しい発見があります。
この記事ではorderメソッドの実装を読んで、引数にどのような値を記述できるのかを調べた記録を書きました。
コードを読んでいる途中で普段あまり見かけないRubyのメソッドにも出会ったのでいくつかまとめました。

# orderメソッドの使用例

DBから取得したレコードを、指定したカラムで並び替えることができます。

```rb
Book.order(:title)
#=> SELECT "books".* FROM "books" ORDER BY "books"."title" ASC

Book.order(title: :DESC)
#=> SELECT "books".* FROM "books" ORDER BY "books"."title" DESC
```

引数にカラム名をシンボルで指定した場合、デフォルトで昇順（ASC）に並びます。ハッシュを用いて昇順、降順を指定できます。

# orderメソッドの引数はどのように記述できるか

## def order

ソースコード：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L409

ここが最初に呼び出される箇所です。

```rb
def order(*args)
  check_if_method_has_arguments!(__callee__, args) do
    sanitize_order_arguments(args)
  end
  spawn.order!(*args)
end
```

`check_if_method_has_arguments!`は引数のサニタイズ処理を行っていそうだな、というところで
次に呼び出されている`order!`にいきます。

:::details spawnについて

spawnは以下に定義されています。
https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/spawn_methods.rb#L10

```rb
def spawn # :nodoc:
  already_in_scope?(klass.scope_registry) ? klass.all : clone
end
```

spawn内で呼び出されている`already_in_scope?(klass.scope_registry)`がtrueになる条件はわかりませんでした。
以下の呼び出しはすべて`already_in_scope?(klass.scope_registry)`がflaseでした。

```rb
Book.order(:title)
Book.where(id: 1).order(:title)
Book.all.order(:title)
```

```rb
# binding.irbで確認
irb(<Book::ActiveRecord_Relation:0x0000ffffa9f8fd00>):001:0> already_in_scope?(klass.scope_registry)
=> false
```

またどの呼び出しにおいても`klass.all`と`clone`は同じ値でした。

```rb
irb(<Book::ActiveRecord_Relation:0x0000ffffa9f8fd00>):002:0> klass.all
  Book Load (0.7ms)  SELECT "books".* FROM "books"

irb(<Book::ActiveRecord_Relation:0x0000ffffa9f8fd00>):003:0> clone
  Book Load (0.7ms)  SELECT "books".* FROM "books"
```

:::

## def order!

ソースコード：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L417

```rb
# Same as #order but operates on relation in-place instead of copying.
def order!(*args) # :nodoc:
  preprocess_order_args(args) unless args.empty?
  self.order_values |= args
  self
end
```

次にここで呼び出されているpreprocess_order_argsにいきます。

## def preprocess_order_args

ソースコード：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L1567

```rb
def preprocess_order_args(order_args)
  @klass.disallow_raw_sql!(
    order_args.flat_map { |a| a.is_a?(Hash) ? a.keys : a },
    permit: connection.column_name_with_order_matcher
  )

  validate_order_args(order_args)

  references = column_references(order_args)
  self.references_values |= references unless references.empty?

  # if a symbol is given we prepend the quoted table name
  order_args.map! do |arg|
    case arg
    when Symbol
      order_column(arg.to_s).asc
    when Hash
      arg.map { |field, dir|
        case field
        when Arel::Nodes::SqlLiteral
          field.public_send(dir.downcase)
        else
          order_column(field.to_s).public_send(dir.downcase)
        end
      }
    else
      arg
    end
  end.flatten!
end
```

ここまできて、orderメソッドの引数にはどのような値を渡せるかを知ることができそうです。

シンボルを記載した場合はすべてasc（昇順）になります。
```rb
when Symbol
  order_column(arg.to_s).asc
```

ハッシュで記載した場合はkey（以下ではfield）によって処理が分岐します。
```rb
when Hash
  arg.map { |field, dir|
    case field
```

`Arel::Nodes::SqlLiteral`を用いた例をパッと記載できないのですが、Arelを用いた複雑な処理もorderメソッドの引数として記載できるようです。
`Arel::Nodes::SqlLiteral`の場合でも、そうでなくてもvalueを`dir.downcase`として実行しています。
```rb
when Arel::Nodes::SqlLiteral
  field.public_send(dir.downcase)
else
  order_column(field.to_s).public_send(dir.downcase)
end
```

dirには何を指定できるのかは、ハッシュのvalueに適当な値を記述して出力されるエラーから確認できます。

```rb
Book.order(title: :aaa)

~省略~/active_record/relation/query_methods.rb:1595:in `block (2 levels) in validate_order_args': Direction "aaa" is invalid. Valid directions are: [:asc, :desc, :ASC, :DESC, "asc", "desc", "ASC", "DESC"] (ArgumentError)
```

上記エラーからハッシュで記載する場合、valueには`[:asc, :desc, :ASC, :DESC, "asc", "desc", "ASC", "DESC"]`が指定できる、ということがわかりました。

またシンボル、ハッシュにかかわらず複数の引数を取れるのでさまざまな書き方ができます。

```rb
Book.order(:id, title: :DESC, :created_at => "asc")
#=> SELECT "books".* FROM "books" ORDER BY "books"."id" ASC, "books"."title" DESC, "books"."created_at" ASC
```

# 道中気になったメソッド

- flat_map
- public_send
- flatten!(flatten)

## flat_map

使用されていた箇所：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L1569

flat_map（collect_concatも同様）は要素をブロックに渡し、配列として連結します。

```rb
[:id, title: :DESC, :created_at => "asc"].flat_map { |a| a.is_a?(Hash) ? a.keys : a }
#=> [:id, :title, :created_at]

[%w[a b c], %w[d f]].flat_map { |e| e.map(&:upcase) }
#=> ["A", "B", "C", "D", "F"]
```

## public_send

使用されていた箇所：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L1587

public_sendはレシーバのpublicメソッドを第一引数にして呼び出します。

```rb
"abc".public_send(:upcase)
#=> "ABC"
```

## flatten!(flatten)

使用されていた箇所：https://github.com/rails/rails/blob/984c3ef2775781d47efa9f541ce570daa2434a80/activerecord/lib/active_record/relation/query_methods.rb#L1595

flattenは配列を再帰的に平坦化し、配列を返します。flatten!を使用すると自身も破壊的に平坦化します。

```rb
["a", [[1], 2], "b"].flatten
#=> ["a", 1, 2, "b"]
```

flattenとflatten!の違い

```rb
ary = ["a", [[1], 2], "b"]

# aryはそのまま
ary.flatten
#=> ["a", 1, 2, "b"]
ary
#=> ["a", [[1], 2], "b"]

# aryも平坦化される
ary.flatten!
#=> ["a", 1, 2, "b"]
ary
#=> ["a", 1, 2, "b"]
```

# 最後に

12月は弊社のアドベントカレンダーも進行中ですのでぜひご覧ください！
https://qiita.com/advent-calendar/2022/icare