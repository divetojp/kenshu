# Go 言語入門

Google が開発したシステムプログラミング言語です。**シンプルな文法・高速なコンパイル・goroutine による軽量並行処理**を特徴とし、Web サーバー・CLI ツール・マイクロサービス・Kubernetes 周辺ツールの事実上の標準言語です。Python の次に習得すると即戦力になります。

---

## はじめて読む人へ

Go を選ぶ理由は主に 3 つです。①Docker・Kubernetes・Terraform・GitHub Actions のランナーはすべて Go 製、②マイクロサービスのバックエンドでは Python より 10〜100 倍速い、③goroutine で並行処理が驚くほど簡単に書けます。Python を知っていれば 1 週間で基本的なサービスが書けます。

### 読む前に押さえること

- [Python基礎](Python) — プログラミングの基礎概念（変数・関数・型）
- [ネットワーク基礎](ネットワーク基礎) — HTTP・DNS の基礎

### 読み終えたら説明できること

- Go の型システムとインターフェースの仕組みを説明できる
- goroutine と channel による並行処理を実装できる
- シンプルな HTTP サーバーを書ける

---

## Go の特徴と設計思想

| 特徴 | 詳細 |
|------|------|
| **静的型付け** | コンパイル時に型エラーを検出。Python と異なり実行前に安全性確認 |
| **高速コンパイル** | 大規模プロジェクトでも数秒でバイナリ生成 |
| **ガベージコレクション** | メモリ管理は自動（C/C++ より安全）|
| **goroutine** | OS スレッドより軽量（数 KB）な並行実行単位 |
| **シンプルな文法** | キーワード 25 個のみ（Python: 35 個、Java: 50 個超）|
| **クロスコンパイル** | `GOOS=linux go build` で Mac から Linux バイナリを生成 |

---

## 基本文法

### 変数・型

```go
// 変数宣言（2 通り）
var name string = "Alice"
age := 30  // 型推論（:= は Go の特徴）

// 基本型
var i int     = 42
var f float64 = 3.14
var b bool    = true
var s string  = "hello"

// 定数
const Pi = 3.14159
```

### 関数

```go
// 複数戻り値（Go の特徴）
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 3)
if err != nil {
    log.Fatal(err)  // エラーハンドリングが明示的
}
```

**Go のエラーハンドリング：** 例外（try/catch）を使わず、エラーを戻り値として扱います。`if err != nil` パターンが Go コードの特徴的なスタイルです。

### 構造体とメソッド

```go
type User struct {
    ID   int
    Name string
    Age  int
}

// メソッド（レシーバ付き関数）
func (u User) String() string {
    return fmt.Sprintf("User{%d, %s, %d}", u.ID, u.Name, u.Age)
}

func (u *User) Birthday() {  // ポインタレシーバで値を変更
    u.Age++
}

alice := User{ID: 1, Name: "Alice", Age: 30}
alice.Birthday()
```

### インターフェース

Go のインターフェースは**暗黙的に実装**されます（implements 宣言不要）。

```go
type Stringer interface {
    String() string
}

// User が String() を持つ → 自動的に Stringer を実装
func Print(s Stringer) {
    fmt.Println(s.String())
}

Print(alice)  // User は暗黙的に Stringer インターフェースを満たす
```

---

## スライス・マップ

```go
// スライス（動的配列）
nums := []int{1, 2, 3, 4, 5}
nums = append(nums, 6)
sub := nums[1:4]  // [2 3 4]

// マップ
scores := map[string]int{
    "Alice": 90,
    "Bob":   85,
}
scores["Carol"] = 92
val, ok := scores["Dave"]  // 存在確認
if !ok {
    fmt.Println("Dave not found")
}
```

---

## goroutine と channel（並行処理）

### goroutine

`go` キーワードで関数を軽量スレッドとして非同期実行します。

```go
func fetchData(url string) {
    // HTTP リクエストなど重い処理
}

// 3 つの URL を並列取得
go fetchData("https://api.example.com/users")
go fetchData("https://api.example.com/products")
go fetchData("https://api.example.com/orders")
```

goroutine は OS スレッドではなく Go ランタイムが管理する軽量なコルーチンです。数万個〜数百万個を同時に起動できます。

### channel

goroutine 間の安全なデータ受け渡し。

```go
ch := make(chan int, 3)  // バッファサイズ 3 のチャネル

// 送信（別 goroutine から）
go func() {
    for i := 0; i < 5; i++ {
        ch <- i  // チャネルに送信
    }
    close(ch)
}()

// 受信
for val := range ch {
    fmt.Println(val)
}
```

### select

複数の channel を待ち受ける。

```go
select {
case msg := <-ch1:
    fmt.Println("ch1:", msg)
case msg := <-ch2:
    fmt.Println("ch2:", msg)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
}
```

---

## HTTP サーバー

```go
package main

import (
    "encoding/json"
    "net/http"
)

type Response struct {
    Message string `json:"message"`
    Status  int    `json:"status"`
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(Response{
        Message: "OK",
        Status:  200,
    })
}

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", healthHandler)
    mux.HandleFunc("GET /users/{id}", getUserHandler)  // Go 1.22+: メソッド指定

    http.ListenAndServe(":8080", mux)
}
```

標準ライブラリだけで本番品質の HTTP サーバーが書けます。Gin・Echo・Fiber などのフレームワークもありますが、標準ライブラリが十分強力です。

---

## Python vs Go の使い分け

| 観点 | Python | Go |
|------|--------|-----|
| データ分析・ML | ◎ | △ |
| プロトタイプ | ◎ | ○ |
| 高スループット API | △ | ◎ |
| 並行処理 | △（GIL の制約）| ◎（goroutine）|
| 実行速度 | △ | ◎（C の 5〜20%）|
| CLI ツール | ○ | ◎（シングルバイナリ）|
| 学習コスト | 低 | 低〜中 |

**DS エンジニアの典型的な分担：**
- データ前処理・モデル学習 → Python
- 推論 API・バックエンドサービス → Go（または FastAPI）
- CLI ツール・運用スクリプト → Go または Bash

---

## 数学的導出

### goroutine のスケーラビリティ

OS スレッドのスタックサイズは通常 1〜8 MB（固定）ですが、goroutine のスタックは最初 2〜4 KB から始まり必要に応じて拡張されます。

!!! info ""
    **1 GB メモリで**

    OS スレッド: 1 GB / 2 MB = ~500 スレッドが上限
    goroutine:  1 GB / 4 KB = ~250,000 goroutine が起動可能
Go ランタイムの M:N スレッドスケジューラが $M$ 個の goroutine を $N$ 個の OS スレッドにマッピングし、ブロッキング I/O 時に別の goroutine を同じスレッドで実行します。

---

## 確認問題

1. Go のインターフェースが「暗黙的な実装」である理由と、Java の `implements` との違いを説明してください。
2. goroutine が OS スレッドより多く起動できる理由を、スタックサイズの観点から説明してください。
3. channel を使わずにグローバル変数でデータを共有することの問題点を説明してください。

---

## 関連ページ

- [Python基礎](Python) — 基礎プログラミング（Go との比較）
- [gRPC](gRPC) — Go で実装されるサービス間通信
- [Docker](Docker) — Go バイナリのコンテナ化
- [マイクロサービス](マイクロサービス) — Go が使われるアーキテクチャ
- [並列・並行処理](並列・並行処理) — 並行処理の一般概念

---

[← ホームへ](Home)
