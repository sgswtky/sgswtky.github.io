---
title: "Goでポインターの扱いがわからない時に見たい内容。"
date: 2021-05-05T18:15:00+09:00
draft: false
categories: ["Go"]
tags: ["Golang", "pointer", "gomock", "Go言語", "ポインタ"]
description: Goのポインタ型を扱う時に自分が考えている内容をまとめた記事。ポインタの扱い方はわかったけど具体的にどうしていくのが良いのかをまとめた。
---

ポインターで何ができるかはわかったけど、つまりどう扱えば良いのかわからない時に見る内容にしたいきたいと思います。

色々意見を交わして考えが変わる事もあるので、その度に変更していけたらと思います。

---

Go言語触ってると頻出するポインター問題。

いろいろな人と開発してると、 **どこをポインターにして書いたらいいのかわからない** という話によくなる。

これは多分正解が無くて、メンバーのGoに対する成熟度やアーキテクチャによってどういう方針で作っていくかが結構変わってくると思う。

なので、 **正解はない** 前提だけど、個人的に **ここはコレで良い気がしてる** というのを書いていきたい。

---

{{< post-ads >}}

## プリミティブ型、プリミティブ型の配列をポインタでメソッド間で受け渡す場合は注意する

例えば下記の型（プリミティブ型はGoに元々にある型のようなもの）

```
*int
*string
*[]byte
[]*string
```

これらの型でreturnを書いている場合、 **本当にその型で返す必要があるのかという所に注意したい。**
例えば **使ってるライブラリがその型で返す** などの場合は仕方なくそうなるかもしれないけど、そうでない場合は**なんとなく違和感のあるアーキテクチャになってないか注意**する必要があると思う。

