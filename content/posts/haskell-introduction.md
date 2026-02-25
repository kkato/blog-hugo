---
title: "Haskellに入門した"
date: 2026-02-25T06:00:00+09:00
draft: true
tags: ["haskell", "ghcup"]
---

Haskellは純粋関数型プログラミング言語で、強力な型システムと遅延評価が特徴です。
以前から気になっていたものの手を出せていなかったので、まずは環境構築から始めてみました。

### ghcupでHaskellの環境を構築する

Haskellのツールチェーンは[ghcup](https://www.haskell.org/ghcup/)で管理するのが一般的です。ghcupを使うとGHC（コンパイラ）、Cabal（ビルドツール）、Stack（プロジェクト管理ツール）をまとめてインストールできます。

```bash
curl --proto '=https' --tlsv1.2 -sSf https://get-ghcup.haskell.org | sh
```

対話形式のインストーラが起動するので、質問に答えていきます。自分は以下のように選択しました。

- チャンネル: **GHCup maintained**（デフォルト）
- プレリリースチャンネル: **No**
- クロスチャンネル: **No**
- PATHの追加: **Prepend**（`.zshrc` に自動追加）
- HLS（Language Server）: **No**（後から追加できるのでスキップ）
- StackとGHCupの統合: **Yes**（StackがGHCup管理のGHCを使うようになる）

インストールが完了すると以下のツールが使えるようになります。

| ツール | バージョン | 説明 |
|--------|-----------|------|
| GHC | 9.6.7 | Haskellコンパイラ |
| Cabal | 3.14.2.0 | ビルドツール |
| Stack | 3.7.1 | プロジェクト管理ツール |

ターミナルを再起動するか、以下のコマンドでPATHを反映させます。

```bash
. ~/.ghcup/env
```

### GHCiで動作確認

GHCi（対話型環境）を起動してHaskellが動くことを確認します。

```bash
ghci
```

```haskell
-- 基本的な演算
ghci> 1 + 2
3

-- リスト操作
ghci> map (*2) [1..5]
[2,4,6,8,10]

-- 関数定義
ghci> let factorial n = if n == 0 then 1 else n * factorial (n - 1)
ghci> factorial 10
3628800

-- 型の確認
ghci> :type map
map :: (a -> b) -> [a] -> [b]
```

`:type` で型を確認できるのが便利です。Haskellの型シグネチャは最初は読みにくいですが、慣れると関数の挙動が型から推測できるようになるらしいので楽しみです。

### Cabalでプロジェクトを作成する

実際にプロジェクトを作ってみます。

```bash
mkdir hello-haskell && cd hello-haskell
cabal init --non-interactive --simple
```

以下のようなファイルが生成されます。

```text
hello-haskell/
├── app/
│   └── Main.hs
├── hello-haskell.cabal
└── CHANGELOG.md
```

`app/Main.hs` を編集して簡単なプログラムを書いてみます。

```haskell
module Main where

-- フィボナッチ数列を無限リストで定義
fibs :: [Integer]
fibs = 0 : 1 : zipWith (+) fibs (tail fibs)

main :: IO ()
main = do
  putStrLn "フィボナッチ数列の最初の10個:"
  print (take 10 fibs)
```

遅延評価のおかげで無限リストを定義しても、`take` で必要な分だけ評価されます。これはHaskellならではの書き方で面白いです。

ビルドして実行します。

```bash
cabal run
```

```text
フィボナッチ数列の最初の10個:
[0,1,1,2,3,5,8,13,21,34]
```

### ghcupでツールチェーンを管理する

ghcupにはTUIがあり、GHCやCabalのバージョン管理ができます。

```bash
ghcup tui
```

特定のバージョンをインストール・切り替えることもできます。

```bash
# GHCのバージョン一覧
ghcup list -t ghc

# 別のバージョンをインストールして切り替え
ghcup install ghc 9.8.4
ghcup set ghc 9.8.4
```

### まとめ

ghcupのおかげでHaskellの環境構築はかなり簡単でした。GHCi、Cabal、Stackがすべて一つのインストーラで揃うのは便利です。
遅延評価や型クラスなど、他の言語にはない概念が多いので少しずつ学んでいきたいと思います。

### 参考

- [GHCup](https://www.haskell.org/ghcup/)
- [Learn You a Haskell for Great Good!](https://learnyouahaskell.github.io/)
- [Haskell Wiki](https://wiki.haskell.org/)
