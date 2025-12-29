---
title: "Go 1.22のHTTPルーティング改善について"
date: 2025-12-29T12:56:44+09:00
draft: false
tags: ["go"]
---

自然言語でも日本語という母語があるように、プログラミングでも自分の母語と呼べる言語を持ちたいと考え、最近はGoを本腰入れて勉強しています。その中で HTTP サーバーを実装しようとした際、Gin・Echo・chi など多くの選択肢があり、どのフレームワーク（ライブラリ）を選ぶべきか迷いました。

そんな中、Go 1.22（2024年2月リリース）から標準ライブラリの`http.ServeMux`が強化されたことを知りました。
この改善により、多くのケースでサードパーティのライブラリが不要になっています。
この記事では、Go 1.22で追加されたServeMuxの新機能について、実例を交えながら解説します。

## これまでのServeMux（Go 1.21以前）の課題

Go 1.21以前のServeMuxには、以下のような制約がありました。

### HTTPメソッドを指定できない

```go
// Go 1.21以前: HTTPメソッドの区別ができない
mux := http.NewServeMux()
mux.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
    // GET、POST、PUT、DELETEなど、すべてのHTTPメソッドがこのハンドラに来る
    // 自分でメソッドを判定する必要があった
    switch r.Method {
    case http.MethodGet:
        // GETの処理
    case http.MethodPost:
        // POSTの処理
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
})
```

このように、メソッドごとに処理を分けたい場合は、自分でswitch文を書く必要がありました。

### パスパラメータを扱えない

```go
// Go 1.21以前: パスパラメータが使えない
// /users/123 のような動的なパスを扱うには、自分でパースが必要
mux.HandleFunc("/users/", func(w http.ResponseWriter, r *http.Request) {
    // URLからIDを手動で抽出する必要があった
    id := strings.TrimPrefix(r.URL.Path, "/users/")
    fmt.Fprintf(w, "User ID: %s", id)
})
```

### 完全一致が難しい

```go
// Go 1.21以前: /users にアクセスしたかったが、/users/ も /users/abc もマッチしてしまう
mux.HandleFunc("/users", handler)
// これは /users だけでなく /users/ にもマッチする
```

こうした課題から、多くのプロジェクトでサードパーティのルーターライブラリが使われていました。

## Go 1.22での新機能

Go 1.22では、これらの課題を解決する3つの大きな機能が追加されました。

### 1. HTTPメソッドの指定

パターンの先頭にHTTPメソッドを指定できるようになりました。

```go
mux := http.NewServeMux()

// GETリクエストのみを処理
mux.HandleFunc("GET /users", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ユーザー一覧を取得")
})

// POSTリクエストのみを処理
mux.HandleFunc("POST /users", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ユーザーを作成")
})
```

メソッドを指定しない場合は、すべてのHTTPメソッドにマッチします（従来の動作と同じ）。
指定したメソッド以外のリクエストが来ると、自動的に`405 Method Not Allowed`が返されます。

### 2. パスパラメータ（ワイルドカード）

`{name}`の形式でパスパラメータを定義できます。

```go
// 基本的なパスパラメータ
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
    // PathValueメソッドでパラメータを取得
    id := r.PathValue("id")
    fmt.Fprintf(w, "ユーザーID: %s", id)
})

// 複数のパスパラメータ
mux.HandleFunc("GET /posts/{postId}/comments/{commentId}", func(w http.ResponseWriter, r *http.Request) {
    postId := r.PathValue("postId")
    commentId := r.PathValue("commentId")
    fmt.Fprintf(w, "投稿ID: %s, コメントID: %s", postId, commentId)
})
```

残りのパスすべてにマッチさせたい場合は、`{name...}`の形式を使います。

```go
// ファイルパスなど、残りすべてのパスを取得
mux.HandleFunc("GET /files/{path...}", func(w http.ResponseWriter, r *http.Request) {
    path := r.PathValue("path")
    // /files/images/avatar.png → path は "images/avatar.png"
    fmt.Fprintf(w, "ファイルパス: %s", path)
})
```

### 3. 完全一致パターン

`{$}`を使うことで、そのパスに完全一致するリクエストのみをマッチさせることができます。

```go
// /users にのみマッチ（/users/ や /users/123 はマッチしない）
mux.HandleFunc("GET /users{$}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ユーザー一覧")
})

// ルートパス "/" にのみマッチ
mux.HandleFunc("GET /{$}", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ホームページ")
})
```

## 実践例

新しいServeMuxを使った簡単なREST APIの例です。

```go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    mux := http.NewServeMux()

    // ルートパス
    mux.HandleFunc("GET /{$}", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Welcome to the API")
    })

    // ユーザー一覧の取得
    mux.HandleFunc("GET /users{$}", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ユーザー一覧")
    })

    // 新しいユーザーの作成
    mux.HandleFunc("POST /users{$}", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ユーザーを作成しました")
    })

    // 特定のユーザーの取得
    mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        fmt.Fprintf(w, "ユーザー %s の情報", id)
    })

    // 特定のユーザーの更新
    mux.HandleFunc("PUT /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        fmt.Fprintf(w, "ユーザー %s を更新しました", id)
    })

    // 特定のユーザーの削除
    mux.HandleFunc("DELETE /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        fmt.Fprintf(w, "ユーザー %s を削除しました", id)
    })

    fmt.Println("Server starting on :8080")
    http.ListenAndServe(":8080", mux)
}
```

このコードを実行して、以下のようなリクエストを試してみましょう。

```bash
# ルートにアクセス
curl http://localhost:8080/

# ユーザー一覧を取得
curl http://localhost:8080/users

# 特定のユーザーを取得
curl http://localhost:8080/users/123

# ユーザーを作成（POSTメソッド）
curl -X POST http://localhost:8080/users

# 間違ったメソッドを使うと405エラーが返る
curl -X PUT http://localhost:8080/users
# => 405 Method Not Allowed
```

## まとめ

Go 1.22のServeMux改善により、以下のことが標準ライブラリだけで実現できるようになりました。

1. **HTTPメソッドの指定**: `GET /users`のような書き方で、メソッドごとにハンドラを分けられる
2. **パスパラメータ**: `/users/{id}`でURLから動的な値を取り出せる
3. **完全一致**: `/users{$}`で、完全一致するパスのみをマッチさせられる

これらの機能により、シンプルなREST APIであれば、サードパーティのルーターライブラリなしで十分に実装できるようになりました。

## 参考資料

- [Routing Enhancements for Go 1.22 - The Go Programming Language](https://go.dev/blog/routing-enhancements)
- [Better HTTP server routing in Go 1.22 - Eli Bendersky's website](https://eli.thegreenplace.net/2023/better-http-server-routing-in-go-122/)
- [The proposal to enhance Go's HTTP router](https://benhoyt.com/writings/go-servemux-enhancements/)
