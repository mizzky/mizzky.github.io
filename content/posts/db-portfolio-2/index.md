+++
date = '2026-06-24T21:17:59+09:00'
draft = true
title = 'DB学習と実行計画検証（調査・学習編）'
+++

## 1. 実行計画の読み方とインデックスの理解

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
  - Bitmap Index Scanで作成したbitmapを元に対象のheapページを読み取る方式

## 2. B-treeインデックスと複合インデックスの理解

### B-treeインデックス
- PostgreSQLにおけるデフォルトのインデックス
- その他インデックス
  - `Hash` `GiST` `SP-GiST` `GIN` `BRIN` `bloom(拡張機能)`

- **順序付け出来る値**に対してB-treeインデックスは強く、オプティマイザもB-treeを使用しやすい

### 複合インデックス
- 複数列をまとめて１つのインデックスにしたもの

❌️　`user_id` `created_at`のインデックスが２つある

✅️　`user_id`でまず並べ、同じ`user_id`の中で`created_at DESC`で並ぶというインデックス
```sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
```
複合B-treeインデックスは「**左から効く**」と考える

## 3. LIMIT/OFFSETとKeyset Paginationの理解
### ページネーション
- ページネーションとは？
  - 大量のデータを一度に全て返さず、一定件数ごとに分割して取得する仕組み

```sql
SELECT *
FROM orders
ORDER BY created_at DESC
LIMIT 20
OFFSET 20;
```
- LIMIT: 取得する最大件数
- OFFSET: 先頭から何件を読み飛ばすかを指定

### OFFSETの問題点
- `OFFSET`では先頭から何件読み飛ばすかで位置を決めるため、ページ取得の間にデータ追加・削除されると重複・取得漏れが起きやすい
- 指定行数読み飛ばすが直接ジャンプするわけではないため、`OFFSET`が大きくなるとパフォーマンスに影響しやすくなる

### Keyset Pagination
- `OFFSET`の代わりに前ページの最後のデータを次ページの開始位置として指定する方式

- メリット
  - 先頭へのデータ追加によるページ間の重複やズレを避けやすい
  - 取得位置が深い場所でも計算コストが増えにくい

- デメリット
  - 特定のページ番号を指定して直接移動がしにくい
  - 前ページの末尾のキーをカーソルとして受け渡す必要があり、実装コストが上がる


## 4. N+1問題とJOIN / batch取得（IN句による一括取得）の理解
### N+1問題とは
- アプリ側のループ内でデータごとにSQLを発行することで、取得件数が増えるほど大量のSQLが実行されてパフォーマンスが低下する問題

```go
orders := getOrders(userID) // 1クエリ

for i := range orders {
    items := getOrderItems(orders[i].ID)
    orders[i].Items = items
}
```
上記のような１クエリに対してループでN回のクエリを実行するような場合、ループの要素（注文明細）が増えれば増えるほど子クエリも増え、SQL処理時間・アプリDB間の通信なども増大する

### JOINによる一括取得
```sql
-- name: ListCartItemsByUser :many
SELECT
    ci.id,
    ci.cart_id,
    ci.product_id,
    ci.quantity,
    ci.price,
    ci.created_at,
    ci.updated_at,
    p.name AS product_name,
    p.price AS product_price,
    p.stock_quantity AS product_stock
FROM cart_items ci
JOIN carts c ON ci.cart_id = c.id
JOIN products p ON p.id = ci.product_id
WHERE c.user_id = $1
ORDER BY ci.id;
```
親テーブルと子テーブルをJOINして１度にデータを取得し、アプリ側でデータを組み立てる。
```go
		rows, err := q.ListCartItemsByUser(c.Request.Context(), userID)
		if err != nil {
			_ = c.Error(apperror.NewInternalError("ListCartItemsByUser", err, apperror.InternalServerMessageCommon))
			return
		}

		items := make([]gin.H, 0, len(rows))
		for _, it := range rows {
			items = append(items, gin.H{
				"id":            it.ID,
				"cart_id":       it.CartID,
				"product_id":    it.ProductID,
				"quantity":      it.Quantity,
				"price":         it.Price,
				"created_at":    it.CreatedAt.Format(time.RFC3339),
				"updated_at":    it.UpdatedAt.Format(time.RFC3339),
				"product_name":  it.ProductName,
				"product_price": it.ProductPrice,
				"product_stock": it.ProductStock,
			})
		}
		c.JSON(http.StatusOK, gin.H{"items": items})
```

