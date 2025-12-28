---
title: "ghqとpecoを使った効率的なリポジトリ管理"
date: 2025-06-04T22:08:44+09:00
draft: false
tags: ["git", "command"]
---

他の記事でも多数紹介されていますが、最近`ghq`と`peco`を使ってみてとても便利だったので、自身のブログでも紹介します。

## ghqコマンド

`git clone`で多くのリポジトリをcloneしてると、ホームディレクトリ直下に大量のディレクトリで散らかり、どこに何があるかわからなくなってしまいます。
`ghq`は、Gitリポジトリをあらかじめ決めたディレクトリにまとめて管理できる便利なツールです。

ー https://github.com/x-motemen/ghq

### インストール方法

```bash
# macOSの場合
brew install ghq
```

### 基本的な使い方

```bash
# リポジトリをクローンする
ghq get username/repository

# ローカルのリポジトリ一覧を表示
ghq list
```

## pecoコマンド

`peco`は、CLI上でインタラクティブにテキストを絞り込み・検索できるツールです。

- https://github.com/peco/peco

### インストール方法

```bash
# macOSの場合
brew install peco
```

### 基本的な使い方

```bash
# ファイルを絞り込み選択
ls | peco

# 履歴から絞り込み選択
history | peco

# プロセスを絞り込み選択
ps aux | peco
```

## ghq + pecoの組み合わせ

この2ツールを組み合わせることで、リポジトリの切り替えが格段に楽になります。

### aliasの設定

以下のaliasを`~/.bashrc`~/.zshrc`に追加します：

```bash
# リポジトリを選択してcdする
alias ghqcd='cd "$(ghq list | peco)"'
```

## まとめ

`ghq`と`peco`を組み合わせることで、リポジトリの切り替えが格段に楽になりました。これはみんなやるべきだと思いました。
