---
title: "Go 標準ライブラリだけで Clean Architecture — sf-meter-api の5層構成と、メモリ→PostgreSQL の差し替え"
emoji: "🏗️"
type: "tech"
topics: ["go", "cleanarchitecture", "restapi", "postgresql", "test"]
published: false
---

## はじめに

「Clean Architecture の本は読んだ。でも Go の小さなプロジェクトにどう落とし込めばいいのか分からない」

この記事では、自分の学習用リポジトリ **sf-meter-api**（[github.com/nemdull/sf-meter-api](https://github.com/nemdull/sf-meter-api)）を題材に、Go でクリーンアーキテクチャを「最小構成で」実践したときの判断を解説します。

sf-meter-api は、SF 作品に登場する架空のテクノロジー（ドミネーター、ニューラルリンク、コールドスリープ…）に **「現実での実現可能性スコア（0〜100）」** を付けて返す、ごく小さな REST API です。エンドポイントは `GET /techs` と `GET /techs/{id}` の 2 本だけ。題材が軽いぶん、レイヤーの役割がはっきり見えます。

技術的な特徴は次の 3 つです。

- Web フレームワークを使わず、**Go 1.22 以降の `net/http` 標準 ServeMux**（`"GET /techs/{id}"` のパスパターン）だけで書いている
- 外部依存は PostgreSQL ドライバの `github.com/lib/pq` のみ
- 環境変数 `DATABASE_URL` があれば PostgreSQL、無ければメモリ実装で動く。**この差し替えが、依存の方向を守った結果として「自然に」できる**ことがこの記事の主題

対象読者は「Go の基本文法は書けるが、設計パターンはこれから」という方です。

## 依存の矢印はいつも内側を向く

Clean Architecture の本質はこの一行に集約されます。

```
presentation ──▶ usecase ──▶ domain ◀── infrastructure
                                  ▲
                      registry（配線だけを知っている）
```

`domain` は DB も HTTP も知りません。`infrastructure` が `domain`／`usecase` の定義したインターフェースを**実装する**ことで、依存の向きが逆転します。`registry` だけが「どの実装をどこに差すか」を知っています。

## ディレクトリ構成（実物）

```
sf-meter-api/
├── main.go                        # 起動・ルーティングだけ
├── domain/
│   └── tech.go                    # エンティティ Tech と、その不変条件
├── usecase/
│   ├── ports.go                   # TechRepository（usecase が必要とする抽象）
│   ├── service.go                 # TechService
│   └── tech_service_test.go
├── presentation/
│   ├── handler.go                 # HTTP ハンドラー（TechUseCase を自分で定義）
│   ├── dto.go                     # 外に出す形（DTO）
│   └── handler_test.go
├── infrastructure/
│   ├── memory.go                  # メモリ実装（開発・テスト用）
│   └── postgres.go                # PostgreSQL 実装
├── registry/
│   └── registry.go                # DI の配線。DATABASE_URL で実装を選ぶ
├── db/init.sql                    # テーブル定義とシードデータ
├── docker-compose.yml             # PostgreSQL 16 + app
└── Makefile
```

`internal/` ディレクトリや `cmd/` は使っていません。この規模ではパッケージを 5 つに分けるだけで十分で、構造を増やすほど「ファイルが多いだけ」になります。

## 各レイヤーを実物のコードで見る

### domain — エンティティと不変条件

```go
// domain/tech.go
package domain

import "fmt"

// Tech represents a fictional technology evaluated by the SF-Meter.
type Tech struct {
	ID          int    `json:"id"`
	Name        string `json:"name"`
	Description string `json:"description"`
	Score       int    `json:"score"` // 0–100: likelihood of real-world realization
}

// Validate checks domain invariants.
func (t Tech) Validate() error {
	if t.Name == "" {
		return fmt.Errorf("tech: name must not be empty")
	}
	if t.Score < 0 || t.Score > 100 {
		return fmt.Errorf("tech: score %d is out of range [0, 100]", t.Score)
	}
	return nil
}
```

`import` は `fmt` だけ。**ビジネスルール（スコアは 0〜100、名前は必須）はここにしか書かない**のがポイントです。DB の制約（`CHECK (score >= 0 AND score <= 100)`）も `db/init.sql` に置いていますが、それは「二重の守り」であって、ルールの正はドメインです。

### usecase — 自分が必要とする抽象を自分で定義する

```go
// usecase/ports.go
package usecase

import "github.com/nemdull/sf-meter-api/domain"

// TechRepository は技術情報のデータ取得を抽象化するインターフェース
type TechRepository interface {
	FindAll() ([]domain.Tech, error)
	FindByID(id int) (*domain.Tech, error)
}
```

```go
// usecase/service.go
package usecase

import "github.com/nemdull/sf-meter-api/domain"

type TechService struct {
	repo TechRepository
}

func NewTechService(repo TechRepository) *TechService {
	return &TechService{repo: repo}
}

func (s *TechService) GetAll() ([]domain.Tech, error) {
	return s.repo.FindAll()
}

func (s *TechService) GetByID(id int) (*domain.Tech, error) {
	return s.repo.FindByID(id)
}
```

いま `TechService` は「リポジトリに委譲するだけ」でほぼ空です。それでも層を置いておく理由は、**将来ここに入るロジック（スコア順の並び替え、複数リポジトリの合成、キャッシュ）の置き場所を先に決めておく**ためです。空の層を「無駄」と見るか「予約席」と見るかは好みが分かれますが、小さく始めたいなら予約席にして、増えてから中身を書けばいい、というのがこのリポジトリの判断です。

### presentation — HTTP のことはここだけが知っている

```go
// presentation/handler.go（抜粋）
package presentation

// TechUseCase は TechHandler が依存するビジネスロジックを抽象化する。
// presentation層側で定義することで、usecase.TechServiceの具象型に縛られず、
// テストではモックを注入できる。
type TechUseCase interface {
	GetAll() ([]domain.Tech, error)
	GetByID(id int) (*domain.Tech, error)
}

type TechHandler struct {
	uc TechUseCase
}

func NewTechHandler(uc TechUseCase) *TechHandler {
	return &TechHandler{uc: uc}
}

// GET /techs/{id}
func (h *TechHandler) GetTechByID(w http.ResponseWriter, r *http.Request) {
	idStr := r.PathValue("id") // Go 1.22+ の ServeMux パスパターン
	id, err := strconv.Atoi(idStr)
	if err != nil || id <= 0 {
		http.Error(w, "Invalid id parameter", http.StatusBadRequest)
		return
	}

	tech, err := h.uc.GetByID(id)
	if err != nil {
		http.Error(w, "Failed to get tech", http.StatusInternalServerError)
		return
	}
	if tech == nil {
		http.Error(w, "Tech not found", http.StatusNotFound)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(toDTO(*tech)); err != nil {
		http.Error(w, "Failed to encode response", http.StatusInternalServerError)
	}
}
```

2 つの判断があります。

1. **`TechUseCase` インターフェースを presentation 側で定義している。** `usecase.TechService` という具象型を import せず、「ハンドラーが必要とする 2 メソッド」だけを自分で宣言する。これで `handler_test.go` はモックを差すだけでテストでき、usecase 層の変更にも引きずられません（Go の「インターフェースは使う側が定義する」慣習そのものです）。
2. **`domain.Tech` をそのまま返さず `dto.go` の `TechDTO` に変換している。** 今は構造が同じですが、API のレスポンス形式とドメインモデルは変化する理由が違います。変換を 1 か所に閉じ込めておくと、片方を変えてももう片方が壊れません。

### infrastructure — 実装は 2 つ、インターフェースは 1 つ

メモリ実装は開発とテスト用です。

```go
// infrastructure/memory.go（抜粋）
type MemoryTechRepo struct {
	mu    sync.RWMutex
	store map[int]domain.Tech
	next  int
}

// NewMemoryTechRepo seeds the repo with default SF technologies.
func NewMemoryTechRepo() *MemoryTechRepo {
	r := &MemoryTechRepo{store: make(map[int]domain.Tech), next: 1}
	seeds := []domain.Tech{
		{Name: "ドミネーター", Description: "支配射撃体系", Score: 35},
		{Name: "シビラシステム", Description: "脳スキャンで犯罪係数を算出する予防監視システム", Score: 91},
		{Name: "ニューラルリンク", Description: "脳とコンピュータを直接接続するBCIデバイス", Score: 82},
	}
	for _, t := range seeds {
		_ = r.create(t)
	}
	return r
}

func (r *MemoryTechRepo) FindByID(id int) (*domain.Tech, error) {
	r.mu.RLock()
	defer r.mu.RUnlock()
	t, ok := r.store[id]
	if !ok {
		return nil, nil
	}
	return &t, nil
}
```

PostgreSQL 実装は `database/sql` と `lib/pq` だけで書いています。

```go
// infrastructure/postgres.go（抜粋）
type PostgresTechRepo struct {
	db *sql.DB
}

// NewPostgresTechRepo opens a connection and verifies it with Ping.
func NewPostgresTechRepo(dsn string) (*PostgresTechRepo, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("postgres: failed to open: %w", err)
	}
	if err := db.Ping(); err != nil {
		return nil, fmt.Errorf("postgres: failed to connect: %w", err)
	}
	return &PostgresTechRepo{db: db}, nil
}

func (r *PostgresTechRepo) FindByID(id int) (*domain.Tech, error) {
	row := r.db.QueryRow(
		`SELECT id, name, description, score FROM techs WHERE id = $1`,
		id,
	)
	var t domain.Tech
	if err := row.Scan(&t.ID, &t.Name, &t.Description, &t.Score); err != nil {
		if err == sql.ErrNoRows {
			return nil, nil
		}
		return nil, fmt.Errorf("postgres FindByID: %w", err)
	}
	return &t, nil
}
```

ここで決めた約束が一つあります。**「見つからない」は `(nil, nil)` で返し、エラーにしない。** `sql.ErrNoRows` は infrastructure の都合なので外に漏らさず、ハンドラーは `tech == nil` を見て 404 を返す。メモリ実装も同じ約束を守るので、どちらを差しても presentation のコードは変わりません。

（`FindAll` では `rows.Err()` も確認しています。`for rows.Next()` を抜けた理由が「終端」なのか「途中のエラー」なのかは、これを見ないと区別できません。）

### registry — 配線だけがここにある

```go
// registry/registry.go
type Registry struct {
	dsn string
}

// New reads DATABASE_URL from the environment;
// NewTechHandler falls back to the in-memory repository when it's unset.
func New() (*Registry, error) {
	return &Registry{dsn: os.Getenv("DATABASE_URL")}, nil
}

func (r *Registry) NewTechHandler() (*presentation.TechHandler, error) {
	var repo usecase.TechRepository
	if r.dsn != "" {
		pgRepo, err := infrastructure.NewPostgresTechRepo(r.dsn)
		if err != nil {
			return nil, err
		}
		repo = pgRepo
	} else {
		repo = infrastructure.NewMemoryTechRepo()
	}
	svc := usecase.NewTechService(repo)
	return presentation.NewTechHandler(svc), nil
}
```

**具象型の名前（`PostgresTechRepo`、`MemoryTechRepo`、`TechService`）が全部出てくるのはこのファイルだけ**です。「DB を変えたくなったら registry を 1 行変える」が、依存の向きを守った見返りです。

### main.go — 起動とルーティングだけ

```go
func main() {
	reg, err := registry.New()
	if err != nil { /* ... */ }
	handler, err := reg.NewTechHandler()
	if err != nil { /* ... */ }

	mux := http.NewServeMux()
	mux.HandleFunc("GET /techs", handler.GetTechs)
	mux.HandleFunc("GET /techs/{id}", handler.GetTechByID)
	mux.HandleFunc("GET /health", handleHealth)

	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	addr := net.JoinHostPort("0.0.0.0", port)
	log.Fatal(http.ListenAndServe(addr, mux))
}
```

Go 1.22 で ServeMux がメソッドとパスパラメータを扱えるようになったので、この規模なら gorilla/mux も chi も要りません。

## テスト — DB なしで 2 層をそれぞれ単体テストする

依存の向きを守った恩恵が一番わかりやすいのがテストです。`go test ./...` は DB も HTTP サーバーも立てずに終わります。

usecase 層は、`TechRepository` を満たすモックを差します。エラー系は `forceErr` で注入。

```go
// usecase/tech_service_test.go（抜粋）
type mockTechRepo struct {
	techs    []domain.Tech
	forceErr error
}

func (m *mockTechRepo) FindAll() ([]domain.Tech, error) {
	if m.forceErr != nil {
		return nil, m.forceErr
	}
	return m.techs, nil
}

func TestTechService_GetAll_RepoError(t *testing.T) {
	repo := &mockTechRepo{forceErr: errors.New("db error")}
	svc := usecase.NewTechService(repo)

	_, err := svc.GetAll()
	if err == nil {
		t.Fatal("expected error, got nil")
	}
}
```

ドメインの不変条件はテーブル駆動で境界値を確認します。

```go
func TestTechService_ScoreRange(t *testing.T) {
	tests := []struct {
		name    string
		score   int
		wantErr bool
	}{
		{"valid low boundary", 0, false},
		{"valid high boundary", 100, false},
		{"below range", -1, true},
		{"above range", 101, true},
	}
	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			err := domain.Tech{Name: "test", Score: tc.score}.Validate()
			if tc.wantErr && err == nil {
				t.Errorf("expected error for score %d, got nil", tc.score)
			}
			if !tc.wantErr && err != nil {
				t.Errorf("unexpected error for score %d: %v", tc.score, err)
			}
		})
	}
}
```

presentation 層は `httptest` で HTTP の層だけを試します。パスパラメータは `req.SetPathValue("id", "999")` で直接セットできるので、ServeMux を経由する必要がありません。

```go
// presentation/handler_test.go（抜粋）
func TestGetTechByID_NotFound(t *testing.T) {
	uc := &mockTechUseCase{findTech: nil}
	h := presentation.NewTechHandler(uc)

	req := httptest.NewRequest(http.MethodGet, "/techs/999", nil)
	req.SetPathValue("id", "999")
	rec := httptest.NewRecorder()
	h.GetTechByID(rec, req)

	if rec.Code != http.StatusNotFound {
		t.Errorf("expected 404, got %d", rec.Code)
	}
}
```

「`(nil, nil)` なら 404」「不正な ID なら 400」「usecase がエラーなら 500」という**ハンドラーの責務だけ**を確認しています。DB の中身には一切触れません。

## 動かし方

```bash
# メモリ実装で起動（DATABASE_URL 無し）
go run main.go
curl localhost:8080/techs/2
# {"id":2,"name":"シビラシステム","description":"...","score":91}

# PostgreSQL 16 と一緒に起動（db/init.sql がテーブル作成と15件のシードを行う）
docker compose up --build

# テストと静的検査
go test ./...
go vet ./...
```

`docker-compose.yml` は `db` に `pg_isready` のヘルスチェックを付け、`app` は `depends_on: condition: service_healthy` で DB の準備を待ちます。「起動順の競合で最初の接続に失敗する」典型的な問題を、リトライを書かずに避けられます。

## 割り切ったこと（正直に）

小さく保つために、意図的に後回しにしているものがあります。

- **`TechRepository` が 2 か所にある。** `domain/tech.go` と `usecase/ports.go` に同じインターフェースが定義されています。「各層が必要な抽象を自分で持つ」方針の名残ですが、今は中身が同一なので片方に寄せるほうが読みやすい。次に触るときの整理候補です
- **書き込み系は PostgreSQL 実装にしかない。** `Create`／`Update`／`Delete` は `postgres.go` に実装済みですが、インターフェースにもハンドラーにもまだ出していません。API は GET 2 本のまま
- **認証・ページング・ログ出力はない。** 学習用の最小構成に留めています
- **DB の接続プール設定を触っていない。** `sql.Open` のデフォルトのままです

## まとめ

| レイヤー | 責務 | import してよいもの |
|---|---|---|
| domain | エンティティと不変条件 | 標準ライブラリだけ |
| usecase | ビジネスロジック、必要な抽象（ports） | domain |
| presentation | HTTP の入出力、DTO 変換、自分用の抽象 | domain |
| infrastructure | リポジトリの実装（メモリ／PostgreSQL） | domain、ドライバ |
| registry | 配線。具象型を知る唯一の場所 | 全部 |

守っているルールは一つだけ、**「依存の矢印を内側に向ける」**です。それを守った結果として、

- `DATABASE_URL` の有無で実装が差し替わる（registry の 1 分岐）
- usecase と presentation を DB なしでテストできる
- 「見つからない」の表現（`nil, nil`）を実装間でそろえれば、ハンドラーは実装を知らずに済む

という具体的な見返りが得られました。パターンを完璧に守ることより、この一点を意識して小さく始めるのがおすすめです。

---

*コードは [github.com/nemdull/sf-meter-api](https://github.com/nemdull/sf-meter-api) にあります。Issue や PR も歓迎です。*
