---
title: "GORMの基本的な使い方"
date: 2026-01-17T22:22:44+09:00
draft: false
tags: ["go"]
---

最近、Goを勉強していてGORMを触る機会があったので、基本的な部分を調べてみました。

## モデル定義

GORMでは構造体をそのままテーブルにマップします。`gorm.Model`を埋め込むと`ID`や`CreatedAt`など便利なカラムが自動で付きます。

```go
package main

import "gorm.io/gorm"

type User struct {
    gorm.Model
    Name  string
    Email string `gorm:"uniqueIndex"`
    Age   uint
}
```

## DB接続とマイグレーション

PostgreSQLを例に、DBへ接続し、モデルに沿ってテーブルを自動生成します。`dsn`は環境に合わせて書き換えてください。

```go
package main

import (
    "log"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

func main() {
    dsn := "host=localhost user=postgres password=postgres dbname=app port=5432 sslmode=disable"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }

    // モデルに合わせてテーブルを作成・更新
    if err := db.AutoMigrate(&User{}); err != nil {
        log.Fatal(err)
    }
}
```

## CRUD操作

基本的なCRUD操作は以下のように行います。

```go
package main

import (
    "fmt"
    "log"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
)

type User struct {
    gorm.Model
    Name  string
    Email string `gorm:"uniqueIndex"`
    Age   uint
}

func main() {
    dsn := "host=localhost user=postgres password=postgres dbname=app port=5432 sslmode=disable"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal(err)
    }
    if err := db.AutoMigrate(&User{}); err != nil {
        log.Fatal(err)
    }

    // CREATE: レコードを追加
    alice := User{Name: "Alice", Email: "alice@example.com", Age: 28}
    if err := db.Create(&alice).Error; err != nil {
        log.Fatal(err)
    }

    // READ: 1件取得（主キーまたは条件指定）
    var user User
    if err := db.First(&user, "email = ?", "alice@example.com").Error; err != nil {
        log.Fatal(err)
    }
    fmt.Printf("found: %+v\n", user)

    // UPDATE: 1カラム更新（モデルを指定）
    if err := db.Model(&user).Update("Age", 29).Error; err != nil {
        log.Fatal(err)
    }

    // DELETE: ソフトデリート（DeletedAtにタイムスタンプが入る）
    if err := db.Delete(&user).Error; err != nil {
        log.Fatal(err)
    }
}
```

### よく使うクエリの書き方

```go
// 条件付きで複数取得
var users []User
db.Where("age >= ?", 25).Order("age desc").Limit(5).Find(&users)

// 複数カラムをまとめて更新
db.Model(&user).Updates(map[string]interface{}{
    "Name": "Alice Updated",
    "Age":  30,
})
```

## 参考

- [GORM Official Guides](https://gorm.io/docs/index.html)
- [GORM PostgreSQL Driver](https://gorm.io/docs/connecting_to_the_database.html#PostgreSQL)