単純にポインタにしたほうが早そうだから、という理由でポインタで受け渡してると、後々にその値を参照する時に頻繁に元に戻す処理([nilチェック](https://play.golang.org/p/aEqF6sH_Cnj))を書く必要があるので、プリミティブ型の場合は気をつけてポインタとして扱う必要があると思う。

また、これは結構知られてる事だけど、`[]byte` をポインタとして扱う場合は `io.Reader` や `io.Writer` などのインターフェースとして扱う方が適切な場合が多い。

### ただし、`Response.Body` は少し気をつける

httpパッケージにある `Response.Body` の [Closeメソッドのコメント](https://golang.org/pkg/net/http/#Server.Close)の通り、Closeしておくのが正解のようなので気をつける必要がある。

```Google翻訳
// NOTE: Google翻訳
Closeは、すべてのアクティブなnet.Listenersと、StateNew、StateActive、またはStateIdleの状態にあるすべての接続をただちに閉じます。正常にシャットダウンするには、シャットダウンを使用します。

Closeは、WebSocketなどのハイジャックされた接続を閉じようとはしません（そしてそれについても知りません）。

Closeは、サーバーの基盤となるリスナーを閉じることで返されたエラーを返します。
```

そして、Closeする場合でも `[]byte, *[]byte, []*byte` で返すのはどれも正解でなく、下記のように `bytes.Buffer` 型で定義して `io.Reader` などで返すのが正解だと思う。

```example.go
func GetExample() (io.Reader, error) {
	resp, err := http.Get("http://example.com/")
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
	body, err := io.ReadAll(resp.Body)
	return bytes.NewBuffer(body), err
}
```

このように定義しておけば基本的にはポインタ型で受け渡ししてるのと同等のパフォーマンスが得られます。

## 構造体のポインタは使いまくるが良さそう

割と混乱しやすい所ではあるけど、 **構造体のポインタ** を使いまくるは良さそうに感じてます。

```example.go
type User struct {
  Status int
}
```

ここで言いたい構造体のポインタは、上記の構造体があったとすると、次のようになる。

```
// 使いまくるが良さそうな型
*User
[]*User

// そうでない型
*[]User // これは配列のポインタ型ですね。
```

### 何が良くて構造体はポインタが良いのか

- **構造体のポインタの配列** で扱えばループでも高パフォーマンス
- Golangは変数のデフォルト値が決まっているため、「構造体が無い」という状態が定義できる →開発してみるとわかるのですが地味に便利でわかりやすい…
- 構造体はメモリを使用する量が多くなりがちなので使ったほうがコスパ良

たとえば、下記のように構造体のポインタのメンバ変数を呼び出せるため構造体がnilでない限りpanicは発生しない（つまり1回だけnilチェックすればおｋ）

```example.go
type User struct {
	Status int
}

func Example() {
	u1 := &User{Status: 1}
	u2 := &User{Status: 2}

	// NOTE: 直接呼び出し可
	fmt.Println(u1.Status)
	fmt.Println(u2.Status)

	// NOTE: usersは、u1とu2のポインタ情報として配列を定義
	var users []*User = []*User{u1, u2}
	
	// NOTE: ループも対応
	for _, user := range users {
		fmt.Println(user.Status)
	}
}
```

なので、例えば Repositoryから構造体を取得するコードを書く場合に下記のようなインターフェースにしておけばメモリコピーを最小限に抑えられる＆値ひとつひとつのnilチェックと比較するとコード量が減る。

```user_repository.go
func (UserRepository) Find(userID string) (*User, error)
func (UserRepository) FindAll() ([]*User, error)
```

{{< post-ads >}}

## テスタビリティのためにメソッドの引数に気をつける

本来の処理を考えるあまりテストしやすいメソッドの設計になっていないのは割とあるあるだと思います。

ポインタの使用有無を統一せずに実装するとテスタビリティにも影響を及ぼす一例を元に紹介したいと思います。

### 前提

ユーザーのステータスを更新する `updateUserService` メソッドがあるとします（分類的にビジネスロジック、サービス、usecaseのような立ち位置のメソッドです）

このメソッドのテストコードについて考えていきます。

```example.go
func updateUserService(userID string, status int) error
```

このメソッドの内部で Repositoryが呼び出されます。これは次のような Interfaceとなっています。

テストではこちらをモック化します。

```interfaces.go
type UserRepositoryInterface interface {
	Find(userID string) (entity.User, error)
	Update(user *entity.User) error
}
type UserLogRepositoryInterface interface {
	UpdateLog(user *entity.User, log string) error
}
```

そして、`updateUserService` 本体は次のようになっています。

```main.go
func main() {
	// APIとかから値渡ってきたと過程
	if err := updateUserService("sgswtky", 1); err != nil {
		panic(err)
	}
}

// TODO: これをテストしたい
func updateUserService(userID string, status int) error {
	user, err := repository.UserRepository.Find(userID)
	if err != nil {
		return err
	}
	user.Status = status
	if err := repository.UserRepository.Update(&user); err != nil {
		return err
	}
	return repository.UserLogRepository.UpdateLog(&user, "UPDATE STATUS")
}

```

gomockを使えば、こちらの `updateUserService` の内部で呼び出されている `repository.*` をモック化する事ができます。(Interface化されてるので)

モック化する事により、想定呼び出し回数や想定される引数で呼ばれているかのチェックが可能となるため、コードの品質担保がしやすいという事が利点です。

実はこのコード、一見すると問題無いように見えますが、 **既に一部分テストできなくなっています。**

### gomockを使ったモック化のテスト

上記の `updateUserService` に正常系のパターンでテストを書くと次のようになります。

```user_test.go
func TestUpdateUserService(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	MockUserRepository := mock_repository.NewMockUserRepositoryInterface(ctrl)
	MockUserLogRepository := mock_repository.NewMockUserLogRepositoryInterface(ctrl)

	repository.UserRepository = MockUserRepository
	repository.UserLogRepository = MockUserLogRepository

	testUserID := "sgswtky"
	user := entity.User{0}
	MockUserRepository.EXPECT().Find(testUserID).Return(user, nil)
	MockUserRepository.EXPECT().Update(gomock.Any()).Return(nil)
	MockUserLogRepository.EXPECT().UpdateLog(&user, gomock.Any()).Return(nil)

	// 想定通りの処理であればエラーにはならないのでエラーの場合失敗
	if err := updateUserService(testUserID, 1); err != nil {
		t.Fail()
	}
}
```

なんの変哲も無いテストで特に問題は無さそうで、テストもPASSします。

しかし、本当にテストコードとして問題が無いのかチェックするために、 `updateUserService` の `user.Status = status` をコメントアウトしてテストを実行してみるとPASSします。

**つまりこのテストコードは正常系のテストコードとしては不十分である事がわかります。**

本来であればステータスが更新された事もテストされている必要があるのですが、漏れています。
**恐らくコードレビューでも漏れがちな考慮漏れ**だと思います。

### ではどうすれば良いのか

コードを見ると、`updateUserService` 内でステータスを更新し、それを `UserRepository.Update` に渡している事がわかります。一部抜粋するとこの部分です。

```main.go
...
	user.Status = status
	if err := repository.UserRepository.Update(&user); err != nil {
		return err
	}
...
```

つまり、`UserRepository.Update` に更新された値が渡されている事をモックが検知する必要がありますが、現在このメソッドのモックは次のように定義しています。

```example.go
MockUserRepository.EXPECT().Update(gomock.Any()).Return(nil)
```

`gomock.Any()` を使っているため、**「引数は何渡されても問題無し、戻り値は常に nil」** とという挙動をします。

この部分が問題であるため修正する必要がありますが、ここで今一度 `UserRepository` の interfaceを思い出してみます。

```interface.go
type UserRepositoryInterface interface {
	Find(userID string) (entity.User, error)
	Update(user *entity.User) error
}
```

`UserRepository.Update` メソッドの引数は `user *entity.User` 型です。

構造体はポインタ使った方が良いと書いたのですが、ここではポインタである事によって弊害が出ているように見えると思います。

しかしもう少し詳しく見ると、ポインタである弊害があるというよりは、 **ポインタと値どちらも使用してしまっているために起こっている問題** である事に気が付きます。

`updateUserService` メソッドでは、`Find` で取得した `User型` を変更して、 `Update` に渡しています。この `Find` はポインタでない `entity.User` を返します。

つまり、下記のどちらかで正しくテストできるようになります。

- `Find` が `*entity.User` を返し、変更された値をチェック
- `Update` が `entity.User` を受け取るように変更し、モックの引数に想定される値を入れる

今回は全体的にポインタ型で書いていくとどうなるかも見たいので、前者で進めます。

### 正しくテストできるコード

今回は全ての `entity.User` を `*entity.User` にして、正しくテストできるように変更します。

Interface

```interface.go
type UserRepositoryInterface interface {
	Find(userID string) (*entity.User, error)
	Update(user *entity.User) error
}
type UserLogRepositoryInterface interface {
	UpdateLog(user *entity.User, log string) error
}
```

`updateUserService`

```user.go
func main() {
	// APIとかから値渡ってきたと過程
	if err := updateUserService("sgswtky", 1); err != nil {
		panic(err)
	}
}

func updateUserService(userID string, status int) error {
	user, err := repository.UserRepository.Find(userID)
	if err != nil {
		return err
	}
	user.Status = status
	if err := repository.UserRepository.Update(user); err != nil {
		return err
	}
	return repository.UserLogRepository.UpdateLog(user, "UPDATE STATUS")
}
```

ここまではほとんど変更がありませんが、テストコードでは若干変更があります。

```user_test.go
func TestUpdateUserService(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	MockUserRepository := mock_repository.NewMockUserRepositoryInterface(ctrl)
	MockUserLogRepository := mock_repository.NewMockUserLogRepositoryInterface(ctrl)

	repository.UserRepository = MockUserRepository
	repository.UserLogRepository = MockUserLogRepository

	testUserID := "sgswtky"
	user := &entity.User{0}
	MockUserRepository.EXPECT().Find(testUserID).Return(user, nil)
	MockUserRepository.EXPECT().Update(user).Return(nil)
	MockUserLogRepository.EXPECT().UpdateLog(user, gomock.Any()).Return(nil)

	// 想定通りの処理であればエラーにはならないのでエラーの場合失敗
	if err := updateUserService(testUserID, 1); err != nil {
		t.Fail()
	}
	// 変更されてるかチェック
	if user.Status != 1 {
		t.Fail()
	}
}
```

テストコードで、`Find` から返る `user` を `Update` でも使用しています。`user` の定義は `user := &entity.User{0}` となっており、変数はポインタ型です。

これにより `gomock.Any()` は回避でき、より詳細にモックを定義できた事になります。
最後に、値が変更された事の確認として、`user.Status` のチェックも追記しました。

### 結論

このように、ポインタをテストで使う時には気をつけて開発する必要がありますが、下記を方針として取り入れて開発する事である程度排除できる気がします。

- 構造体は基本的にポインタとして扱う方針
- `gomock.Any()` は便利だけど本当に使うべきか今一度考えてから使うが良さそう

## まとめ

- プリミティブ型、プリミティブ型の配列をポインタでメソッド間で受け渡す時は臭うコードになってないか確認
  - 回避はインターフェースで
  - http.Response.Body は気をつける
- 構造体はポインタで使っていく
- テストの時の `gomock.Any()` を安易に使わないように今一度考える

結構初歩的な内容や「思想強」と思われるような内容になりました。何か意見やマサカリあればDMください🙇 


