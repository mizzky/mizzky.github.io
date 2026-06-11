+++
date = '2026-06-07T12:18:55+09:00'
draft = false
tags = ["log"]
title = 'slogを使った構造化ログ（調査・学習編）'
+++

slogを扱うにあたり、そもそもslogとはからエラー型の取り扱いなどわからないところがあったので調べたことをまとめる。

## Goのエラーラッピング


### errorsパッケージ

**errors.go**

- Newで構造体生成することで同一文字列でも別インスタンスになるので意図しない参照の衝突を防止してる
- Error()はそのerrorの文字列表現を返す
```go
func New(text string) error {
	return &errorString{text}
}
```

**wrap.go**
- wrap.goでエラーを包む概念を実装　
- Unwrap()を使ってエラーを内包したエラーを再起的にチェックして、Isではerrとtargetの比較を、asではerrの中にtargetがあるかどうかを検索している。

**fmt/errors.go**
- %wのフォーマッタがあるかどうかチェック
- %vはエラーを文字列としてフォーマットする、%wはエラー値を保持したラップエラーを返す


### errors.Isとerrors.Asの違い
- Isはerrからtargetと一致するerrorを探して判定する。
- Asはerrからtargetに代入できる型のerrorを探し、見つかった場合はその具体的なエラー値を後続処理に受け渡せる。


## slogの調査
### 公式プロポーザル
[A Proposal for Structured Logging (Go Blog)](https://go.dev/blog/slog)

#### slogのデザイン
- 使いやすさ：直感的に使えるAPIであること
- 高いパフォーマンス：ログ記録がプログラムのボトルネックとならないようアロケーションを最小限に抑える
- 既存エコシステムとの統合：既存の構造化ロギングパッケージやログ管理サービスと連携できる柔軟なバックエンドインターフェースを提供

#### 基本的な使い方
```go
slog.Info("hello, world", "user", os.Getenv("USER"))
```
各種ログレベル(`Debug`,`Info`,`Warn`,`Error`)に対応するメソッドを提供。第一引数がメッセージ、それ移行は任意のKey/Valueを受け取る

#### パフォーマンス配慮
- 強力な型付け
```go
slog.LogAttrs(ctx, slog.LevelInfo, "finished", slog.Int("count", 3))
```
- 通常の`slog.Info`のようなAPIは引数が`any`となるためボクシングによるアロケーションが発生する
- パフォーマンスが重要な場合には`slog.LogAttrs`を用いて型を明示することができる

#### log/slog パッケージを用いたJSON形式出力の設定例

```go
func NewJSONLogger(w io.Writer, level slog.Leveler) *slog.Logger {
	handler := slog.NewJSONHandler(w, &slog.HandlerOptions{
		Level:       level,
		ReplaceAttr: redaction.RedactAttr,
	})
	return slog.New(handler)
}
```
- `slog.NewJSONHandler`を使うことでJSON形式のログ出力を行うことができる




### 学習した詳細

[Goのエラーラッピング（Error Wrapping） #63](https://github.com/mizzky/sol/issues/63)

[構造化ログ (slog) の仕様調査 #64](https://github.com/mizzky/sol/issues/64)


### 主題Issue
[構造化ロギングとエラーハンドリングの再設計 #60](https://github.com/mizzky/sol/issues/60)
