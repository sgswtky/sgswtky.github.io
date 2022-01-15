---
title: "多相について今一度調べてみた"
date: 2018-12-14T19:17:05+09:00
draft: false
categories: ["Other"]
description: エンジニアになって数年が経つが、未だに多相についての理解が浅かったので調べてみた。
---

エンジニアになって数年が経つけど、たまに基礎の基礎を知らなかったりします。

最近 Scalaで関数型プログラミングの勉強をしてる時に `高カインド多相` というものが出てきました。

他の多相も全く知らないのに、名前を見るにすごく高度そうな多相。良い機会だから多相について調べてみようと思ったのが今回のきっかけです。

---

Wikipediaや、調べて出てきた他の方のブログ、本なんかを参考にしてます。
間違いがあったらご指摘頂けると🙏

---

## 多相性(ポリモーフィズム・多態性・多様性)

3つは全て全く同じものです。

- ポリモーフィズム
- 多態性
- 多様性

これらは何？と言われるとプログラム言語の型システムを取り扱う性質の事を指します。

型システム = 変数、式、オブジェクト、関数...などを指します。

---

## 部分型多相(派生型多態・サブタイピング多相・部分型付け)

- 同意語
  - 派生型多態
  - サブタイピング多相
  - 部分型付け

この多相を指して単にポリモーフィズムと呼ぶ場合もあります。

複数種類の型を１つの型のように扱える性質の事を指します。

例えば、 Animal を extendsした3つの型があるとします。

```Subtype.scala
trait Animal {
  def call(): String
}
case class Cat() extends Animal { override def call(): String = "bark" }
case class Dog() extends Animal { override def call(): String = "mew" }
case class Cow() extends Animal { override def call(): String = "low" }
```

3つの型は、Animal型として扱えるので、これが実現できる Scalaは部分型多相の性質を持っています。

```signture.txt
// 引数が Animal であるため部分型多相
def connectLeads(a: Animal): Unit
// 戻り値が Animal であるため部分型多相
def randomAnimal(): Animal
```

もう少し詳しく説明すると、上記の **connectLeads・randomAnimalは、引数・戻り値が Animal** です。

これらは **Animalの実態が `Cat`, `Dog`, `Cow` のどれであるかについては関与していません** 。

これが部分型多相です。

ScalaやJavaは、extends（継承）によってこれらの性質が提供されています。

`Animal` と `Cat`, `Dog`, `Cow` は、以下のように表記できるらしいです。（Wikipediaに書いてあった）

```Example.txt
Cat <: Animal
Dog <: Animal
Cow <: Animal
```

（この表記が何なのかは知りません）

---

## アドホック多相（オーバーロード）

**要はオーバーロードがその性質に値** します。

Scalaでは以下のように表現できると思います。

```Adhoc.scala
case class Book()
case class MobilePhone()
case class Computer()
class SecondHandBuyer {
  def sell(book: Book): Int = 100
  def sell(mobilePhone: MobilePhone): Boolean => Int =
    isSetAccessories => if (isSetAccessories) 20000 else 17000
  def sell(computer: Computer): Int = 40000
}
```

良いサンプルではないですが、`sell` 関数は特定のオブジェクトを渡す事で売却金額が得られます。

引数・戻り値の型はそれぞれバラバラですね。オーバーロードは異なる引数値だけど同名メソッドを定義する事が可能です。

このような **同名で全く関係ない異なる型の引数・戻り値に対応、関数の振る舞いはそれぞれ異なるような関数をアドホック多相** といいます。

実はこの3種類の `sell` メソッドは、コンパイラから見ると全く別の関数3つとして認識されているらしい...。

`アドホック` （その場しのぎ）の由来は、型システムの基本的な機能でない事を指すため `アドホック` という命名だそうです。

---

{{< post-ads >}}

## パラメータ多相（パラメトリック多相）

- 同意語
  - パラメトリック多相

**特定の型を指定しなくても良い性質の事** を指します。

Scalaでいうと、下記のような「A型」が該当します。

```Parameter.scala
def get[A](l: List[A], n: Int): A = l match {
  case _ if l.length < n => throw new Exception
  case h :: Nil => h
  case _ :: t => get(t, n - 1)
}
get(List(1,2,3), 3)
get(List("a", "b", "c"), 3)
```

`get` は Listの `n番目` の中身を返します。

このコードには、`List[String]`, `List[Int]` を渡しても動作します。

`List[A]` というのは Listに入った `A型` ならなんでも良いという事です。

Javaなどでは、**ジェネリクス** がこれに該当します。

---

## 高階多相（高カインド多相・HigherKind）

- 同意語
  - 高カインド多相

Scalaでよく見かける `[F[_]]` の事です。 Haskellや C++ も高階多相らしいです。

これは、型コンストラクタ（≠ 普通の型）を受け取り型を返す性質で、型コンストラクタは `List, Stream, Option` などが該当します。

例えば、以下の モナドのトレイトと ファンクタのトレイトは高階多相です。

```HigherKind.scala
trait Monad[F[_]] {
  def unit[A](a: => A): F[A]
  def flatMap[A,B](ma: F[A])(f: A => F[B]): F[B]
}

trait Functor[F[_]] {
  def map[A,B](fa: F[A])(f: A => B): F[B]
}
```

これが実現できる Scalaは、高階多相であることがわかります。

---

## 余談:ダックタイピング

多相とは関係ありそうで関係無いのですが、ダックタイピングというものもあります。

これはいわゆる Go言語でいう所の Interfaceです。

```example.go
package main

import (
	"fmt"
)

type Tori interface{
	naku()
}

type Ahiru string
func(Ahiru) naku() {
	fmt.Println("kuwaa")
}

type Karasu string
func(Karasu) naku() {
	fmt.Println("kaaa")
}

func main() {
	var ahiru Tori = Ahiru("")
	var karasu Tori = Karasu("")
	ahiru.naku()
	karasu.naku()
}
```

`Ahiru` と `Karasu` は `Tori` インターフェースを満たしているので、 `Tori` として扱えます。

> もしもそれがアヒルのように歩き、アヒルのように鳴くのなら、それはアヒルである

という言葉があり、そのインターフェースの通りに実装すればその型として扱えるというものです。

実は **Go言語の場合、ダックタイピングを使う事で初めて部分型多相を実現** できます。

ダックタイピングは、他には `Perl`, `Python`, `Ruby` で使用する事が可能です。

---

## 最後に

今回は様々な多相について調べてみました。
ここに書いた事は恐らく基礎だと思うのですが、知らないことが多くありました…。

今回調べてみて、高階多相についても深く知る事ができたので良かったです。
