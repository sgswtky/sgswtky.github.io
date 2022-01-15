---
title: "AWS Cognito UserPool で reCAPTCHA + golang lambdaでカスタム認証"
date: 2021-02-21T10:00:00+09:00
draft: false
categories: ["AWS", "Cognito"]
tags: ["AWS", "Cognito", "ユーザー管理"]
description: awsのサービス Cognito というユーザー管理のためのプロダクトを Lambda(Golang) + PureJS + reCAPTCHA v3 で動かした時の記事です。Cognitoは無料枠も大きく非常に有用なプロダクトで、柔軟性の効く作りになっており非常に良いプロダクトでした。
---

Cognito は AWS で提供されてるユーザー管理のための機能群のようなもの。

アプリケーションを作成するとほぼ必ずと言って良いほどつきまとうのが **ユーザー管理とその周辺機能**。

Cognitoはこれをサポートするようなプロダクトで、サインアップ・サインインは標準で提供し、 Oauth2.0、SAML2.0の対応やSNS認証やメール認証といった機能も簡単に使える。

コレを実際に動かしてみます。

## 今回Cognitoを最低限使うための構成

- フロント: `PureJS`
- インフラ: `AWS Cognito`, `Lambda(Go)`, `Google reCAPTCHA v3`

Cognitoは `aws-cli`、`aws-sdk`で操作できる他、ユーザーとして Cognitoを操作するためのライブラリも存在している。`amplify` のライブラリ群にもその実装があり、今回はそちらを使用した。

`amplify`は各クライアント向けに下記のようなライブラリがある。
- https://github.com/aws-amplify/amplify-js
- https://github.com/aws-amplify/amplify-ios
- https://github.com/aws-amplify/amplify-android

### フロントのコードについて

普段jsは触らないためとりあえず動けばおｋで PureJSで作ってます。package.jsonで `npm run start` :8080で起動します。

npmのバージョンは `v14.15.5` で下記ファイル群を準備して立ち上げれば動くと思います。
（jsのライブリロードとかは面倒くさくてやってないです）

- https://gist.github.com/sgswtky/385b836ac5ad3ef04d0557199f725e8a

```console.sh
npm i
npm run start
```

#### 画面

これらのファイルから立ち上がるフロントの画面は下記画像のようになっており、 **サインアップ**、**サインイン**、**チャレンジでのサインイン** のフォームが存在します。

{{< figure src="/images/cognito-front-image.png" class="center" >}}

| 項目名 | 内容 |
| --- | --- |
| sign up | cognitoのユーザープールにユーザーを登録します |
| sign in | ユーザープールに登録されているユーザーでログインします |
| sign up challenge| sign in と同じ事を実現しますが、メールアドレスを用いたワンタイムパスワードで認証します。 |

### フロントに reCAPTCHA v3の実装済み

v3なので v2までとは違い、画像認証やロボットアクセスの確認等が無くなったバージョンの使用を想定しています。

reCAPTCHAはトークン作成時にドメインを指定するため、指定のドメイン以外で reCAPTCHAを組み込むとエラーになりますが、ローカルの `/etc/hosts` に `127.0.0.1` でドメインを追記して動作させていました。

※使用しない場合は該当コードを消して試してください。

### LambdaとCognitoの設定について

ブラウザから Cognitoを呼び出す事はできるので、次に Cognitoから呼び出す Lambdaを設定する必要があります。

下記の各それぞれのファイルを各々 `go build` して Lambdaファンクションの作成＆アップロードしておく。

※ 環境変数 `RECAPTCHA_SECRET` に reCAPTCHAのシークレットキーが必要なので Lambdaに設定しておいてください。

