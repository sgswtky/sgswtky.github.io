---
title: "Flutterが楽しい"
date: 2020-04-09T17:01:38+09:00
draft: false
categories: ["Flutter"]
tags: ["Flutter"]
description: flutterに入門してみた話。BLoCパターンなど独自の文化？書き方があってとても良かった。Flutter Webが正式リリースされたらもう一度触りたい。
---

## Flutterって何

https://flutter.dev/

2018年12月に正式リリースされた、Googleが開発するオープンソースのモバイルアプリ向けフレームワーク（iOS, Androidへクロスコンパイル可、もちろん実機でのデバッグも可）。
flutterで開発する際は Dart（https://dart.dev/）という言語で開発する。

web版は現時点ではベータ。

web版が正式にリリースされれば、iOS・Android・Webフロントエンドの全てが一度に開発できるようになる非常に協力なフレームワーク。

---

## 1.アプリ開発なのに敷居が低く感じた

Flutterのようなアプリ開発は関係者が多い（エディタ、ランナー、デバイス、SDK…など）のでトラブルが多いイメージがあったが、とてもスムーズに快活できて、敷居が低く感じられた。

---

### 充実したドキュメント（英語）

公式ドキュメント（英語）がすごく充実しています。

簡単なアプリを作るのであれば公式の手順に従って進めていけば誰でも作れるようになりそう、と思うくらいに簡単でした。

flutter Docs: https://flutter.dev/docs

---

### flutter doctorコマンドにより環境設定が簡単に

開発環境構築がとても簡単で、何ができて何ができていないのか `flutter doctor` コマンドですぐにわかる。

```console.sh
▶ flutter doctor
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, v1.12.13+hotfix.9, on Mac OS X 10.15 19A603, locale ja-JP)
[✗] Android toolchain - develop for Android devices
    ✗ Unable to locate Android SDK.
      Install Android Studio from: https://developer.android.com/studio/index.html
      On first launch it will assist you in installing the Android SDK components.
      (or visit https://flutter.dev/setup/#android-setup for detailed instructions).
      If the Android SDK has been installed to a custom location, set ANDROID_HOME to that location.
      You may also want to add it to your PATH environment variable.

[✓] Xcode - develop for iOS and macOS (Xcode 11.1)
[!] Android Studio (not installed)
[✓] IntelliJ IDEA Ultimate Edition (version 2019.1)
[!] VS Code (version 1.38.1)
    ✗ Flutter extension not installed; install from
      https://marketplace.visualstudio.com/items?itemName=Dart-Code.flutter
[!] Connected device
    ! No devices available
```

iOSの環境だけ整備してあるので、 Android系の環境が整っていない事がわかる。

かなり昔にアプリ開発に入門しようとした際、環境構築でうまくいかなくて辞めた自分にとっては非常にありがたいと感じました。

---

### widget（画面に配置するUIの部品）

flutterには、マテリアルデザインの widgetがプリセットのような形で予め定義されているので、それらを組み合わせるだけでアプリが簡単に完成する。

超大雑把に言うと、見た目の部品はできてるのであとは挙動をコーディングしたらアプリ完成。すごい。

それをベースに自分のアプリで使いたいようにカスタマイズしても良いし、containerウィジェットを使って、自分でウィジェットをイチから作っても良い。

widget一覧: https://flutter.dev/docs/development/ui/widgets

---

### cupertino widget(iOSライクなUI widget)

最初マテリアルデザインで開発してて、iOSライクな見た目じゃなくて気持ち悪いなあと思っていたのですが、このようにiOSライクな見た目のウィジェットも使えました。

cupertino: https://flutter.dev/docs/development/ui/widgets/cupertino

※Appleの本社がある場所がカルフォルニア州クパチーノにあるからそこから命名？

---

## 2. jsに割と似てる言語 `dart`

Flutterでコーディングする言語。JavaScriptっぽい。

クラス定義してエディタのサジェストが使える点ではJSよりも使いやすい印象です。（昔はできなかった）
Flutterで開発する際に使う **dart** は静的型付けに開発できます。（一応動的型付けとして開発する事も可能らしい）

---

### dartのnullの扱い: NNBD

現在dartはnull安全ではない言語ですが **non-null by default(NNBD)** として解決する方針のようです。（詳しく知りたい方はNNBDで検索）

