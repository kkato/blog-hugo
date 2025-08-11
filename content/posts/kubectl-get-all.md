---
title: "kubectlで全てのリソースをgetする方法"
date: 2025-08-11T13:02:00+09:00
draft: false
tags: ["kubernetes", "command"]
---

`kubectl get all`だとCustom Resourceが表示されないので、全てのリソースが把握できずに困りました。`kubectl api-resources`コマンドを使えば、Custom Resourceも含めてサポートされているAPIリソースが全て把握できるので、それを応用して全てのリソースを表示できるようようにしました。\
備忘録として雑に書いておきます。

## 全てのリソースを表示するコマンド

以下のコマンドで、特定のnamespaceに存在する、全てのリソース(Custom Resourceも含めて)を表示することができます。

```
kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | tr '\n' ',' | sed 's/,$//') -n <namespace>
```

### 各コマンドの説明

```
kubectl api-resources --verbs=list --namespaced -o name
```
- `kubectl api-resources`: クラスター内の全てのKuberentesリソース種別を表示
- `--verbs=list`: list操作が可能なものに絞る
- `--namespaced`L 名前空間に属するリソースに絞る (NamespaceやNodeなどのcluster-scopedなりソースは除外)
- `-o name`: kubectlで直接使えるリソース名だけを出力

```
tr `\n` `,`
```

- 出力されたリソース名を、開業区切りからカンマ区切りに変換

```
sed 's/,$//'
```

- 末尾に着いてしまった余分なカンマを削除
