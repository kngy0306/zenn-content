---
title: "Rubyの構造体について"
emoji: "😎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Ruby"]
published: true
---

社内での勉強会で使用していた書籍ではじめて`OpenStruct`に出会い、それに派生してRubyでの構造体について調べてまとめました。

# 構造体とは

そもそも構造体ってなんだろうと調べたとき、下記記事の

> 元々ある型の変数を寄せ集めて新しい型を作れるのが構造体の特徴です。

という説明がとてもしっくりきました。

[構造体とは｜「分かりそう」で「分からない」でも「分かった」気になれるIT用語辞典](https://wa3.i-3-i.info/word13243.html)

# Rubyで構造体を扱う

Rubyで構造体を扱うときの方法として、以下の2つについて取り上げます。

- Struct
- OpenStruct

## Struct

組み込みライブラリであり、`Struct.new(*args, keyword_init: nil)`という構文でStructのサブクラスを返します。

Structには`keyword_init`という引数が存在します。これに`keyword_init: true`を指定することで、構造体の初期化時にキーワード引数を使用するようにできます。

```ruby
user = Struct.new(:name, :age, keyword_init: true)
alice = user.new(name: "Alice", age: 20)
p [alice.name, alice.age] # ["Alice", 20]
```

これにより引数の何番目にどの値が入るんだっけ？といったことを防ぐことができます。
Ruby3.2からはこれがデフォルトになるため、わざわざ`keyword_init: true`を記載する必要はなくなるようです。

https://www.ruby-lang.org/en/news/2022/09/09/ruby-3-2-0-preview2-released/#:~:text=Struct,Feature%20%2316806%5D

https://docs.ruby-lang.org/ja/latest/class/Struct.html

## OpenStruct

標準ライブラリであり、`require 'ostruct'`でライブラリを読み込む必要があります。`OpenStruct.new(hash = nil)`という構文でOpenStructオブジェクトを返します。

```ruby
require 'ostruct'
bob = OpenStruct.new({name: "Bob", age: 25})
p [bob.name, bob.age] # ["Bob", 25]
```

要素を動的に追加できる点が`Struct`とは異なります。

```ruby
require 'ostruct'
bob = OpenStruct.new({name: "Bob", age: 25})
puts bob # #<OpenStruct name="Bob", age=25>

# 初期化後に要素を追加
bob.gender = "male"
bob.height = 170
puts bob # #<OpenStruct name="Bob", age=25, gender="male", height=170>
```

要素を動的に追加できることは便利でもありますが、既存のメソッドを上書きしてしまうなどの危険性もあります。
以下の記事では`methods`メソッドが上書きされてしまうことが例として記載されています。

https://ruby-doc.org/stdlib-3.1.2/libdoc/ostruct/rdoc/OpenStruct.html#:~:text=Builtin%20methods%20may%20be%20overwritten%20this%20way%2C%20which%20may%20be%20a%20source%20of%20bugs%20or%20security%20issues%3A

上記特徴やパフォーマンスが悪いことなどの理由から、OpenStructは非推奨である旨が`ostruct.rb`のソースコード上の注意事項に記載されています。

https://github.com/ruby/ostruct/blob/69f6661f6219175adc8949ff61ff10b558bc8494/lib/ostruct.rb#L67-L107

# 構造体の用途

構造体を使用するのは、「データをいくつかの固まりで保持したいが、クラスを使うまでもないとき」だと現時点で自分は考えています。

**例）「名前」と「年齢」が入った「ユーザ」というデータを保持したいとき**

## Structを使用したとき

```ruby
user = Struct.new(:name, :age, keyword_init: true)
alice = user.new(name: "Alice", age: 20)
p [alice.name, alice.age] # ["Alice", 20]
```

## クラスを使用したとき

```ruby
class User
  attr_reader :name, :age

  def initialize(name:, age:)
    @name = name
    @age = age
  end
end

alice = User.new(name: "Alice", age: 20)
p [alice.name, alice.age] # ["Alice", 20]
```

上記のように値を保持し、値を出力するだけ、といった用途でクラスを使用するのは大げさかなと感じました。

# StructとHashの使い分け

Hashを使用しても構造体と同じような要件を満たすことができます。

```ruby
charlie = {name: "Charlie", age: 30}
p [charlie[:name], charlie[:age]] # ["Charlie", 30]
```

StructとHashには以下のような違いがあります。

- Structは要素の取得時にシンボルと文字列の区別がない
- Hashは要素を動的に追加できる
- 存在しない要素名にアクセスしたとき
- 要素の取得方法

## Structは要素の取得時にシンボルと文字列の区別がない

Structは要素の取得時に使用するキーに対して、シンボルと文字列、どちらでも取得ができます。
Hashではシンボルで要素を追加した場合、文字列では取得できません。

```ruby
user = Struct.new(:name, :age, keyword_init: true)
bob = user.new(name: "Bob", age: 20)
p [bob["name"], bob[:age]] # ["Bob", 20]

charlie = {name: "Charlie", age: 30}
p [charlie["name"], charlie[:age]] # [nil, 30] ← "name"というキーは存在しない
```

## Hashは要素を動的に追加できる

```ruby
user = Struct.new(:name, :age, keyword_init: true)
bob = user.new(name: "Bob", age: 20)
bob.gender = "male" # エラー undefined method `gender='

charlie = {name: "Charlie", age: 30}
charlie[:gender] = "male" # OK
```

## 存在しない要素名にアクセスしたとき

**Structは要素の取得時にシンボルと文字列の区別がない**の項目でも確認したとおり、Hashでは存在しない要素を取得してもエラーにはならないため、typoに気づきにくくなります。

```ruby
user = Struct.new(:name, :age, keyword_init: true)
bob = user.new(name: "Bob", age: 20)
p [bob.nema, bob.age] # エラー no member 'nema' in struct (NameError)

charlie = {name: "Charlie", age: 30}
p [charlie[:nema], charlie[:age]] # [nil, 30]
```

## 要素の取得方法

Structはドット記法でも要素を取得できます。

```ruby
user = Struct.new(:name, :age, keyword_init: true)
bob = user.new(name: "Bob", age: 20)
p [bob.name, bob.age]     # ["Bob", 20]
p [bob[:name], bob[:age]] # ["Bob", 20]

charlie = {name: "Charlie", age: 30}
p [charlie.name, charlie.age]     # ["Charlie", 30]
p [charlie[:name], charlie[:age]] # エラー undefined method `name'
```

## StructとHashの使い分けについての感想

保持する要素数が決まっていないときや、動的に要素を追加することがある場合はHashを使用し、それ以外のときはStructを使うほうが良さそうだなと感じました。

# 参考文献

https://qiita.com/scivola/items/6493e2fb3de657753c44

https://qiita.com/megane42/items/5bfdb0764fa575efbdab
