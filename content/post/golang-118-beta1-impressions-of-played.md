---
title: "Golang 1.18 Beta1 でOption型を作って遊んだ"
date: 2022-02-03T03:33:33+09:00
draft: false
categories: ["Go"]
tags: ["Golang", "Go言語", "ジェネリクス"]
description: ジェネリクスが実装された1.18でOptionモナドを実装してみた内容です。ジェネリクスに夢を見すぎていたのですが色々と現実に引き戻されました。
---

ジェネリクスが実装されるとの事で前々から気にはなってた 1.18。

ジェネリクスと聞くだけで他にも色々実装されちゃうんじゃないかと期待を膨らませていました。（暗黙的な型変換とか継承とか、、でもそれらはなかった）

## 果たして消せるのか大量のエラーチェック

Goといえばエラーチェック。丁寧にエラーをチェックして適宜きちんとエラー処理しましょうという設計。

なので Exceptionは無い。

```example.go
value, err := repository.GetByID(id)
// これのこと👇
if err != nil {
    return err
}
```

方針はわかるけど3行使うのがどうしても気になってた。

ジェネリクスが来たらこの大量に書かれるエラーチェックが消えるんじゃないかと期待してた。

```repository.go
// イメージ。動きません。
func GetByID(id int) Error[Value] {
    ...
}
```

上記のようなリポジトリがあったとして、下記のように使うイメージ

```usecase.go
// イメージ。動きません。
errV := repository.GetByID(id).
          FlatMap(value => repository.GetByID2(value.ID)).
          FlatMap(value => repository.GetByID3(value.ID)
if errV.IsExists() {
    return errV.ToErr
}
```

こんな風に使えたらなあと。

最終的には `for yeild` とか実装されてたら IOモナドが使えるライブラリとか出てきてひたすら flatMapする感じで書けるのかなあと。

```usecase.go
// イメージ。動きません。
errV := for {
    value  <:= repository.GetByID(id)
    value2 <:= repository.GetByID2(value.ID)
    value3 <:= repository.GetByID3(value.ID)
} yeild value3
if errV.IsExists() {
    return errV.ToErr
}
```

もちろんイカ `<:=` オペレータは存在しないし、`for yeild` も無いです。

そういった想像を膨らませながら Optionモナドをミニマムで実装してみた。

## Optionモナドを実装してみた

Listよりもモナドとしては比較的理解しやすい Optionを Goで実装したら、どういう風に書けるのか確認したかったので書いてみた。

Option型に対して良い感じにロジックを書けそうならエラーをラップする前述 `Error[T]` 的なのも悪くない感じで書けるのでは？と考えたわけです。

```option.go
package main

type Option[T any] interface {
	isNone() bool
}

type Some[T any] struct {
	value T
}

func (s Some[T]) isNone() bool { return false }

type None[T any] struct{}

func (s None[T]) isNone() bool { return true }
```

上記のような感じに宣言しておいて、型パラメータを設定して呼ぶ感じです。


```main.go
func main() {
	fmt.Println(None[any]{}.isNone()) // true
	fmt.Println(Some[any]{}.isNone()) // false
}
```

※ `any` は `interface{}` のエイリアスみたいなもので 1.18から採用されてます。3文字で定義できるので便利です。

そして続き。メソッド類は下記のようになります。

```option.go
func Unit[T any](v T) Option[T] {
	return &Some[T]{value: v}
}

func Map[T, V any](opt Option[T], f func(T) V) Option[V] {
	switch t := opt.(type) {
	case *Some[T]:
		return &Some[V]{value: f(t.value)}
	case Some[T]:
		return &Some[V]{value: f(t.value)}
	default:
		return &None[T]{}
	}
}

func FlatMap[T, V any](opt Option[T], f func(T) Option[V]) Option[V] {
	switch t := opt.(type) {
	case *Some[T]:
		return f(t.value)
	case Some[T]:
		return f(t.value)
	default:
		return &None[T]{}
	}
}
```

※ `Some` の case文が冗長ですが、型switchでは fallthroughは使えないため冗長になってしまいました。


これで `Some` は値がある場合に使われ、`None` は値が無い場合に使われるようになり、どちらも `Option` として扱える状態となりました。

コレを使うとしたら下記のような感じ。

