+++
date = '2026-06-24T21:17:59+09:00'
draft = true
title = 'DB学習と実行計画検証（調査・学習編）'
+++

## 実行計画の読み方とインデックスの理解

### EXPLAIN
- 仮説を立てる、SQL実行はせず、どの程度のコストになるかを統計情報から推定している

### EXPLAIN ANALYZE
- クエリを実際に実行して、推定値だけではなく実測値も計測する

- `EXPLAIN ANALYZE`では実際にクエリを実行するので副作用が通常通りに発生する。テーブルを変更せずに分析したい場合はロールバックすること
```sql
BEGIN;

EXPLAIN ANALYZE
UPDATE tenk1
SET hundred = hundred + 1
WHERE unique1 < 100;

ROLLBACK;
```

### スキャンノード
- Seq Scan
  - テーブル本体を先頭から順番に全て読む方式
- Index Scan
  - インデックスを使って該当行を探してテーブル本体から取りに行く方式
- Bitmap Index Scan
  - インデックスを使ってBitmap Heap Scanがheapを読むためのbitmapを作る方式
- Bitmap Heap Scan
  - Bitmap Index Scanで作成したBitmap Heapを元に対象のheapページを読み取る方式


### 学習した詳細
[実行計画の読み方とインデックスの理解 #92](https://github.com/mizzky/sol/issues/92)




### 主題Issue
[大量データ下のEC主要クエリ最適化と実行計画検証 #62](https://github.com/mizzky/sol/issues/62)