null安全な言語になればnull考慮漏れが引き起こすバグも減るはずなのでこの辺りは期待しています。

---

### dartを触った感じ、少し関数型っぽく書けそう

Scalaを関数型で開発してる型にとっては少し物足りないかもしれませんが、最低限の便利なメソッド等は実装されている印象でした。

ジェネリクスがサポートされてるので、`List<String>` や `List<Int>` のような型もサポートされています。

また、`List` には `map, expand(flatMap) reduce, fold, forEach, where(filter)` のような高階関数も標準で存在しています。

`traverse, forAll` の実装や、 `Either, Option` あたりの型があったら嬉しかった。（traverse は、Listに限って言えば、unitまたはpure同等のモノが実装されてるようなものなので、自分で作ろうと思えば作れるとは思うのですが…）

もっと厳密に関数型として開発したい場合は、関数型のパッケージも開発されてるようでした。

dartz: https://github.com/spebbe/dartz

---

### dart + Flutterは相性が良い？

コードを書いてて、List<Widget> の各要素の間に **線のWidget(Divider)** を入れたい時にスマートなやり方がわからなかった。
この辺は Flutter と Dart の相性みたいな話かなと思った。

```
[WidgetA, WidgetB, WidgetC]
を
[WidgetA, Divider, WidgetB, Divider, WidgetC]
にするイメージ
```

処理の方法ですが、実際には色んなウィジェットを使ってるので簡略化してStringで表したものが👇です。

```main.dart
void main() {
    final divider = 'Divider';
    final widgets = ['WidgetA', 'WidgetB', 'WidgetC'];
    final r = widgets.expand((s) => [s, divider]).toList();
    r.removeLast();
    print(r);
}
```

Widgetの間に Widget(Divider) を入れたいというやりたい事はシンプルだけどわかりにくい処理を書かないといけないのがちょっと微妙に感じました。

```
List<T> addSeparate(List<T>: list, T: separate)
```

みたいなメソッドがList型に実装されてたら・Flutterのウィジェット用のユーティリティメソッドがあればなあと思いました。
ユーティリティメソッドみたいなの生やせばよくない？と言われればそれまでではあるのですが…😭

---

{{< post-ads >}}

## 3.ウィジェットの情報更新について

SPAのフロントをがっつり開発した経験が無い自分にとってはここが結構難しかった。しかし恐らくSPAのフロント等であればあるあるなのかもしれない。

詳しいメリット・デメリット等についてはそれぞれのやり方で調べてみてください。あくまで所感ベースで書いてます。

---

### 情報更新の方法①: setState

---

#### setStateの挙動

Flutterを始めて最初に取得するウィジェットの更新方法が恐らく **setState** だと思う。

例えばボタンのウィジェットでボタンが押された際に何か変更を加える場合

```main.dart
onPressed: () {
  setState(() => _count++);
}
// _countはStateのメンバ変数
```

このように書く必要がある。
`onPressed` の定義をクロージャで行い、その中で **setState** を呼ぶが、与える引数もクロージャ。

**setState** は簡単に実装できて、小規模であればデータ、ビジネスロジックがUIに与える影響が分かりやすいモノに感じた。

---

#### setStateで難しい点

分かりやすくはあるけど、Flutterを書いていて **setStateは規模が大きくなるにつれて難しい点がいくつかある** と感じた。

setStateを使ってコードを書いていくと
 - ロジックをメソッドで切り出す事は可能ではある
 - 結局最終的にステートのメンバ変数に依存（そこに代入する形）
 - メソッドを分離したとしても、一度async使ったら丁寧にまとめ上げていく必要がある（ロジック分散によるコードの肥大化）

という点から、いくらメソッドを綺麗に切り分けたとしても限界があると感じた。

例えば setState の中身が100行あった場合、とてもわかりにくくなってしまう。

```main.dart
onPressed: () {
  setState(() => {
    // 1...
    // 2...
    // ...
    // 99...
    // 100...
    _count++;
  });
}
// _countはStateのメンバ変数
```

さらにこの中で Futureを使ってたらそれを合成する必要も出てくる。

UIとビジネスロジックが蜜な関係になりすぎていて、コードが肥大化していくと **何がどうなったらUIがどう変化するのかわからない** という点が問題になりそうだと感じた。

---