### IN句による一括取得
親テーブルと子テーブルをJOINするやり方以外に、子テーブルのデータをIN句でまとめて取得する方法もある。この場合、親テーブルの取得に１回、子テーブルの取得に１回の合計２回になる。
```sql
SELECT *
FROM order_items
WHERE order_id IN (101, 102);
```

```go
itemsByOrderID := map[int64][]OrderItem{}

for _, item := range items {
    itemsByOrderID[item.OrderID] = append(
        itemsByOrderID[item.OrderID],
        item,
    )
}

for i := range orders {
    orders[i].Items = itemsByOrderID[orders[i].ID]
}
```

### JOINとIN句の使い分け
JOINして１回で取得する方法と、IN句で取得する２クエリ方式の使い分けは以下のように考える

- JOINが向いているケース
  - データが１対１、多対１
  - レスポンスが平ら
  - 関連データの件数が少ない
  - 子テーブルの列でソートや絞り込みを行う場合
  - JOIN後の行数が極端に増えない

- IN句による一括取得が向いているケース
  - １対多
  - 親を先にページネーションしたい(LIMITによる絞り込み)
  - レスポンスが入れ子になっている
  - 親ごとの子件数が多い
  - 複数の１対多を同時取得し、JOIN結果の組み合わせが多い場合

## GoにおけるDB接続プールの理解

### type DB

type DB配下のイメージ図
```
type DB struct
  ├─ connector driver.Connector
  │    └─ 新しい driver.Conn を作る入口
  │
  ├─ freeConn []*driverConn
  │    ├─ driverConn
  │    │    └─ ci driver.Conn
  │    └─ driverConn
  │         └─ ci driver.Conn
  │
  ├─ numOpen
  │    └─ 開いている/開こうとしている接続数
  │
  └─ maxOpen / maxIdleCount
       └─ 接続数の上限設定
```

- 必要に応じてDBが`driver.Connector`に接続を作らせる。
- DBが複数の接続を接続プールとして管理する

DB接続のイメージ
```go
db, err := sql.Open("postgres", dsn)
```
このタイミングではpostgresという名前のドライバを探して`driver.Connector`を持った`*sql.DB`を作る処理。
実際に接続が作られるのは、DB操作のメソッド。メソッド実行時に接続の空きを確認し、無ければ新しい接続を作る。
```go
db.PingContext(ctx)
db.QueryContext(ctx, "SELECT ...")
db.ExecContext(ctx, "INSERT ...")
db.BeginTx(ctx, nil)
```

### Open
- 登録済みのドライバを探し、`*sql.DB`を作る
  - ドライバを型アサーションして`driver.DriverContext`を取得し、`driver.Connector`を作成する。
  - ドライバ自身がConnectorを作る仕組みを持っていない場合は`database/sql`が`dsnConnector`を作って補う
  ```go
    return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
  ```


### 学習した詳細
- [実行計画の読み方とインデックスの理解 #92](https://github.com/mizzky/sol/issues/92)
- [B-Treeインデックスと複合インデックス #93](https://github.com/mizzky/sol/issues/93)
- [LIMIT/OFFSETとKeyset Paginationの理解 #94](https://github.com/mizzky/sol/issues/94)
- [N+1問題とJOIN / batch取得（IN句による一括取得）の理解 #95](https://github.com/mizzky/sol/issues/95)
- [GoにおけるDB接続プールの理解 #97](https://github.com/mizzky/sol/issues/97)


### 主題Issue
[大量データ下のEC主要クエリ最適化と実行計画検証 #62](https://github.com/mizzky/sol/issues/62)