- [サインアップ前](https://gist.github.com/sgswtky/c0f83c91ccb66901cb52dfc6b202fb9f)
- [認証前](https://gist.github.com/sgswtky/5c877df2553f56d7d668bdea844d0a57)
- [認証チャレンジの定義](https://gist.github.com/sgswtky/d3bfee839856767b56f0b61151dedf5a)
- [認証チャレンジレスポンスの確認](https://gist.github.com/sgswtky/62e6ad7f66da8d89bc4738361949e8ca)
- [認証チャレンジの作成](https://gist.github.com/sgswtky/4e55a27b26959f88f17c4ce06730dc9a)

次に、登録した Lambdaファンクションを使用するユーザープールのトリガーからそれぞれ選んでいく。（上記テーブルのトリガー名を参考に設定）

マネジメントコンソールに下記のような画面があると思うのでそこから設定できる。

{{< figure src="/images/aws-cognito-trigger.png" class="center" >}}

### 実際にサインアップ・サインインを動かしてみる

ブラウザからサインアップ・サインインしてみると、CloudWatch Logsから Lambdaのログ出力がされている事が確認できると思う。

{{< figure src="/images/aws-cognito-pre_sign_up.png" class="center" >}}

サインアップ・サインインそれぞれのコード内 `localVerify` メソッドにて、 reCAPTCHAスコアが0.5未満の場合に認証をエラーにする処理を入れているので、もし動かない場合はその処理をコメントアウトして試すと良いかもしれません。

下記のような処理が入っています。

```example.go
func localVerify(recaptchaResponse *RecaptchaResponse) error {
	// TODO: write verify logic.
	errMsg := "recaptcha failed"
	if !recaptchaResponse.Success {
		return errors.New(errMsg)
	}
	if recaptchaResponse.Score < 0.5 {
		return errors.New(errMsg)
	}
	return nil
}
```

また、sign up 時のフロント側の処理で一部正しくない処理をしている箇所があるため注意が必要です。

`amplify-js` の `signUp` メソッドを使用すると `validationData` パラメータがうまく機能しません。通常であれば reCAPTCHAのトークンをここに入れてバリデーションするのですが、 `signUp`の処理だけ正常に送信されず `clientMetadata` を代用しています。

このバグは進捗として、プルリクはマージされているのですがリリースがされていないように見えました。 https://github.com/aws-amplify/amplify-js/issues/7655

現時点の最新版 `3.0.25` ではバグが含まれたままなので、もし使用する場合はこのメソッドを使わないようにする必要がありそうです。

### チャレンジ認証でサインインを動かしてみる

今回こちらの内容をミニマムで試しました。(https://docs.aws.amazon.com/ja_jp/cognito/latest/developerguide/user-pool-lambda-challenge.html)

独自の認証方法を定義するには `認証チャレンジの定義`, `認証チャレンジレスポンスの確認`, `認証チャレンジの作成` トリガーが必要になります。

ユーザー側から見ると以下のような処理となります。

1. メールアドレス入力 ==> Cognito側でチャレンジコード生成
2. チャレンジコード取得・入力 ==> Cognito側で認証 ==>  sign in 完了

Lambdaは、Cognito任意のタイミングで呼び出されるため、都度それらに応じたレスポンスを返す必要があります。

※実装してあるコードは、ログインする際にパスワードではなくメールに送信された4桁の数字（チャレンジコード）を入力する事でサインインさせるコードです。

※メール送信するコードは用意するのが面倒くさかったので `createAuthChallenge.go` の CloudWatch Logsに4桁のコードが出力されるので、メールが送信された想定で考えてください。

{{< post-ads >}}

## 実際動かしてみて、良さそうに思う点

- 料金が安い（MAU50,000ユーザーまで無料）
- スケールへの考慮が不要
- 認証の拡張性が柔軟

### 料金が安い（CognitoはMAU50,000ユーザーまで無料）

特に注目すべきは料金で、Cognito単体の料金は MAU50,000人以下は無料。それ以上なら人数に対して料金がかかる料金形態。驚くほど安い。

lambdaとの連携が可能なので自前のユーザーデータベースと繋ごうと思ったら一応可能です。（もしかすると二重管理的な感じになるので若干微妙かもしれないですが…）

実はこの自前のユーザーデータベースと繋ぐの、 **Auth0**（CognitoみたいなSaas）でやろうとすると `CustomConnection` という機能を使う必要があり、 Enterpriseプランでしか使えなかった気がします。なのでそちらと比べるととてもお得に感じます。

※ **Auth0** の Enterpriseの料金は非公開のため不明ですが、１つ下の DeveloperPro でも $130/mo かかります。

しかし、 **Auth0** の方が多機能でドキュメントが充実している感じが強いです。

### サーバーレスであるためスケールへの考慮が不要

Saasのような形で提供されており、インフラ管理はAWSとなるためスケールへの考慮が不要。（各種limitはそれはまあありますが…）

### Lambdaを用いて認証の拡張性が柔軟

拡張性については lambdaを用いて、各認証フローや処理へのカスタマイズが可能。

- サインアップ前
- 認証前
- カスタムメッセージ送信
- 認証後
- ユーザー確認後
- ユーザー移行時
- IDトークン生成前

これにより入れたいタイミングで入れたい処理が入れられる。

## 個人的に残念だと思う点

- コストメリットが低い(MAU多い場合に)
- 認証の拡張性は柔軟だけど、選択肢が lambda一択しか無い

### コストメリットが低い(MAU多い場合に)

真逆の事書いてるじゃん、となりますがMAU多いと料金結構上がります。

MAU多いと言っても 100万人くらいの話。例えば 100万人で日本円で約50万ほどかかるので、MAUは多いが売上がそこまで無いシステムの場合、コストメリットが低い。

100万人 =>  約50万円、2000万人 => 約600万円という形でMAUによって料金がどんどん上がっていくため、MAU数に比例して利益が生まれるシステムでないと採用は難しそうな印象。まあ管理を全部任せられるのでそのメリットが欲しい場合は良さそう。

### 認証の拡張性は柔軟だけど、選択肢が lambda一択しか無い

静的型付け言語を扱っている場合やり辛さがどうしても出てくる。API経由での操作可能にしてくれたら嬉しかった🤔（自分のコードが `interface{}` で受け取って処理してるからかもしれない…。リクエスト内容を構造体に定義しても良いのかも）

# 最後に

スタートアップ企業とかで超使えそうなイメージ。MAU5万人まで無料、Lambdaもほぼタダみたいなもんだし実質ほぼ無料でユーザー管理周りの機能が使えちゃう事になる。すごい…🤔

gistのコード貼り付けたら驚くほど長い投稿になったのでリンクにしてあります。逆に見にくくなってたらすみません…。

---

2021年最初の投稿。また気が向いたら更新します。

