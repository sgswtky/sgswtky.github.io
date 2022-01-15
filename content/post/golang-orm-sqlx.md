---
title: "jmoiron/sqlx - Golang O/R Mapper触ってみた"
date: 2018-07-10T13:53:07+09:00
draft: false
categories: ["Go"]
tags: ["ORMapper", "sql", "library"]
description: jmoiron/sqlx という GolangのORMを試しに使ってみました。簡単に CRUDで試したりトランザクション処理を試してみたり、NamedExecという機能も使ってみました。感想としては、極々薄い感じのORMで個人的に好みのORMでした。
---

## このORM触ってみようと思ったきっかけ

会社で僕のチームでは3層WebアプリでGoをサーバーサイドとして使ってます。

フロントはVuejsをでSPAとしてブラウザで動作する感じです。
AWS上で動いていて、サーバーサイド（Go）とRDS（MySQL）のデータのやり取りで `dbr` というORMを使っているのですが、以前このORMで一点問題になる事があった。

日本をターゲットにしたプロダクトを作っているので日本を基準にした時刻でDBとやり取りしたいので、JSTとして時刻を扱いたいけど `dbr` によって強制的にUTCとなる。
UTCからJSTに無理矢理変換するなどして対応しているのですが、正直かっこ悪いので根本的な対応をしたいです。

色々調べてみた所dbrのコミッタはこれをissueでなく仕様としているような発言が以下のissueから見て取れる。

https://github.com/gocraft/dbr/issues/96

このため、この先 `dbr` でなく別のORMを使用することを検討しています。
軽く調べてみて今回紹介する `jmoiron/sqlx` が個人的にすごく好きな感じだったので紹介したいと思います。

## GoのORMに超個人的に求めてるもの

### なんとなくGoに合う感じがする

とても曖昧ですが、Goをしばらく触っていて感じるのは多機能で色々やってくれるモノは基本的にGoに合わない気がしています。
例えばJavaのspring系のフレームワークであったりMyBatisのようなXMLとかアノテーションたくさん使っていろんな機能がある奴。


Goはどちらかというと小さいライブラリを組み合わせて使うほうが向いている感じがあって、それぞれが疎結合になってるというのが一番綺麗でわかりやすいような気がしてます。（そして個人的に好き）

### 超個人的に求めてるものリスト

- なんでもやってくれない、なるべく最低限
- ActiveRecordみたいな黒魔術でない
- リレーション張る機能が無い
- 構造体とのマッピングができる
- 自分で書くコードが減りそう
- 読みやすいか
- 生SQLが書ける

## sqlxのREADMEを流し読みした感じの印象

 - database/sql に一連の拡張機能追加してるだけ
 - database/sql 使ってるなら簡単に導入できる
 - コンセプトがシンプル

な印象を受けて、この時点で **良さそう** と思った

## CRUDを簡単に作る等で触ってみた

簡単にCRUD、NamedExecというsqlxが作ったメソッド、トランザクションを一通り試してみた。

### select句

selectでマッピングを行ってくれる。

カラム名は構造体のタグで明示的に指定する。

```example.go
db, _ := sqlx.Connect("mysql",
  "root:rootpasswd@(example.host:3306)/example?parseTime=true")
type UserStatus struct {
	UserID string                `db:"user_id"`
	UserName string              `db:"user_name"`
	UserStatusDescription string `db:"user_status_description"`
}
userStatus := []UserStatus{}
err := db.Select(&userStatus, `
  select
    user_id,
    user_name,
    user_status_description
  from
    users u
  inner join user_status us
    on u.user_status = us.user_status_id
`)
```

上記のようなコードで、 `userStatus` にSQLの結果一覧が入ってくるようになる。

### insert, update, deleteなど更新系クエリ

その他のクエリは以下のように `tx.Query` を使う

```example.go
r, err := db.Query("update users set user_name = 'update' where user_id = ?", 1)
```

戻り値には Rowsとerorrが入る

{{< post-ads >}}

### NamedExecメソッドについて

