---
title: "Argo CDをインストールする"
date: 2025-05-06T15:27:19+09:00
draft: true
---


https://argo-cd.readthedocs.io/en/stable/getting_started/

### Argo CDをインストールする

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Argo CD CLIをダウンロードする

```
brew install argocd
```

### API Serverにアクセスする

```
% kubectl port-forward svc/argocd-server -n argocd 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
```

```
argocd admin initial-password -n argocd
```


```
Unable to create application: application spec for recipe-api is invalid: InvalidSpecError: repository not accessible: repositories not accessible: &Repository{Repo: "https://github.com/kkato/argocd", Type: "", Name: "", Project: ""}: repo client error while testing repository: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.109.191.26:8081: i/o timeout"
```
