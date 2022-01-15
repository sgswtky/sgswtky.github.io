---
title: "Jenkins Pipeline でGroovyを書いてたら期待しない動きをした"
date: 2018-07-17T20:14:21+09:00
draft: false
categories: ["Jenkins"]
tags: ["Jenkins", "Pipeline"]
description: Jenkinsを使ってた時の話。バージョンアップはとても大切。
---

先日から古い `Jenkins2` のジョブをメンテする仕事をしていて、Pipelineで Groovyを書いてたら特定のパターンでは動くけどそれ以外だと動かないという事情に遭遇した。

やりたかったことは、パイプラインAから、ジョブAを並列で2つ起動させたかった。
色々調べたけど結局わからなくて、バージョンアップしたら治った話。

## 実際に発生した事象について

### 記述したけど動かなかったコード

コードは極々単純で以下のようなコード。

```pipeline.groovy
node("master"){
  def tasks = [:]
  for (i = 0; i < 2; i++) {
    tasks[i.toString()] = {
        build job: "Job_A"
      }
  }
  parallel tasks
}
```

このコードを実行したら通常であれば ジョブA が２つ実行されるはず。しかし以下のように1つしか Job_Aが実行されなかった。

```jenkins.log
Started by timer
[Pipeline] node
Running on master in /var/lib/jenkins/workspace/Pipeline_A
[Pipeline] {
[Pipeline] parallel
[Pipeline] [0] { (Branch: 0)
[Pipeline] [1] { (Branch: 1)
[Pipeline] [0] stage
[Pipeline] [0] { (branch_2)
[Pipeline] [1] stage
[Pipeline] [1] { (branch_2)
[Pipeline] [0] build (Building Job_A)
[0] Scheduling project: Job_A
[Pipeline] [1] build (Building Job_A)
[1] Scheduling project: Job_A
[0] Starting building: Job_A #4
// ↑ここで Job_A が ビルド番号4で起動しているのがわかるが、起動しているのは #4 だけ。
[Pipeline] [0] }
[Pipeline] [1] }
[Pipeline] [0] // stage
[Pipeline] [1] // stage
[Pipeline] [0] }
[Pipeline] [1] }
[Pipeline] // parallel
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

{{< post-ads >}}

### 理由はわからないが何故か動くコード

以下のコードは期待する動きをした。

でもサーバー名を渡してるパラメータを廃止したかったため、 `params.size()` でなく、普通に forで回数分ループするコード（記事上部のコード）にしたかったが動かない。

```pipeline.groovy
node("master"){
  def params = ["server-01", "server-02"]
  def tasks = [:]

  for (i = 0; i < params.size(); i++) {
    def param = params[i]
    tasks[param] = {
      param: {
        build job: "Job_A", parameters: [new StringParameterValue("HOST", param)]
      }
    }
  }
  print(tasks)
  parallel tasks
}
```

## 解決方法

これの原因が全くわからなくて、バージョンアップした所正常に動くようになった。

## 動作バージョン情報について

バージョンアップの際に対象のプラグインだけでなく全てのプラグインのバージョンアップを行ってしまいました。

そのため下記に記載するプラグイン以外もバージョンアップしちゃってます。。。

| 対象 | 旧バージョン | 新バージョン |
|---|---|---|
| Jenkins | v2.30 | v2.132 |
| Pipeline（プラグイン） | 2.5 | 2.6 |

（Jenkinsのマイナーバージョンが100くらい上がってますね💦）

## まとめ

結構長い間（といっても合計3時間くらい）苦しんでたので解決できてよかった。
Jenkinsのバージョンアップだけは動かなくなる可能性がある（と勝手にそう思い込んでた）のでなるべくしたくなかったけど行って治ってよかった。

毎日定時でAMIをまるごとバックアップを取ってからか、バージョンアップ等のオペレーションの敷居がある程度低く、バックアップは大事だと再認識した。