実際に触っていると `NamedExec` というメソッドを見つけた。

このメソッドが便利そうで、 `map[string]interface{}` で渡して良い感じにクエリが組み立てられそうだと感じた。

いちいちただの `insert` とか `update` を書くのが面倒くさいから構造体渡したら作成・更新やってほしいと思っていて、実際にプロダクションのコードに `reflect` を使って構造体渡すだけで作成・更新してくれるメソッドを作っていたりする。

（これは多分ActiveRecordと同じだと思うのですが、チームでメンテできるし…ということでそんな感じになってます）


```example.go
// 公式READMEより引用
_, err = db.NamedExec(`
    INSERT INTO person (first_name,last_name,email) VALUES (:first,:last,:email)
`,
    map[string]interface{}{
        "first": "Bin",
        "last": "Smuth",
        "email": "bensmith@allblacks.nz",
})
```

`NamedExec` はそういう面倒くささを払拭するのに結構使えそうだと思った。

例えば以下のコードで構造体のカラム名と中身のmap（`map[string]interface{}`）が取れる。

これと `NamedExec` を組み合わせる事で構造体を渡すことで作成・更新してくれるメソッドが作れそうだと思った。

```example.go
func namedExecMap(inf interface{}) (entityMap map[string]interface{}) {
	tof := reflect.TypeOf(inf)
	vof := reflect.ValueOf(inf)
	for i := 0; i < tof.NumField(); i++ {
		tag := tof.Field(i).Tag.Get("db")
		entityMap[tag] = vof.Field(i).Interface()
	}
	return entityMap
}
```

### トランザクションも動かしてみた

トランザクションも一応試してみた。

```example.go
tx, err := db.Beginx()
```

最初、 `db.Begin` でトランザクションを開始していた所、 `NamedExec` が使えずがっかりしていたが取得するメソッドが違っただけで `Beginx` から取得する事で使えた。帰ってくる型が違って以下のように返る。

| 呼び出すメソッド | 戻り値の型 | NamedExec使える |
|:--------------|:------------:|-------------:|
| db.Beginx() | database/sql/Tx | ○ |
| db.Begin() | jmoiron/sqlx/Tx | × |

InteliJで書いてるとタイピング中に `db.Begin()` のほうが優先してサジェストされるのでいつか無意味にハマりそうだなと思った。

## 動かしてみて気になった所

### selectでdatetime型がちゃんと認識しない

この問題は `sqlx` 固有の問題ではなく、 `database/sql` を生で使ってても発生する。

dbrだと `time.Time` への変換は行われる。でも、呼び出されるメソッドを時前で実装すればどうにかなる。

- `Scan(value interface{}) error`
- `Value() (driver.Value, error)`

```self_time_type.go
type DBTime struct {
	value time.Time
}
// select用
func (dbTime *DBTime) Scan(value interface{}) error {
	t, err := time.Parse("2006-01-02 15:04:05 +0000 MST", fmt.Sprintf("%s", value))
	if err != nil {
		return err
	}
	dbTime.value = t
	return nil
}
// 時間取る用
func (dbTime *DBTime) Time() time.Time {
	return dbTime.value
}
// プレースホルダ用
func (DBTime DBTime) Value() (driver.Value, error) {
	return DBTime.Time().Format("2006-01-02 15:04:05"), nil
}
```

`database/sql` の対応してない箇所をあえて `sqlx` が対応してないように見えて好印象だった。
この件については`time.Time` が公式パッケージなんだから対応したら良いのに、と自分は思ってたりします。


## 実際に動かしてみて

詳しい使い方については `jmoiron` さんのgithub Pagesに詳細に書いてあった。

http://jmoiron.github.io/sqlx/

極々薄い感じのORMでとても良い感じだった。

database/sql をほんの少し拡張しただけ、みたいな感じで責任範囲が明確。他の責任範囲が明確なライブラリと組み合わせる事で更に便利に使えそう。

ORMリプレースが本格的に動き出したら、もっと踏み込んで触ってみたい。
