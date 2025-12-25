# kkato.page

技術ブログのリポジトリです。

## 概要

個人の技術ブログサイトです。
Hugoを使用した静的サイトジェネレーターで構築され、GitHub Pagesでホスティングされています。

## 技術スタック

- **静的サイトジェネレーター**: [Hugo](https://gohugo.io/)
- **テーマ**: [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **ホスティング**: GitHub Pages
- **アナリティクス**: Google Analytics

## セットアップ

### 必要な環境

- Hugo (extended版推奨)
- Git

### ローカル開発

1. リポジトリをクローン:
```bash
git clone --recurse-submodules https://github.com/kkato/blog-hugo.git
cd blog-hugo
```

2. サブモジュールを初期化（クローン時に `--recurse-submodules` を指定しなかった場合）:
```bash
git submodule update --init --recursive
```

3. 開発サーバーを起動:
```bash
hugo server -D
```

4. ブラウザで http://localhost:1313 にアクセス

## コンテンツの追加

新しい記事を作成:
```bash
hugo new posts/記事名.md
```

記事は `content/posts/` ディレクトリに作成されます。

## ビルド

本番環境用のビルド:
```bash
hugo --minify
```

ビルドされたファイルは `public/` ディレクトリに出力されます。

## デプロイ

GitHub Actionsを使用して、mainブランチへのpush時に自動的にGitHub Pagesにデプロイされます。
