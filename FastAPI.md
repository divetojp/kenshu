# Python × Web API（FastAPI）

## Web API とは

**Web API** （Application Programming Interface）とは、プログラムから HTTP で呼び出せるインターフェースです。

たとえば：
- 天気アプリが気象データを取得する仕組み
- フロントエンド（Astro）がデータをサーバーから取得する仕組み
- scikit-learn モデルに予測をリクエストする仕組み

DS の文脈では「訓練したモデルを API として公開する」のが典型的な使い方です。

---

## はじめて読む人へ

FastAPI は、Python で Web API を作るためのフレームワークです。データや機械学習モデルを、ブラウザや別のアプリから呼び出せる形にします。


### 読む前に押さえること

- API は、プログラム同士がデータをやり取りする入口です。
- HTTP メソッドは、取得・作成・更新・削除などの意図を表します。
- 型定義を書くと、リクエストの形を自動で検証できます。

### 読み終えたら説明できること

- エンドポイント、リクエスト、レスポンスの意味を説明できる。
- GET と POST の違いを理解できる。
- FastAPI で簡単なAPIを起動できる。

---

## FastAPI とは

**Python で REST API を作るためのフレームワーク** です。Python の型ヒントを活用した自動ドキュメント生成と高速な動作が特徴です。

> **情報工学メモ：REST とは何か（ステートレス、リソース指向、HTTP メソッドとの対応）**  
> **REST**（Representational State Transfer）は Web API の設計スタイルの一つです。主な原則は以下の 3 つです。① **リソース指向** ：操作対象を URL で表現します（例：`/users/1` はユーザー ID=1 のリソース）。② **HTTP メソッドで操作を表現** ：取得は `GET`、作成は `POST`、更新は `PUT/PATCH`、削除は `DELETE` と役割を分担します。③ **ステートレス** ：各リクエストはそれだけで完結し、サーバー側はセッション情報を保持しません（認証情報などはリクエストごとに送ります）。これにより、サーバーをスケールアウト（複数台に増やす）しやすくなります。FastAPI はこの REST の原則に沿って API を作るフレームワークです。REST 設計の詳細（認証・バージョニング・ステータスコードの使い分けなど）は [Web / API 設計](WebAPI設計) で扱います。

まず、FastAPI 本体と、それをローカルで起動するための `uvicorn` をインストールします。FastAPI は API の定義、uvicorn はその API を HTTP サーバーとして動かす役割です。

```bash
pip3 install fastapi uvicorn
```

> **uvicorn** は FastAPI アプリを動かすための ASGI サーバーです。**ASGI**（Asynchronous Server Gateway Interface）とは、Python の Web アプリとサーバーをつなぐ非同期対応の規格です。古い規格 WSGI の後継で、`async/await` を使った高速な処理が可能です。

---

## エンドポイントとは

**API に対してリクエストを送る「窓口」の URL** のことです。`/users` や `/predict` など、用途ごとに異なる URL（エンドポイント）を定義します。

FastAPI では `@app.get("/url")` のようなデコレーターで、URL とそれを処理する関数を紐づけます。

## 最初の API

次のコードは、FastAPI アプリの最小例です。`app = FastAPI()` で API アプリを作り、`@app.get(...)` で URL と関数を結びつけます。

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"message": "Hello, FastAPI!"}

@app.get("/items/{item_id}")
def read_item(item_id: int, q: str = None):
    return {"item_id": item_id, "q": q}
```

`read_root` は `/` に GET リクエストが来たときに実行されます。`read_item` の `{item_id}` はパスパラメータで、URL の一部を Python の引数として受け取ります。

関数が辞書を返すと、FastAPI は自動で JSON レスポンスに変換します。Web API では、この「リクエストを受け取り、JSON を返す」という流れが基本です。

起動します：

`main.py` を保存したら、次のコマンドで開発サーバーを起動します。

```bash
uvicorn main:app --reload
# → http://localhost:8000 で API が起動します
# → http://localhost:8000/docs で自動生成ドキュメントが見られます
```

**`--reload`** をつけると、コードを変更するたびに自動再起動します（開発時に便利です）。

---

## HTTP メソッドとエンドポイント

HTTP メソッドは、同じ URL でも「何をしたいか」を区別するためのものです。`GET` は取得、`POST` は作成、`DELETE` は削除という意味を持たせます。

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 仮のデータストア
items = []

# GET：データを取得します
@app.get("/items")
def get_items():
    return items

# POST：データを作成します
class Item(BaseModel):
    name: str
    price: float
    in_stock: bool = True

@app.post("/items")
def create_item(item: Item):
    items.append(item)
    return item

# DELETE：データを削除します
@app.delete("/items/{index}")
def delete_item(index: int):
    removed = items.pop(index)
    return {"removed": removed}
```

