+++
date = '2026-06-12T10:25:44+09:00'
draft = true
title = 'DB学習と実行計画検証（計画編）'
+++

## 全体計画
- 大量データ下で、実行計画を読んで根拠を持ってDB設計を判断できるようになる
- DBを速くすることが最優先ではない

### Main Issue
主題となるIssueは以下
```markdown
## 1. 概要 (Background & Objectives)

現状のECアプリケーションでは、商品一覧、カート取得、注文作成、注文一覧取得などの主要機能が `database/sql` + `sqlc` を用いて実装されている。一方で、現在のデータ量は小さく、100万件規模の注文・注文明細・商品データが投入された場合に、既存クエリやインデックス設計がどのように劣化するかは検証できていない。

本Issueでは、大量データ下におけるEC主要クエリの実行計画を `EXPLAIN ANALYZE` で確認し、インデックス設計、N+1問題、ページネーション、DB接続プール設定を検証する。単なるCRUD実装ではなく、PostgreSQLの実行計画と計測結果に基づいて、パフォーマンス改善の根拠を説明できる状態を目指す。

## 2. 学習・調査タスク (Learning & Research)

- [ ] **B-Treeインデックスと複合インデックスの理解**
    - B-Treeインデックスが検索・ソートに効く理由を説明できる。
    - 複合インデックスにおけるカラム順序が、検索条件と `ORDER BY` に与える影響を説明できる。
    - 既存の `orders(user_id, created_at DESC)` などのインデックスが、どのクエリに効くかを整理する。

- [ ] **PostgreSQLの実行計画の読み方**
    - `EXPLAIN ANALYZE` の `cost`, `actual time`, `rows`, `loops` を読めるようにする。
    - `Seq Scan`, `Index Scan`, `Bitmap Index Scan`, `Bitmap Heap Scan` の違いを説明できる。
    - インデックスが存在しても使われないケースを理解する。

- [ ] **大量データ下のページネーション設計**
    - `LIMIT/OFFSET` がデータ増加時に遅くなりやすい理由を説明できる。
    - keyset pagination の考え方と、ECの注文一覧に適用する場合の条件を整理する。
    - 既存APIに導入する場合の互換性・実装コストを検討する。

- [ ] **N+1問題とJOIN / batch取得の理解**
    - ループ内で個別クエリを発行する実装が、データ件数増加時にどのように劣化するかを説明できる。
    - JOINまたはbatch取得でクエリ回数を削減する設計を比較できる。
    - 注文一覧・注文明細取得を題材に、N+1の発生条件を整理する。

- [ ] **GoにおけるDB接続プールの理解**
    - `sql.DB` が接続そのものではなく接続プールであることを説明できる。
    - `SetMaxOpenConns`, `SetMaxIdleConns`, `SetConnMaxLifetime` の役割を整理する。
    - 大量リクエスト時に接続数・待ち時間・DB負荷へ与える影響を理解する。

## 3. 実装タスク (Implementation)

- [ ] **大量データ投入スクリプトの作成**
    - 100万件規模の注文データと、それに紐づく注文明細データを生成できるようにする。
    - `users`, `products`, `orders`, `order_items`, `cart_items` の整合性を保つ。
    - 再実行可能で、検証用DBを壊しにくい投入手順を用意する。

- [ ] **計測対象クエリの選定**
    - 商品一覧取得
    - ユーザー別注文一覧取得
    - 注文明細取得
    - カート取得
    - 必要に応じて管理者向け検索・集計クエリ

- [ ] **EXPLAIN ANALYZE取得手順の整備**
    - before / after の実行計画を保存できる形式を決める。
    - 計測条件（データ件数、実行SQL、インデックス有無）を記録する。
    - 実行計画をGitHub Pages記事から参照できる形に整理する。

- [ ] **インデックス改善の実装**
    - 実行計画を確認したうえで、必要なインデックスを追加する。
    - 追加したインデックスごとに、対象クエリ・期待効果・副作用を記録する。
    - 不要または効果の薄いインデックスを追加しない判断も記録する。

- [ ] **N+1比較用の検証コード作成**
    - 意図的なN+1版と、JOINまたはbatch取得版を比較できるようにする。
    - クエリ回数と実行時間の差を測定する。
    - 本番コードに残すべき実装と、検証専用コードを切り分ける。

- [ ] **DB接続プール設定の追加**
    - `sql.DB` に接続プール設定を追加する。
    - 設定値の根拠をコメントまたはドキュメントに残す。
    - 過剰な接続数でDBに負荷をかけない方針を整理する。

## 4. 検証タスク (Verification)

- [ ] **商品一覧クエリの実行計画確認**
    - 大量商品データ下で、既存クエリの実行計画を確認する。
    - インデックス追加前後で `actual time`, `rows`, scan type がどう変化したか記録する。

- [ ] **ユーザー別注文一覧クエリの実行計画確認**
    - `orders.user_id` と `created_at DESC` を使うクエリの実行計画を確認する。
    - 複合インデックスのカラム順序が効いているか確認する。
    - `LIMIT/OFFSET` と keyset pagination の比較を行う。

- [ ] **注文明細取得クエリの実行計画確認**
    - `order_items.order_id` を使う取得処理の実行計画を確認する。
    - 注文詳細表示に必要なJOINの有無とインデックス効果を確認する。

- [ ] **N+1問題の計測**
    - N+1版とJOIN / batch取得版で、クエリ回数と実行時間を比較する。
    - データ件数が増えたときに差が拡大することを確認する。
    - 実装として採用すべき方針を説明できるようにする。

- [ ] **DB接続プールの簡易負荷確認**
    - 接続プール設定前後で、大量リクエスト時の挙動を比較する。
    - `MaxOpenConns` を増やしすぎた場合のリスクを整理する。
    - アプリケーション側とDB側のどちらがボトルネックになるかを確認する。

## 5. 完了定義 (Definition of Done)

- [ ] 3つ以上の主要クエリについて、`EXPLAIN ANALYZE` のbefore / afterを記録している。
- [ ] 追加したインデックスの根拠を、実行計画に基づいて説明できる。
- [ ] 複合インデックスのカラム順序を、自分の言葉で説明できる。
- [ ] N+1版とJOIN / batch取得版の差を、クエリ回数と実行時間に基づいて説明できる。
- [ ] `LIMIT/OFFSET` と keyset pagination の違いを説明できる。
- [ ] `sql.DB` の接続プール設定の採用理由を説明できる。
- [ ] 大量データ下のDB設計ガイドラインがGitHub Pagesに整理され、本Issueから辿れる。
- [ ] `go test ./... -count=1` がPASSする。

## 6. スコープ外 (Out of Scope)

- 分散トランザクション / Sagaパターン
- DBシャーディング
- レプリケーション構成
- Redis等の外部キャッシュ導入
- 本格的な負荷試験基盤の構築
- フロントエンドのページングUI改善
- ORM比較(sqlx / GORM等)
- 本番監視基盤の構築

## 7. 参考資料 (References)

- PostgreSQL Documentation - Indexes: https://www.postgresql.org/docs/current/indexes.html
- PostgreSQL Documentation - EXPLAIN: https://www.postgresql.org/docs/current/sql-explain.html
- PostgreSQL Documentation - Using EXPLAIN: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL Documentation - Indexes and ORDER BY: https://www.postgresql.org/docs/current/indexes-ordering.html
- Go `database/sql` package: https://pkg.go.dev/database/sql
- Go Managing database connections: https://go.dev/doc/database/manage-connections
```

## 現状
- DBに関する知識が不足している、基礎的なCRUD、抽出SQL程度の理解しかない
- EXPLAINEなどの実行計画、インデックスも理解が足りない
- バックエンドの接続で決められたカラムを読み書きすることしかできない状況を改善するため、データベースの仕組みに触れつつ学習を進める

## 進め方

ドラフト
```
DBがどう動いているか？
↓
なぜDBは遅くなるのか？
↓
実行計画を読めるようにする
↓
SQL/API設計をどうするべきなのか？
↓
GoからDBをどう扱うか？
↓
リポジトリに当てはめて考えてみる
↓
改善を設計する

```

上記のざっくりした流れの中で、学習フェーズについては以下のようなタスクがある
1. DBがどう読むか
   - スキャン方式
   - B-Tree
   - 複合インデックス
   - EXPLAIN ANALYZE

2. SQL/API設計がどう劣化するか
   - LIMIT/OFFSET
   - keyset pagination
   - N+1
   - JOIN / batch取得

3. GoアプリからDBをどう扱うか
   - sql.DB
   - connection pool
   - 大量リクエスト時の待ち

これらの知識をつけて理論面の学習ができた段階で、実際にDB操作を行い手を動かすことで理解を定着させることを狙う。