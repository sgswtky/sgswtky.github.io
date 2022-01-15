---
title: "go-xorm/xorm - Golang O/R Mapper触ってみた"
date: 2018-07-15T23:17:35+09:00
draft: false
categories: ["Go"]
tags: ["ORMapper", "sql", "library"]
description: go-xorm/xorm という ORMを触った時の記事。簡単なCRUD処理、JOIN、または直接SQLを実行して試してみました。
---

先日 `sqlx` を触ってみて、他のORMも触ってみたいと思って触ってみた

### DB接続の定義

接続は他のORMと同じような感じ。

```example.go
package orm

import (
	"testing"
	_ "github.com/go-sql-driver/mysql"
	"github.com/go-xorm/xorm"
)

var engine *xorm.Engine

func TestXorm(t *testing.T) {
	var err error
	engine, err = xorm.NewEngine("mysql", "root:rootpasswd@tcp(localhost):3306/example?parseTime=true&charset=utf8")
	if err != nil {
		panic(err)
	}
}
```

### 今回テストで操作するDDL＆構造体

```example.sql
CREATE TABLE `users` (
  `user_id` varchar(11) NOT NULL DEFAULT '' COMMENT 'userID',
  `user_name` varchar(255) NOT NULL COMMENT 'userName',
  `created_at` datetime NOT NULL COMMENT 'createdAt',
  `created_user` varchar(255) DEFAULT NULL COMMENT 'createdUser',
  `updated_at` datetime NOT NULL COMMENT 'updatedAt',
  `updated_user` varchar(255) DEFAULT NULL COMMENT 'updatedUser',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='user';
```

```example.go
type Users struct {
	UserId string
	UserName string
	CreatedAt time.Time `xorm:"created"`
	CreatedUser string
	UpdatedAt time.Time `xorm:"updated"`
	UpdatedUser string
}
```

これで CRUDを動かしてみる。

### GET (SELECT * FROM .. LIMIT 1) を動かしてみる

`SELECT * FROM users LIMIT 1`

```get.go
user := Users{}
has, err := engine.Get(&user)
```

hasには値を取得できたかどうかが入る。 `user` に結果が入る

### Find (SELECT * FROM ..) を動かしてみる

- 通常のFind。全て取得する

```find.go
user := make([]Users, 0)
err = engine.Find(&user)
// SELECT * FROM users
```

- 第2引数にUsers構造体を指定することで WHERE句として扱ってくれる

```find.go
user := make([]Users, 0)
err = engine.Find(&user, Users{UserName: "bbbb"})
// SELECT * FROM users WHERE user_name = "bbbb"
```

- 構造体でないパターンで指定も可能

```find.go
err = engine.Where("user_name = ?", "bbbb").Find(&user)
// SELECT * FROM users WHERE user_name = "bbbb"
```

### JOINも検証

もう一つテーブルが必要なので下記を定義して検証を進めた

```example.sql
CREATE TABLE `friends` (
  `user_id` varchar(11) NOT NULL DEFAULT '' COMMENT 'userID',
  `friend_id` varchar(11) NOT NULL COMMENT 'friendID',
  `created_at` datetime NOT NULL COMMENT 'createdAt',
  `created_user` varchar(255) DEFAULT NULL COMMENT 'createdUser',
  `updated_at` datetime NOT NULL COMMENT 'updatedAt',
  `updated_user` varchar(255) DEFAULT NULL COMMENT 'updatedUser',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='user';
```

```example.go
type Friends struct {
	UserId string
	FriendId string
	CreatedAt time.Time `xorm:"created"`
	CreatedUser string
	UpdatedAt time.Time `xorm:"updated"`
	UpdatedUser string
}
```

今回、usersテーブルとfriendsテーブルの結果が欲しいため以下のような構造体とレシーバを作成する

```users_friends.go
type UsersFriends struct {
	Users `xorm:"extends"`
	Friends `xorm:"extends"`
}

func (UsersFriends) TableName() string {
	return "users"
}
```

- `friends` テーブルとJOINしてみる

```join.go
usersFriends := make([]UsersFriends, 0)
err = engine.Join("INNER", "friends", "friends.user_id = users.user_id").Find(&usersFriends)
```

`UsersFriends` 構造体にちゃんと結果が入ってきた。JOINの時に既存の構造体を使いまわしてるのが割とわかりやすくて良いかもしれない。

### Iterateで反復処理の記述も可能

- 全てのユーザーの名前に「さん」付する処理。Iterate意外に便利そうな気がする
- ただ、SQLで付けるのと切り分けが難しそう…。

```iterate.go
users := make([]Users, 0)
err = engine.Iterate(new(Users),
  func(i int, bean interface{}) error {
    user := bean.(*Users)
    user.UserName = user.UserName + "さん"
    users = append(users, *user)
    return nil
  })
```