```main.go
package main

import "fmt"

func main() {
	dollars := 10

	optDollars := Unit(dollars)
	optYen := Map(optDollars, USDToYen)
	optZeikomi := FlatMap(optYen, ToJpTaxIncluded)
	optLabel := Map(optZeikomi, ToLabel)
	fmt.Println(optLabel)
	fmt.Println(optLabel.isNone())
}

func USDToYen(i int) float64 {
	return float64(i) * 114.68
}

func ToJpTaxIncluded(value float64) Option[string] {
	if float64(0) == value {
		return None[string]{}
	}
	return Some[string]{
		value: fmt.Sprintf("%f(税込み)", value*1.1),
	}
}

func ToLabel(priceStr string) string {
	return fmt.Sprintf("とってもお得な %s でご提供しております。", priceStr)
}
```

これを実行すると下記のような出力になる。

```console.log
▶ go run main.go option.go
&{とってもお得な 1261.480000(税込み) でご提供しております。}
false
```

`Some[string]` に `とってもお得な 1261.480000(税込み) でご提供しております。` が入っています。
`Some` なので `isNone()` は `false` となります。

`dollars` を `0` にして実行すると、下記のような結果となります。

```console.log
▶ go run main.go option.go
&{}
true
```

これは `ToJpTaxIncluded` で `None` になったためそれ以降の処理が行われなかったためこのような結果となります。

{{< post-ads >}}

### Unit, Map, FlatMap が関数である理由

前述のこれらの関数を見て **いやいや、レシーバにしたらメソッドチェーンでいけるやん** と見た 100%の人がそう思われたのではないかと思います。

結論、これはレシーバにできなかったため関数として定義しています。

メソッドチェーンで書けたら

```example.go
optLabel := Unit(dollars).
  Map(USDToYen).
  FlatMap(ToJpTaxIncluded).
  Map(ToLabel)
```

と書けるので非常にわかりやすくて見通しが良いと思います。

`Some` のレシーバに 下記のように実装してみます。

```option.go
// ※うごきません
func (s Some[T]) Map[V any](f func(T) V) Option[V] {
	return &Some[V]{value: f(s.value)}
}
```

（`Some` に生やせるので処理短くて良い感じ…）

これで `go run` するとエラーが出ます。

```console.log
▶ go run main.go option.go
# command-line-arguments
./option.go:43:21: methods cannot have type parameters
./option.go:43:22: invalid AST: method must have no type parameters
```

`Error[T]` 的な型を作って処理を簡素化したりできるかな、という夢は散りました。

このあたりの事は[この辺に](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md#no-parameterized-methods) 書いてあるみたいなので時間がある時に読んでおきたい。

## 触ってみての感想

とりあえず新しい機能に触れられて面白かった。

レシーバで型パラメータが扱えないのは残念ではあるけど、それを除いても言語としての幅が広がってるのを感じた。

例えば下記のように単純なReduce関数があったとして、どうReduce処理するかだけ与えてあげれば動くようになるのはとても強力に感じた。

```reduce.go
func main() {
	calcList := []int{
		1, 2, 3, 4, 5, 6, 7, 8, 9, 10,
	}
	// 👇どうreduceするか
	reducer := func(i1, i2 int) int { return i1 + i2 }
	sum := Reduce(calcList, reducer)
	fmt.Println(sum)
}

// Reduce バグあるかも
func Reduce[T any](sliceT []T, f func(t1, t2 T) T) T {
	result := sliceT[0]
	if len(sliceT) <= 1 {
		return result
	}
	for _, v := range sliceT[1:] {
		result = f(result, v)
	}
	return result
}
```

今までのGoだとこういうの書きたくても `reflect` 使わないと書けないとか、型ごとにReduce必要とかでどうも処理が冗長になってた感じある。

listutil的なのがかなり書きやすくなった感じありそう。

また、これまで誰が書いても大体同じようなコードになってた気がするけど、そうでなくなってきたのを感じた。

## 最後に

1.18を触ってみたけど、自分の担当するプロダクトでは今現在 1.11, 1.16がメインなので当分触る事は無さそう。（GAEで対応されたら使える）

map, flatMap, reduceなどなど関数が引数になる関数好きなので早く使いたい。機会があったら積極的に使っていきたいけど既存のGoユーザーは慣れてないのでとっつきにくそう…。

