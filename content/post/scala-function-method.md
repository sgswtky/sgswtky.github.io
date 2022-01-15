---
title: "ScalaのFunctionが実装するメソッド"
date: 2018-10-23T09:30:08+09:00
draft: false
categories: ["Scala"]
tags: ["Function"]
description: Scalaの理解を深めるため Function がデフォルトでどんなメソッドが使えるのか調べてみた。
---

しばらく放置してた。反省。

Scalaで関数を定義すると、すでにその時点でいくつかのメソッドが実装されている。
（レシーバとも言う？）

`Function1` で実際にどんなメソッドを実装しているかを見てみた。
以下の4つを実装してるっぽい。

```
compose
andThen
apply
toString
```

| method | description |
| --- | --- |
| compose | 関数の合成 |
| andThen | 関数の合成 |
| apply | 関数の適用 |
| toString | String型に変換 |

## compose（関数合成を行うメソッド）

- レシーバの関数を後に適用
- ２つの `Function1` から新しい `Function1` を生成

引数に ` func{n}` を結合するメソッドを３つ compose してみる

```example.scala
object method extends App {
  def exampleFunc1(s: String): String = s + " func1"
  def exampleFunc2(s: String): String = s + " func2"
  def exampleFunc3(s: String): String = s + " func3"

  val ef1: String => String = exampleFunc1
  val ef2 = exampleFunc2 _
  val ef3 = exampleFunc3 _

  val f = ef1 compose ef2 compose ef3
  val r = f("EXAMPLE")
  println(r)
}
```

```output.txt
EXAMPLE func3 func2 func1
```

composeはレシーバのメソッドを後ろから実行するような動きをするみたい。

↓この部分は実際は

```
val f = ef1 compose ef2 compose ef3
```

↓こう呼び出される

```
val f = ef1.compose(ef2).compose(ef3)
```

順番的にはこうなる

```
1. ef1.compose(ef2).compose(ef3)
2. {ef2のあとef1が実行されるFunction1}.compose(ef3)
3. {ef3のあとef2のあとef1が実行されるFunction1}
```

## andThen（関数合成を行うメソッド）

- こちらも関数合成
- composeとは逆でレシーバを先に適用
- ２つの `Function1` から新しい `Function1` を生成

引数に ` func{n}` を結合するメソッドを３つ andThen する

```example.scala
object method extends App {
  def exampleFunc1(s: String): String = s + " func1"
  def exampleFunc2(s: String): String = s + " func2"
  def exampleFunc3(s: String): String = s + " func3"

  val ef1: String => String = exampleFunc1
  val ef2 = exampleFunc2 _
  val ef3 = exampleFunc3 _

  val f = ef1 andThen ef2 andThen ef3
  val r = f("EXAMPLE")
  println(r)
}
```

```output.txt
EXAMPLE func1 func2 func3
```

andThenは composeとは逆の動きをする認識

```
ef1.andThen(ef2).andThen(ef3)
{ef1の後ef2を適用する関数}.andThen(ef3)
{ef1の後ef2の後ef3を適用する関数}
```

{{< post-ads >}}

## apply（コンストラクタ）

- 関数を適用する時にすでに呼ばれてる
- 意識せずにいつもシンタックスシュガーで呼び出してる

```example.scala
object method extends App {
  def exampleFunc1(s: String): String = s + " func1"

  val ef1: String => String = exampleFunc1
  val r1 = ef1.apply("EXAMPLE")
  val r2 = ef1("EXAMPLE")

  println(r1)
  println(r2)
}
```

r1 と r2 は同じ結果となる

## toString

- 継承元に存在しない場合は `java.lang.Object` が呼ばれる認識
- `Function1` の場合は overrideしてるけどこれは呼ばれてなくてよくわからない。。。

```example.scala
object method extends App {
  def exampleFunc1(s: String): String = s + " func1"

  val ef1: String => String = exampleFunc1
  println(ef1.toString())
}

```

InteliJで上記の `toString` を辿ると `Function1` で以下のような実装であることがわかる

```Function1.scala
override def toString() = "<function1>"
```

でも結果は下記のようになる。。。謎

```output.txt
cop.method$$$Lambda$6/817406040@61009542
```

# Function2（引数2つのメソッド）

これまで `Function1` を例として見てきたけど、Function `2` では実装してるメソッドが異なる。

`Function1` で実装されていた `compore, andThen` は実装されておらず、代わりに以下のメソッドが実装されている。

```list.txt
curried
tupled
```

## curried（カリー化）

定義した関数の引数をまとめてカリー化が行える。

```example.scala
object method extends App {

  def f2(s1: String, s2: String): String = s1 + s2

  val f: (String, String) => String = f2

  // non curry
  val a1: String = f("EX", "AMPLE")
  println(a1)

  // curry
  val curry: String => String => String = f.curried
  val c1: String => String = curry("EX")
  val c2: String = c1("AMPLE")
  println(c2)
}
```

出力は以下のようになる

```output.txt
EXAMPLE
EXAMPLE
```

## tupled（引数のタプル化）

引数をタプルで順番に受け取れるようにしてくれる。

```example.scala
object method extends App {

  def f2(s1: String, s2: String): String = s1 + s2

  val f: (String, String) => String = f2
  val t: (String, String) = ("EX", "AMPLE")


  // *** compile error ***
//  val r = f(t)
//  println(r)
  // tupled
  val tupled = f.tupled
  val tr: String = tupled(t)
  println(tr)
}
```

出力は以下のようになる

```output.txt
EXAMPLE
```

`tupled` 変数に型指定したけどなぜかエラーになる。よくわからない…。
`tupled` の型は恐らく `(String, String) => String` である。

`compile error` の部分は元の関数 `f` に対してタプルを渡している。
これは単純に型が違うのでエラーとなる。 tupledを呼び出す事でタプルで受け付ける関数に変換できる。

## 触ってみたまとめ

scalaは全てオブジェクトになっていて、関数ですらオブジェクトになっている。
こういう細かい実装されてる関数に詳しくなってscalaの使いたい機能をサクサク使えるようになりたい。

