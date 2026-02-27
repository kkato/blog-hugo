---
title: "kubectlで全てのリソースをgetする方法"
date: 2025-08-11T13:02:00+09:00
draft: false
tags: ["kubernetes", "command"]
---

`kubectl get all`だとCustom Resourceが表示されないので、全てのリソースが表示できずに困ることがありました。`kubectl api-resources`を使えば、Custom Resourceを含めた全てのリソースを表示することができます。

### 全てのリソースを表示する

以下のコマンドで、特定のnamespaceに存在する、全てのリソース(Custom Resourceも含めて)を表示することができます。

```bash
kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | grep -v "events" | tr '\n' ',' | sed 's/,$//') -n openebs
```

- `kubectl api-resources`: クラスター内のCustom Resourceを含む全てのKubernetesリソース名を表示
- `--verbs=list`: list操作が可能なものに絞る
- `--namespaced`: 名前空間に属するリソース名に絞る (NamespaceやNodeなどのcluster-scopedリソースは対象外)
- `-o name`: kubectlでgetできるリソース名だけを出力
- `grep -v "events"`: eventsリソースは大量に出力されてしまうため、除外する
- `tr '\n' ','`: 出力されたリソース名を、改行区切りからカンマ区切りに変換
- `sed 's/,$//'`: 末尾に付いてしまった余分なカンマを削除
