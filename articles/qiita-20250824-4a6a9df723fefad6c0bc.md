---
title: "『初めてのGo言語 第2版』読書＆学習ログ（第1〜2章）"
emoji: "😀"
type: "tech"
topics: [Go,読書,技術書,初学者向け,学習記録]
published: false
---
# はじめに

入社1年目のエンジニアです。普段は業務系のフロントエンドや管理画面まわり（React/TypeScript・APEX など）に触れており、バックエンド基礎を固めるために Go を学び直しています。本記事は オライリー・ジャパン『初めてのGo言語 第2版』 を読み進めた 第1章・第2章 の学習メモです。

# 第1章メモ：ツールチェーン・フォーマッタ・セミコロン挿入
**go fmt**：標準フォーマットで差分を綺麗に

全パッケージに実行するなら：
```go
go fmt ./...
```


もしコミット後に整形漏れに気づいたら、ロジック変更と混在させずに フォーマットだけのコミットにするとレビューが楽。
```go
go fmt ./...
git add -A
git commit -m "chore: format only"
```

**go vet**：コンパイルは通る“怪しさ”を検出

構文エラーではないが「おそらく意図しない」パターンを指摘してくれる静的解析。

```go
go vet ./...
```

例：フォーマット指定子と引数型の不一致など。

ちょい実験：go vet が拾う典型例
```go
package main

import "fmt"

func main() {
	// ❌ フォーマット指定子と引数型の不一致（vet が指摘）
	// fmt.Printf("%d\n", "42")

	// ✅ 修正
	fmt.Printf("%d\n", 42)

	// ❌ Printf なのにフォーマット指定子なし（場合により指摘）
	// fmt.Printf("value", 100)

	// ✅ 明確にする
	fmt.Println("value", 100)
	// または
	// fmt.Printf("value %d\n", 100)
}
go vet ./...
```

### セミコロン自動挿入と { の位置

Go は改行やトークン位置から ; を自動挿入する。

改行直前が識別子・リテラル・break/continue などだと ; が入るため、func main() の直後に改行して { を置くとパースが崩れる。

```go
// ❌ ダメ（")" の直後で改行 → 暗黙の ; 挿入）
func main()
{
    fmt.Println("hello")
}

// ✅ 正しい（開き波カッコは同じ行）
func main() {
    fmt.Println("hello")
}
```

### Makefile で “いつもの手順” を自動化（例）

ビルド・整形・vet・テストなど、プロジェクト共通の手順を Makefile にまとめておくと再現性が上がる。

```Makefile
SHELL := /bin/bash
PKG   := ./...
BIN   := bin
APP   := app

.PHONY: init fmt vet test build run tidy clean 

init:
	@mkdir -p $(BIN)
	@go mod tidy

fmt:
	@go fmt $(PKG)

vet:
	@go vet $(PKG)

test:
	@go test $(PKG)

build: fmt vet
	@mkdir -p $(BIN)
	@go build -o $(BIN)/$(APP) .

run: build
	@./$(BIN)/$(APP)

tidy:
	@go mod tidy

clean:
	@rm -rf $(BIN)
```

# 第2章メモ：リテラル・定数・型変換の基本
### 整数リテラルの表記

基数の接頭辞：

0b...（2進） 0o...（8進） 0x...（16進）

桁区切りの _ が利用可能：
```go
n := 1_000_000 // 可読性向上
```
### 文字列リテラル：interpreted と raw

interpreted（解析あり）："。エスケープが解釈される。
```go
s := "hello\nworld"
```
```go
raw（ロー）：バッククォート `。そのままの文字列（複数行・正規表現・JSON 断片などに便利）。

raw := `C:\path\to\file
literally \n not escaped`
```
### 暗黙の型変換は行わない

型が違う値の演算や代入では 明示的な変換 が必要。
```go
var x int = 10
var y float64 = 1.23
z := float64(x) + y // ✅ 明示変換
```
### 「未型付けの定数」と「型付き定数」

Go の数値/文字/文字列/真偽のリテラルはしばしば 未型付け定数 として扱われ、使用文脈で型が決まる。
```go
const A = 10         // 未型付け定数
var f64 float64 = A  // ✅ 文脈から float64 に収まるため可
```

型付きの定数は他型の変数へは 直接代入できない（変換が必要）。
```go
const I int = 42
var f float64
// f = I        // ❌ NG
f = float64(I) // ✅ OK
```
### := は関数の外では使えない

短変数宣言は 関数スコープ内限定。
```go
// パッケージスコープでは var / const を使う
// msg := "hi" // ❌ NG

func main() {
    msg := "hi" // ✅ OK
    fmt.Println(msg)
}
```

# まとめ

- go fmt は毎回の儀式：標準フォーマットで差分を綺麗に保つ。

- go vet は“怪しさ”検出の相棒：コンパイルは通る地雷を早めに発見。

- セミコロン自動挿入の理解が重要：{ は同じ行に置く。

- リテラル表記は柔軟（0b/0o/0x, _ 区切り、raw 文字列）。

- 暗黙変換なし／未型付け定数の文脈決定を押さえる。

- 型付き定数 → 他型は明示変換が必要。

- := は関数内限定。パッケージスコープは var / const。



