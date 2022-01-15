---
title: "Digdagの require オペレータで別のワークフローを複数回起動させる"
date: 2019-07-26T14:38:49+09:00
draft: false
categories: ["Digdag"]
tags: ["Workflow"]
description: Digdagというワークフローエンジンで別ワークフローを複数回起動させた話。
---

## digdag

https://www.digdag.io/

TreasureDataが開発するオープンソースのジョブスケジューラ

個人的に、Jenkinsと比較してコード管理・ログの扱いを厳重に管理できる好印象なプロダクト

---

このdigdagで、別ワークフローを複数回起動させるという事が業務で必要となり、検証を行いました。

通常、 `require` オペレータを使用すると `session_time` の関係で別のワークフローを複数回起動する事はできないけど、 `session_time` を動的に変更して無理矢理複数回起動させる。

`call` を使えば同じ事ができるけど、 `ignore_failure` を使いたかったので、 `require` で動くようにしてみた。

現時点で調べてみた感じ同様の内容は特に記事になってなかったので、せっかくなので書いてみる。

（`session_time` と `require` であえてできないようにしてる事を無理矢理やってる感じがあるので、もし本番に適用する場合は自己責任で…）

### requireを単純に使用するパターン

`parent.diglk`

```parent.dig
timezone: UTC

+setup:
  echo>: start ${session_time}

+pypy:
  py>: task.EXAMPLE.getParam

+repeat:
  for_each>:
    param: ${params}
  _do:
    require>: child
    params:
      param: ${param}
```

`child.dig`

```child.dig
timezone: UTC

+setup:
  echo>: ${moment(session_time).utc().format('YYYY-MM-DD HH:mm:ss Z')}

+child_job:
  sh>: sleep 5 && echo ${param}
```

`task/__init__.py`

```task/__init__.py
# coding=utf-8
import json
import digdag

class EXAMPLE(object):
  def getParam(self):
    list = ["e", "x", "a", "m", "p", "l", "e"]
    digdag.env.store({'params': json.dumps(list)})
```

requireで `child.dig` を呼び出すも、複数回呼ばれる事はありません。

WebUIで見るとこうなる。

![digdagのWebUI](/images/digdag-require-reuse-normal-require.png)

{{< post-ads >}}

### requireを理複数回起動させる

内容は比較的単純で、呼び出す `require` 全てに違う `session_time` を指定している。

`require` で `session_time` 指定せずに起動した場合、親の `session_time` を使うため、２度目の起動ができない。なのでループ毎に違う `session_time` を指定するだけで全てのループで `require` が起動する。

`reuse_parent.dig`

```reuse_parent.dig
timezone: UTC

+setup:
  echo>: start ${session_time}

+pypy:
  py>: task.EXAMPLE.getParamNum

+repeat:
  for_each>:
    param: ${params}
  _do:
    require>: reuse_child
    ## ここで加算して全てのループで違う値にする事で全て起動する
    session_time: ${moment(session_time).add(param.no, 's').format('YYYY-MM-DDTHH:mm:ssZ')}
    params:
      param: ${param.value}
```

↓こちらは特に変更無いです

`reuse_child.dig`

```reuse_child.dig
timezone: UTC

+setup:
  echo>: ${moment(session_time).utc().format('YYYY-MM-DD HH:mm:ss Z')}

+child_job:
  sh>: sleep 5 && echo ${param}
```

`task/__init__.py`

```task/__init__.py
# coding=utf-8
import json
import digdag

class EXAMPLE(object):
  def getParamNum(self):
    list = ["e", "x", "a", "m", "p", "l", "e"]
    result = []
    for i in range(len(list)):
      result.append({'no': i, 'value': list[i]})
    digdag.env.store({'params': json.dumps(result)})
```

結果、このように `require` を使用していますが順番に実行されます。

![digdagのWebUI](/images/digdag-require-reuse-require.png)

#### パラメータで付与している理由

現在時刻を使って加算する方法などもあると思うけど、うまく動作しなかった。

動的なパラメータを `moment` で加算するとうまく動かなかったので、 python側で生成し、digdagが扱う際には静的になってるパラメータを使う事で全ての `require` が動作した。

#### オススメしないパターン

なんとなく触っていて、 `子ワークフローを共有する` というのは避けたほうが良いと思いました。

子を複数の親が使うと、片親が使ってる時、もう片親も使い始めて `session_time` がかぶった時にどちらかがスキップされる挙動になると思います。

## requireオペレーターを使った時の挙動

普通に使った時の挙動などはこちらのQiitaに詳しくまとめられていました。ので、こちらなどを参考にすると良いと思います。

https://qiita.com/shiozaki/items/b3aaff926b7b601b4f86

## 最後に

書かなきゃと思ってたけどずっとサボってて 2019年初記事が7月下旬になってしまった。

今回のコードをこちらに上げてありますので機会がありましたらお試しください。

https://github.com/sgswtky/example-digdag-require-reuse