この例では、`items` というリストを仮のデータストアとして使っています。`GET /items` は一覧取得、`POST /items` は新しい item の追加、`DELETE /items/{index}` は指定した位置の削除です。

実務ではメモリ上のリストではなくデータベースに保存します。メモリ上のリストはサーバーを再起動すると消えるため、学習用の簡易例として理解してください。

---

## Pydantic モデル（型の自動検証）

Pydantic モデルは、API に送られてくる JSON の形を定義するために使います。どのフィールドが必要か、値の型は何かをクラスとして書くと、FastAPI が自動で検証してくれます。

これにより、文字列が必要な場所に数値が来た、必須項目が抜けている、といった入力ミスを早い段階で検出できます。API の入口でデータの形を確認することは、安全で分かりやすい設計につながります。

> **バリデーションとは：** 受け取ったデータが正しい形式・範囲かをチェックすることです。例えば「年齢は整数でなければならない」「メールアドレスは @ を含む」などのルールを自動で検証します。不正なデータが来ると 422 エラーを返し、処理を続けません。

Pydantic を使うと、リクエストのデータを型で検証できます。

次のコードでは、ニュース記事を作成する API の入力形式を `NewsItem` として定義しています。クライアントから送られる JSON がこの形に合っているかを FastAPI が自動で確認します。

```python
from pydantic import BaseModel, Field
from typing import Optional
from datetime import date

class NewsItem(BaseModel):
    title: str = Field(..., min_length=1, max_length=200)
    content: str
    author: str
    published_at: date
    tags: list[str] = []
    is_published: bool = False

@app.post("/news")
def create_news(item: NewsItem):
    # FastAPI が自動でバリデーションしてくれます
    # 型が合わなければ 422 エラーが返ります
    return {"created": item}
```

`Field(..., min_length=1, max_length=200)` の `...` は必須項目であることを表します。`tags: list[str] = []` は文字列リスト、`is_published: bool = False` は省略されたときの初期値です。

Pydantic モデルを使うと、API の利用者にとっても「どんな JSON を送ればよいか」が明確になります。FastAPI の自動ドキュメントにもこの型情報が反映されます。

---

## pandas と組み合わせる

データサイエンスの API では、CSV や DataFrame から集計結果を返すことがあります。FastAPI と pandas を組み合わせると、分析結果を HTTP 経由で他のアプリから利用できるようになります。

```python
import pandas as pd
from fastapi import FastAPI
from fastapi.responses import JSONResponse

app = FastAPI()

@app.get("/data/summary")
def get_summary():
    df = pd.read_csv("data.csv")
    summary = {
        "rows": len(df),
        "columns": list(df.columns),
        "mean_score": df["score"].mean(),
        "top10": df.nlargest(10, "score").to_dict(orient="records"),
    }
    return summary

@app.get("/data/filter")
def filter_data(min_score: float = 0, category: str = None):
    df = pd.read_csv("data.csv")
    if min_score:
        df = df[df["score"] >= min_score]
    if category:
        df = df[df["category"] == category]
    return df.to_dict(orient="records")
```

`/data/summary` は、行数、列名、平均値、上位 10 件をまとめて返します。`/data/filter` は、クエリパラメータ `min_score` や `category` に応じて DataFrame を絞り込みます。

DataFrame はそのまま JSON にできないため、`to_dict(orient="records")` で「行ごとの辞書のリスト」に変換しています。大きなデータを返す場合は、件数制限やページングも考える必要があります。

---

## 非同期処理（async def）