### 情報更新の方法②: BLoCパターン

そこで、BLoCパターンというモノを試してみた。

---

#### BLoCパターンとは？

UIからビジネスロジックを切り離す事を目的とした手法。

正式名称は `Business Logic Component` らしい。

実際にある程度流れを理解してみて、Flutterを学習する上で鬼門になる、初級者と中級者の壁のようにも感じた。（言いすぎかもしれない）

---

#### 実際に動かしたコード

鬼門かも、とは言ったものの実装を見ると割とシンプルに感じると思います。

`flutter create` すると作成されるサンプルアプリのボタン押して表示されている数に `+1` してそれを表示する機能を BLoCパターン化してみる。

まずはウィジェットに埋もれてるステート（数字を管理してる部分）をBLoCとして切り出すイメージで別ファイルに分ける。

```plus_counter_bloc.dart
import 'dart:async';

class PlusCountBloc {
  StreamController<int> _PlusCountController =
      StreamController<int>.broadcast();

  StreamSink<int> get items => _PlusCountController.sink;

  Stream<int> get outStream => _PlusCountController.stream;

  var _count = 0;

  void plusCount(int plus) {
    _count = _count + plus;
    items.add(_count);
  }
}
```

※ちなみに、`flutter BLoC` でググると綺麗にまとめてくれてる人が沢山居るので、sink, streamについてはググってみてください。

次に、`_MyHomePageState` を以下のように書き換える。変更した部分にコメントを入れたので何をしてるかはそこを見てください。

```main.dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    // ※!
    final _plusCountBloc = PlusCountBloc();
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            // 数字表示部分のテキストを StreamBuilderに変更
            StreamBuilder<int>(
              stream: _plusCountBloc.outStream,
              initialData: 0,
              builder: (BuildContext context, AsyncSnapshot<int> snapshot) {
                return new Text(
                  '${snapshot.data}',
                  style: Theme.of(context).textTheme.display1,
                );
              },
            )
          ],
        ),
      ),
      // onPressedで、BLoCの plusCountを呼ぶ
      floatingActionButton: FloatingActionButton(
        onPressed: () => _plusCountBloc.plusCount(1),
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}
```

ステートを持つコードが完全に消えた点に注目して欲しい。

下記のコードが書かれていたが、BLoCに役割を移したので **UIはステートの管理という役割から開放された**。

```main.dart
// 消えたコード
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }
```

---

#### BLoCパターンで開発するメリット

- UIとロジックの分離によるコードの見通しがよくなる（vueっぽい）
- ロジック部分（BLoC）の一元管理（シングルトン化）による複数ウィジェットの更新（※1）
- クリーンアーキテクチャが適用できそう（多分）
- StatelessWidgetにできる（※2）

※1: ここでは書いてないけど、InheritedWidgetというものを使って、BLoCをシングルトン的に扱う事が可能。そのため、1BLoCにたいして複数ウィジェットと紐付けられる。載せたサンプルコードでは `※!` の部分で定義してるのでシングルトンではない。

InheritedWidget: (https://api.flutter.dev/flutter/widgets/InheritedWidget-class.html)

※2: BLoCを使うとウィジェットがステートを持たなくなる = ステートレスなウィジェットとなり、StatelessWidgetが使える。そのためクラス定義とシグネチャを次のようにしても動くようになる。

```main.dart
class MyHomePage extends StatelessWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  Widget build(BuildContext context) {
    /* StatefulWidgetと同じ中身 */
  }
}
```

StatelessWidgetは、StatefulWidgetのように値を変更する事ができない。全て final な値として宣言される。

他には、非常にザックリとした理解だけどウィジェットの更新タイミングの違い、あとはステータスを持たないので軽い。みたいな理解。

プロダクションレベルで開発する時はもっとこの辺の挙動や違いを把握しつつ開発する必要がありそう。

---

#### まとめ

BLoCパターンは実際に触ってみて、クリーンアーキテクチャ的にユースケース層に相当する処理の領域 + リアクティブプログラミングに感じた。

---

## 4.Flutter触ってみた感想

触ってる間はとりあえず楽しかった。

環境設定で大ゴケしたり、詰まって先に進めないエラーが大量に出るわけでもなく、比較的スムーズに開発する事ができた。

もし機会があればプロダクションレベルでの開発を経験してみたいと思った。