### goのsql標準にもある Rows から取り出して加工する処理

- rowsの処理も書ける

```rows.go
users := make([]Users, 0)
rows, err := engine.Rows(users)
if err != nil {
  panic(err)
}
defer rows.Close()
for rows.Next() {
  // ...
}
```

### sum 合計するクエリを発行する関数もある

- sumできるカラムの存在するテーブルを作成して取得してみる

```sum.sql
CREATE TABLE `sums` (
  `user_id` varchar(11) NOT NULL DEFAULT '' COMMENT 'userID',
  `login_count` int(11) unsigned NOT NULL COMMENT 'friendID',
  PRIMARY KEY (`user_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='user';
```

```sum.go
type Sums struct {
  UserId string
  LoginCount int
}
```

- sumするにもテーブルの構造体が必要

```sums.go
s := new(Sums)
total, err := engine.SumInt(s, "login_count")
fmt.Println(total)
```

- totalに結果が入る

### 直接SQLを記述して実行

- 直接SQLを指定する事が可能
- `...FROM sums").Find(&us...` の `Find` を `Sum` 等に変える事で違う形で結果の取得が可能

```sql.go
users := make([]Users, 0)
err = engine.SQL("SELECT * FROM sums").Find(&users)
```

### update

- やり方が色々ある
- http://gobook.io/read/github.com/go-xorm/manual-en-US/chapter-06/index.html
- Where句で指定

```update.go
user := new(Users)
user.UserName = "UPDATE"
affected, err := engine.Where("user_id = ?", "test1").Update(user)
fmt.Println(affected)
```

### delete

- Where＆Deleteで削除できた。
- 論理削除（deletedAt）にも対応してるみたい http://gobook.io/read/github.com/go-xorm/manual-en-US/chapter-07/1.deleted.html

```delete.go
affected, err := engine.Where("user_id = ?", "test1").Delete(user)
```

{{< post-ads >}}

## 実際に使ってみて感じた事

### テーブル名指定の必要があり、書く量が多くなりそう

直接SQLを直接書く場合を除いて、`xorm` のクエリビルダを使う場合に、構造体とテーブル名を一致させなければならないルールがある。

※`TableNames` メソッドを実装すれば変更はできる

JOINを行う＆クエリビルダを使う場合、ほぼ必ず `TableNames` メソッドを実装しなければならないため、書く量が増えてしまう。 ※上記のJOIN部分を参照

`engine.Table()` の指定で、指定するテーブルを変更できるらしい（試してない

http://gobook.io/read/github.com/go-xorm/manual-en-US/chapter-02/3.tags.html

### 値と変数のマッピングについて

xormにはキャメルケースからスネークケースへのマッピング等を自動で行っている。
マッピングの種類は3種類くらいある。デフォルトでは `SnakeMapper` 。
`golint` で `userID` にすべきと警告を受けるが、実際に `userID` にして `SnakeMapper` を使
うと `user_i_d` に変換されてしまう。
xormを使うなら `userID` が `user_id` として変換される `GonicMapper` を使う事になりそう。

http://gobook.io/read/github.com/go-xorm/manual-en-US/chapter-02/1.mapping.html

xormでは、全ての命名をこのマッパーに合わせて行う事を推奨している。
もしこの命名規則に則らないテーブルが作成された時は `func TableName() string` を実装すれば回避可能。
カラムの場合は `xorm： "'column_name'"` で対応する。

### ドキュメントに頻出する `.Id()` について

- `.Id()` でプライマリキーを指定して更新できる機能が備わっている
- 下記で構造体のタグを`xorm:"user_id"` -> `xorm:"user_id pk"` としていて、 `"pk"` を足す事でこの機能を使える

```pk.go
type Users struct {
	UserId string `xorm:"user_id pk"`
	UserName string
	CreatedAt time.Time `xorm:"created"`
	CreatedUser string
	UpdatedAt time.Time `xorm:"updated"`
	UpdatedUser string
}
```

- これで `Id` が `test2` のユーザーの `user_name` が `UPDATE` に変更される

```pk.go
user := Users{UserName: "UPDATE"}
affected, err := engine.Id("test2").Cols("user_name").Update(&user)
```


## 実際に触ってみてまとめ

慣れれば問題無いかもしれないが、 `engine.Where(...)` で、その後に `Find`, `Delete`, `Update` を指定するのが個人的に受け付けなかった。
動詞が先だったらしっくりきてたかもしれない…。

Whereだけを使い回す事ってよくあるからやりたい事わかるっちゃわかるんだけど…。

Iterate処理はとても便利そうだった。
APIとかでDBから値とってレスポンス返却する時にほんの少し加工するだけのパターンって結構あると思うので、何かしらに使えそう。

ドキュメントは公式ウェブサイトに詳しく書かれていたので、もし使う時はここを参考にしたい。

http://xorm.io/docs/

