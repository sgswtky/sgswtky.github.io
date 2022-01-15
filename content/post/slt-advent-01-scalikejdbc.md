---
title: "ScalikeJDBCのクォート付く付かない挙動調べてみた"
date: 2018-12-13T21:30:21+09:00
draft: false
categories: ["Scala"]
tags: ["ScalikeJDBC"]
description: 所属チームのプロダクトで使用している ScalikeJDBCのクォーテーションが付く付かないの挙動が気になったので調べてみた。
---

ScalikeJDBCというScalaのライブラリを業務で使用しているのですが、
一部不明な点があったので調べてみました。

## ScalikeJDBC（JDBCを用いたScalaのORM）

![ScalikeJDBC_ロゴ](/images/scalikejdbc-logo.png)

ScalikeJDBCは、JDBCをラップしたScalaのライブラリです。
様々なDBに対応していて、直感的に書けるQueryDSLで効率的にSQLが書けます。

自分もいくつかのScalaプロジェクトで、MySQLのコネクタとして採用しています。
日本語の情報が多く、Scala触り始めの方でも扱う敷居は低いのではないかと思います。

## QueryDSL（クエリ直接記述）

ScalikeJDBCのDSLはコード中にSQLを記述するように書けて、以下の2種類の書き方が可能です。

```example.scala
val userMap = sql"""
    select * from user
    where created_at > '2018-10-01'
  """.map(_.toMap).list.apply
```

SQLInterpolation という機能を使用している場合は、クエリビルダのようにクエリを組み立てられます。

```example.scala
val u = User.syntax("u")
val groupMember = withSQL {
  select.from(User as u).where.gt(u.createdAt, createdAt)
}.map(_.toMap).list.apply
```

## 挙動がわからず確認してみようと思ったきっかけ

QueryDSLを使って、以下のように WHERE句に LocalDateの値を使用してSELECT句のクエリを流しました。

```example.scala
val value = LocalDate.now()
val userMap = sql"""
  select * from user
  where value = $value
""".map(_.toMap).list.apply
```

👇下記のようにSQLが流れる想定でしたが、

```
select * from user where value = '2018-12-13'
```

👇以下のSQLが実行されてエラーとなりました。

```
select * from user where value = 2018-12-13
```

つまり、自動でクォートされる型・されない型があるようです。

{{< post-ads >}}

## "クォート" が付与される型と クォート が付与されない型

今回は、この時にクォートされる型とされない型について SELECT句で調べてみました。

公式に対応されている型は、👇下記リンク先に書かれています。
http://scalikejdbc.org/documentation/sql-interpolation.html

Scala組み込みのオブジェクトや、`java.sql` , `java.util.Date`, `org.joda.time.`, `java.time` が含まれている事がわかります。

👇下記のように、変数 **value** の型を変えてQueryDSL内部でSQL実行、実行されたSQLにクォートが付加されたかされなかったかを見ていきます。

```example.scala
val value:A = ???
val userMap = sql"""
  select * from user
  where value = $value
""".map(_.toMap).list.apply
```

まずは  **Scala組み込みのオブジェクト** 。

| 型 | クォート |
|---|---|
| Int | × |
| Int | × |
| Short | × |
| Long | × |
| Float | × |
| Double | × |
| BigInt | × |
| BigDecimal | × |
| String | ○ |

※ ○ = 付加される、× = 付加されない

ここまでは想定通りの結果でした。
次に **DateTime系の時間を扱うオブジェクト** の結果を見ていきます。

| 型 | クォート | フォーマット |
|---|---|---|
| org.joda.time.DateTime | ○ | '2018-12-13 20:14:34.913' |
| org.joda.time.LocalDateTime | × | 2018-12-13 20:14:53.638 |
| org.joda.time.LocalDate | × | 2018-12-13 |
| org.joda.time.LocalTime | × | 20:16:01 |
| java.sql.Date | ○ | '2018-12-13 00:00:00.0' |
| java.sql.Time | ○ | '1970-01-01 00:00:00.0' |
| java.sql.TimeStamp | ○ | '2018-12-13 20:16:41.704' |
| java.time.ZonedDateTime | × | 2018-12-13T20:11:25.862681+09:00[Asia/Tokyo] |
| java.time.Instant | × | 2018-12-13T11:12:07.373849Z |
| java.time.LocalDateTime | × | 2018-12-13T20:12:40.509394 |
| java.time.LocalDate | × | 2018-12-13 |
| java.time.LocalTime | × | 20:13:51.527330 |
| java.util.Date | ○ | '2018-12-13 21:16:28.688' |

※ ○ = 付加される、× = 付加されない

予想を裏切る結果となったのではないでしょうか。

`org.joda.time.DateTime`, `java.sql.Date`, `java.sql.Time`, `java.sql.TimeStamp`, `java.util.Date` あたりは、ちゃんとクォートが付加されていますね。

しかし `java.sql.Time` は、 `Time` なので時間の情報しか所持していないように見えるにも関わらず、年月日のエポック秒が表示されてしまいました。

深く追っていないのですが、`java.sql.Time`のコメントに以下のように書いてあったので、

```
The date components should be set to the "zero epoch" value of January 1, 1970 and should not be accessed.
```

ScalikeJDBCもしくはその先で使用しているライブラリが、`java.sql.Time` の言う日付コンポーネントにアクセスしているという事が挙動からわかりました。

## 最後に

ScalikeJDBCを使用すると、**型によってクォートが付加されない事がある** という事がわかりました。

クォートが付かない型を無理矢理使い続けると、自前でクォートを付けざるを得ないため、 **スマートに書けるはずの QueryDSLがスマートでは無くなってしまいます** 。

クォート付けるために implicit使うのも個人的になんとなく好きでなく、できればライブラリにやってほしい…という気持ちがあります。

今回 `org.joda.time.LocalDate`を使う事になってこの問題にぶち当たったのですが、恒久的な対応はせず、クエリ部分に 一時的に `org.joda.time.DateTime` を代用して回避しています。

今回の記事ではライブラリの内部を詳細に調査していないので、ScalikeJDBCが原因か、JDBCが原因かまでは追っていません。

また調査の時間が取れたら記事を書きたいと思います。
