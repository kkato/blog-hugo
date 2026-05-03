---
title: "LSM-tree を理解する"
date: 2026-05-03T17:39:53+09:00
draft: true
tags: ["database"]
---

最近 Build Your Own Database From Scratch in Go という本を読んでいるのですが、LSM-tree まわりの説明がいまいち理解できなかったので、他の文献も参考にしつつ自分なりにまとめてみました。

B-tree / B+tree については[別の記事](/posts/btree-index/)にまとめたので、ここでは LSM-tree について書いていきます。

### LSM-tree

LSM-tree (Log-Structured Merge-tree) は書き込み性能を重視した構造で、Cassandra や RocksDB などで採用されています。

基本的なアイデアは「**書き込みはメモリに溜めてシーケンシャルに吐き出す**」というものです。ディスクへのランダム書き込みを避けるためにこのような設計になっています。

全体像は以下のとおりです。

```text
              ┌──→ WAL (ディスク)
   書き込み ──┤
              └──→ Memtable (メモリ)
                        │
                      flush
                        v
                   SSTable Lv0
                        │
                   compaction
                        v
                   SSTable Lv1
                        │
                   compaction
                        v
                   SSTable Lv2
```

#### Memtable

Memtable はメモリ上にあるソート済みのデータ構造（赤黒木やスキップリストで実装されることが多い）です。書き込みはまずここに入ります。

メモリなのでプロセスが落ちると消えてしまうため、書き込みと同時に WAL (Write-Ahead Log) にも追記しておき、復旧時に Memtable を再構築できるようにします。

#### SSTable

Memtable がある程度のサイズに達すると、ディスクに書き出されます。このディスク上のファイルを **SSTable (Sorted String Table)** と呼びます。

SSTable には以下の特徴があります。

- キーでソート済み
- 一度書いたら **イミュータブル**（後から書き換えない）
- 末尾にキーの位置を示すインデックスを持つことが多い

イミュータブルなのでロックなしで読めるのが LSM-tree の強みです。

```text
SSTable のイメージ
+--------------------+
| key=apple  val=1   |
| key=banana val=2   |
| key=cherry val=3   |
| key=durian val=4   |
+--------------------+
| index: apple→0行   |
|        cherry→2行  |
+--------------------+
```

#### Tombstone

LSM-tree では SSTable をその場で書き換えることができないので、削除するときも「削除した」というマーカーを書き込みます。これを **Tombstone（墓石）** と呼びます。

```text
時系列での書き込み:
  PUT key=foo val=1   → SSTable Lv0 に記録
  DELETE key=foo      → tombstone を記録
```

読み込み時には新しい SSTable から順に見ていき、tombstone を見つけたら「このキーは削除済み」と判断します。実際に古いデータが消えるのは後述の compaction のタイミングです。

#### Compaction

書き込みが続くと SSTable がどんどん増えていきます。読み込みのたびに大量の SSTable をスキャンするのは非効率なので、定期的に複数の SSTable をマージして1つにまとめる処理を行います。これを **compaction** と呼びます。

```text
   ┌─────────────────┐
   │ SSTable A       │
   │  foo=1          │───┐
   │  bar=2          │   │
   └─────────────────┘   │     ┌────────────┐     ┌─────────────────┐
                         ├────→│ compaction │────→│ SSTable C       │
   ┌─────────────────┐   │     └────────────┘     │  (merged)       │
   │ SSTable B       │   │                        │  bar=2          │
   │  foo=tombstone  │───┘                        │  baz=3          │
   │  baz=3          │                            └─────────────────┘
   └─────────────────┘
```

compaction では以下のことが行われます。

- 同じキーがあれば新しい方を残す
- tombstone があればそのキーを削除する
- 複数の SSTable を1つにまとめてサイズを抑える

compaction の戦略は LevelDB や RocksDB で採用されている **Leveled Compaction** や、Cassandra で使われる **Size-Tiered Compaction** など複数あります。

#### Bloom filter

LSM-tree で「キー `foo` を取得したい」と言われたとき、最悪すべての SSTable を見にいく必要があります。これを避けるために **Bloom filter** がよく併用されます。

Bloom filter は「**そのキーは絶対にここには無い**」を高速に判定するための確率的データ構造です。ハッシュ関数を複数使って、キーをビット列にマップします。

```text
Bloom filter (ビット配列):
  key=foo を h1, h2, h3 でハッシュ → bit[2], bit[5], bit[9] を 1 に
  key=bar を h1, h2, h3 でハッシュ → bit[1], bit[5], bit[7] を 1 に

  [0,1,1,0,0,1,0,1,0,1,...]

検索 key=baz:
  h1, h2, h3 → bit[3], bit[6], bit[8] を確認
  どれかが 0 → 「絶対に無い」と判定でき、SSTable を見にいかなくて済む
```

特徴は以下のとおりです。

- false negative はない（「ある」と言われたら本当にあるかは要確認、「無い」と言われたら本当に無い）
- false positive はあり得る
- ビット配列とハッシュ関数だけなのでメモリ効率がよい

各 SSTable に対応する Bloom filter を持っておくことで、無駄なディスク読み込みを大幅に減らせます。

### B+tree との比較

ざっくりまとめると以下のような特性の違いがあります。

| 観点 | B+tree | LSM-tree |
|------|--------|----------|
| 書き込み | ランダム書き込みになりやすい | シーケンシャル書き込み中心で高速 |
| 読み込み | 1回の探索で済むので速い | 複数 SSTable を見る可能性があり遅め（Bloom filter で緩和）|
| 範囲スキャン | リーフの連結リストで高速 | 各 SSTable をマージしながら読む |
| 削除 | その場で更新 | tombstone を書いて compaction で消す |
| 代表例 | PostgreSQL, MySQL | Cassandra, RocksDB |

書き込みヘビーなワークロードでは LSM-tree、読み込みや範囲スキャンが多いワークロードでは B+tree が向いています。ただ最近は両者のいいとこ取りを狙った実装（PostgreSQL の `pg_lsm` のような拡張）もあるので、適材適所で選ぶのがよさそうです。

### 参考

- [LSM ツリーについて学んだので整理する](https://zenn.dev/ksrnnb/articles/lsm-tree-summary)
- [Build Your Own Database From Scratch in Go](https://build-your-own.org/database/)
- [Database Internals](https://www.oreilly.com/library/view/database-internals/9781492040330/)
