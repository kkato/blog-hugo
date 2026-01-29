---
title: "kubectlで全てのリソースをgetする方法"
date: 2025-08-11T13:02:00+09:00
draft: false
tags: ["kubernetes", "command"]
---

`kubectl get all`だとCustom Resourceが表示されないので、全てのリソースが表示できずに困りました。`kubectl api-resources`を使えば、Custom Resourceも含めてサポートされているAPIリソースを全て把握できるので、それを応用して全てのリソースを表示できるようになりました。
備忘録として雑に書いておきます。

### 全てのリソースを表示する

以下のコマンドで、特定のnamespaceに存在する、全てのリソース(Custom Resourceも含めて)を表示することができます。

```bash
kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | grep -v "events" | tr '\n' ',' | sed 's/,$//') -n openebs
```

#### 各コマンドの説明

```bash
kubectl api-resources --verbs=list --namespaced -o name
```
- `kubectl api-resources`: クラスター内の全てのKubernetesリソース種別を表示
- `--verbs=list`: list操作が可能なものに絞る
- `--namespaced`: 名前空間に属するリソースに絞る (NamespaceやNodeなどのcluster-scopedリソースは除外)
- `-o name`: kubectlで直接使えるリソース名だけを出力

```bash
tr '\n' ','
```
- 出力されたリソース名を、改行区切りからカンマ区切りに変換

```bash
sed 's/,$//'
```
- 末尾に付いてしまった余分なカンマを削除