FastAPI は非同期処理に対応しています。I/O 待ちが多い処理（DB クエリ、外部 API 呼び出し）は `async def` にすると並列処理できます。

非同期処理は、外部 API やデータベースの返事を待っている間に、サーバーが他のリクエストを処理できるようにする仕組みです。待ち時間が多い Web API では重要な考え方です。

```python
import httpx  # 非同期 HTTP クライアント

@app.get("/external-data")
async def get_external():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://api.example.com/data")
        return response.json()
```

`await client.get(...)` は、外部 API の応答を待つ処理です。待っている間、サーバー全体が止まるのではなく、他のリクエストを処理できます。

CPU バウンドな処理（機械学習の推論など）は通常の `def` でも構いません。

機械学習の推論のように CPU や GPU を使って計算している時間は、単に `async` にしても速くなるわけではありません。非同期が効くのは、主に通信やファイル読み込みのような I/O 待ちです。

---

## ミドルウェア（CORS の設定）

フロントエンド（Astro など別のドメイン）から API を呼ぶ場合、**CORS** の設定が必要です。

CORS は、ブラウザが異なるオリジンの API 呼び出しを制限するセキュリティの仕組みです。フロントエンドと API のドメインやポートが違う場合、API 側で許可する必要があります。

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:4321", "https://diveto.jp"],
    allow_methods=["*"],
    allow_headers=["*"],
)
```

`allow_origins` には、API を呼び出してよいフロントエンドの URL を書きます。開発中は `localhost` を許可し、本番では公開サイトの URL だけを許可するのが基本です。

---

## 自動ドキュメント

FastAPI は型情報から自動で API ドキュメントを生成します。

| URL | 内容 |
|-----|------|
| `http://localhost:8000/docs` | Swagger UI（インタラクティブに試せます） |
| `http://localhost:8000/redoc` | ReDoc（読みやすい仕様書） |

ブラウザから直接 API をテストできるため、開発・デバッグが非常に楽です。

---

## Docker で動かす

Docker を使うと、Python のバージョン、依存パッケージ、起動コマンドをひとまとめにできます。自分の PC では動くのにサーバーでは動かない、という環境差を減らせます。

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

この Dockerfile は、Python 3.11 の軽量イメージを使い、`requirements.txt` の依存関係を入れ、最後に uvicorn で FastAPI アプリを起動します。`--host 0.0.0.0` はコンテナ外からアクセスできるようにする指定です。

```txt
# requirements.txt
fastapi
uvicorn
pandas
scikit-learn
joblib
```

`requirements.txt` には、コンテナ内でインストールする Python パッケージを書きます。API が pandas や scikit-learn を使うなら、ここに忘れず含めます。

```bash
docker build -t my-api .
docker run -p 8000:8000 my-api
```

`docker build` はイメージを作り、`docker run` はそのイメージからコンテナを起動します。`-p 8000:8000` は、手元の 8000 番ポートをコンテナ内の 8000 番ポートにつなぐ指定です。

---

## よくある疑問

**Q. Flask との違いは？**  
A. FastAPI は型ヒントを活用した自動バリデーション・ドキュメント生成・非同期対応が標準で付いています。Flask はシンプルで歴史が長く情報も多いですが、型の安全性は弱いです。新規プロジェクトは FastAPI が推奨されることが多いです。

**Q. データは DB に保存すべき？**  
A. 本番 API では必須です。このページのサンプルはメモリや CSV を使っていますが、再起動でデータが消えます。[データベース × Web](データベース-Web) ページで DB との接続方法を解説しています。

**Q. 認証はどうする？**  
A. FastAPI には OAuth2・JWT 認証のサポートが組み込まれています。公式ドキュメントの「Security」セクションを参照してください。

---


## 確認問題

1. Python × Web API（FastAPI） は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [pandas / scikit-learn / matplotlib](pandas-sklearn) — モデルの準備
- [機械学習モデルのデプロイ](MLデプロイ) — この API を本番に公開する
- [データベース × Web](データベース-Web) — データの永続化
- [Docker](Docker) — コンテナ化の詳細
- [セキュリティ](セキュリティ) — API のセキュリティ対策

---

[← ホームへ](Home)
