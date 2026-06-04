# データベース × Web

> **データサイエンス学部の方へ：**  
> SQL の基本（SELECT・JOIN・集計など）は既習の前提で書いています。  
> このページでは **Web アプリ・API からデータベースを使う方法** に集中します。

---

## はじめて読む人へ

データベースとWebをつなぐと、ユーザー登録、投稿、検索、予測ログの保存など、状態を持つアプリを作れるようになります。ここでは、API とデータベースの間で何が起きているかを意識して読みます。


### 読む前に押さえること

- API はリクエストを受け取り、DBから必要なデータを取得または保存します。
- ORM は、Python のオブジェクトとDBのテーブルを対応させる仕組みです。
- 接続情報やパスワードは環境変数で管理します。

### 読み終えたら説明できること

- CRUD と Web API の関係を説明できる。
- SQLAlchemy の役割を理解できる。
- N+1問題やマイグレーションの基本を説明できる。

---

## Web 開発でのデータベースの役割

静的サイト（Astro サイトなど）はデータベースを使いませんが、動的な Web アプリには必須です。

| ユースケース | 例 |
|------------|-----|
| ユーザー管理 | 会員登録・ログイン・プロフィール |
| コンテンツ管理 | 記事・商品・イベントの CRUD（後述） |
| 分析結果の保存 | ML モデルの予測ログ・統計サマリー |
| セッション管理 | カート・フォームの一時保存 |

---

> **CRUD とは：** データ操作の基本 4 種類の頭文字です。**C**reate（作成）・**R**ead（取得）・**U**pdate（更新）・**D**elete（削除）。ほぼすべての Web アプリは、この 4 操作の組み合わせでデータを管理します。SQL の INSERT・SELECT・UPDATE・DELETE に対応します。

## Web 開発でよく使う DB の種類

| DB | 特徴 | 用途 |
|----|------|------|
| **PostgreSQL** | 高機能・型が豊富（配列・JSON 対応） | 本番 Web サービス全般 |
| **MySQL / MariaDB** | 高速・シェアが大きい | Web アプリ全般 |
| **SQLite** | ファイル 1 つで動きます | 開発・小規模ツール・Streamlit アプリ |
| **Redis** | インメモリ・超高速 | キャッシュ・セッション管理 |

---

## SQL クイックリファレンス

> **情報工学メモ：インデックスとは（B 木、なぜ速くなるか）**  
> テーブルに数百万行あるとき、`WHERE id = 1234` のような検索を全行スキャンすると非常に遅くなります。**インデックス（索引）** は本の索引と同じで、特定の列の値とその行の場所を事前に整理した構造です。多くの RDBMS では **B 木（B-tree）** というデータ構造でインデックスを実装しており、二分探索に似た仕組みで O(log n) の時間で目的の行を見つけられます。ただしインデックスは INSERT・UPDATE のたびに更新コストがかかるため、検索頻度の高い列（主キー・外部キー・WHERE でよく使う列）に絞って設定するのが基本です。

DS コースで学んだ内容の確認用です。

```sql
-- 基本的な取得
SELECT id, name, score
FROM   users
WHERE  score >= 80
ORDER  BY score DESC
LIMIT  10;

-- 集計
SELECT   category, COUNT(*) AS count, AVG(score) AS avg_score
FROM     users
GROUP BY category
HAVING   COUNT(*) >= 5;

-- JOIN
SELECT u.name, o.amount, o.created_at
FROM   users  u
JOIN   orders o ON u.id = o.user_id
WHERE  o.created_at >= '2024-01-01';

-- サブクエリ
SELECT *
FROM   users
WHERE  score > (SELECT AVG(score) FROM users);

-- INSERT / UPDATE / DELETE
INSERT INTO users (name, email) VALUES ('田中', 'tanaka@example.com');
UPDATE users SET score = 95 WHERE id = 1;
DELETE FROM users WHERE id = 1;   -- WHERE 忘れに注意！
```

このSQL群は、Web API の裏側でよく実行される操作です。`GET /users` のような取得APIでは `SELECT`、フォーム送信で新しいユーザーを作るときは `INSERT`、プロフィール編集では `UPDATE`、削除ボタンでは `DELETE` が対応します。

Webアプリでは、SQLを単独で覚えるより、「どのHTTPリクエストが、どのDB操作につながるか」を意識すると理解しやすくなります。特に `UPDATE` と `DELETE` は `WHERE` を忘れると大量の行を変えてしまうため、API側でも対象IDを明確に受け取る設計が重要です。

---

## Python から PostgreSQL に接続する

### psycopg2（生 SQL）

```bash
pip install psycopg2-binary
```

`psycopg2-binary` は、Python から PostgreSQL に接続するためのドライバです。ドライバは、Python の関数呼び出しを PostgreSQL が理解できる通信に変換する役割を持ちます。

```python
import psycopg2
import os

# 接続（接続情報は環境変数で管理します）
conn = psycopg2.connect(os.environ["DATABASE_URL"])
cur  = conn.cursor()

# クエリ実行（SQL インジェクション対策：%s でプレースホルダを使います）
cur.execute(
    "SELECT id, name, score FROM users WHERE score >= %s",
    (80,)  # ← タプルで渡します
)
rows = cur.fetchall()
for row in rows:
    print(row)  # (1, '田中', 88.5)

# 書き込み操作はコミットが必要です
cur.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ("山田", "yamada@example.com")
)
conn.commit()

cur.close()
conn.close()
```

このコードでは、`DATABASE_URL` から接続情報を読み、カーソルでSQLを実行しています。`%s` は文字列フォーマットではなく、DBドライバに値を安全に渡すためのプレースホルダです。`(80,)` のようにタプルで渡すことで、入力値がSQL命令として混ざるのを防ぎます。

書き込み操作では `conn.commit()` が必要です。DBは、途中で失敗したときに中途半端な状態を残さないよう、変更を一時的にトランザクション内に保持します。`commit` して初めて確定され、失敗時には `rollback` で取り消せます。

### SQLAlchemy（ORM・推奨）

**ORM** （Object Relational Mapper）とは、DB のテーブルを Python クラスとして扱える仕組みです。SQL を直接書く量を減らし、型安全に操作できます。

```bash
pip install sqlalchemy psycopg2-binary
```

SQLAlchemy はORM本体、`psycopg2-binary` はPostgreSQLとの接続に使うドライバです。ORMを使う場合でも、裏側ではドライバを通じてSQLが実行されています。

```python
from sqlalchemy import create_engine, Column, Integer, String, Float
from sqlalchemy.orm import DeclarativeBase, Session

DATABASE_URL = "postgresql://user:pass@localhost:5432/mydb"
engine = create_engine(DATABASE_URL)

# テーブル定義
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"
    id    = Column(Integer, primary_key=True, autoincrement=True)
    name  = Column(String(100), nullable=False)
    email = Column(String(255), unique=True)
    score = Column(Float, default=0.0)

# テーブル作成
Base.metadata.create_all(engine)

# CRUD 操作
with Session(engine) as session:
    # Create
    user = User(name="田中", email="tanaka@example.com", score=88.5)
    session.add(user)
    session.commit()

    # Read
    users = session.query(User).filter(User.score >= 80).all()
    for u in users:
        print(u.name, u.score)

    # Update
    user = session.get(User, 1)
    user.score = 95.0
    session.commit()

    # Delete
    session.delete(user)
    session.commit()
```

`User` クラスは Python のクラスですが、`__tablename__ = "users"` によってDBの `users` テーブルに対応します。`Column` は列の定義で、型や主キー、一意制約などを表します。`Base.metadata.create_all(engine)` は、定義されたテーブルがなければ作成します。

`Session` は、DB操作をまとめて管理する作業単位です。`session.add` で追加予定にし、`commit` で確定します。`session.get(User, 1)` は主キーで1件取得し、`session.delete` は削除予定にします。ORMは便利ですが、どのタイミングでSQLが発行されるかを意識しないと、後のN+1問題につながります。

---

## FastAPI × データベース

FastAPI とデータベースを組み合わせると、リクエストを受け取ってDBを読み書きし、その結果をJSONとして返すAPIを作れます。ここで大切なのは、リクエストごとにDB接続を適切に開き、処理が終わったら閉じることです。

接続を閉じ忘れると、アクセスが増えたときにDBへ接続できなくなることがあります。そのため、依存関係としてセッションを管理し、リクエスト単位で確実に解放する設計にします。

FastAPI と SQLAlchemy を組み合わせた典型的なパターンです。

```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel

app = FastAPI()

# DB セッションを各リクエストで取得・解放する依存関係
def get_db():
    db = Session(engine)
    try:
        yield db
    finally:
        db.close()

class UserCreate(BaseModel):
    name: str
    email: str

class UserResponse(BaseModel):
    id: int
    name: str
    score: float

    class Config:
        from_attributes = True  # ORM モデルから変換します

@app.get("/users", response_model=list[UserResponse])
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()

@app.post("/users", response_model=UserResponse)
def create_user(data: UserCreate, db: Session = Depends(get_db)):
    user = User(**data.dict())
    db.add(user)
    db.commit()
    db.refresh(user)
    return user

@app.get("/users/{user_id}", response_model=UserResponse)
def get_user(user_id: int, db: Session = Depends(get_db)):
    user = db.get(User, user_id)
    if not user:
        raise HTTPException(status_code=404, detail="ユーザーが見つかりません")
    return user
```

`Depends(get_db)` は、各リクエストでDBセッションを用意し、処理後に閉じるための仕組みです。エンドポイント関数は `db` を受け取るだけでよく、接続の後片付けを毎回手で書かずに済みます。

`response_model` は、返すJSONの形をPydanticモデルで指定します。DBのモデルをそのまま返すのではなく、APIとして公開する形を明示することで、内部だけで使う列や秘密情報を誤って返すリスクを減らせます。

---

## N+1 問題（Web DB でよく起きるバグ）

> **情報工学メモ：トランザクションと ACID 特性**  
> **トランザクション** とは、複数の DB 操作をひとまとまりの「1 処理」として扱う仕組みです。たとえば「口座 A から引き落とし→口座 B に入金」の 2 操作は、途中で失敗すると不整合が生じます。トランザクションを使えば「全部成功（コミット）か全部取り消し（ロールバック）」のどちらかを保証できます。信頼性の高い DB はトランザクションに **ACID** の 4 性質を保証します： **A**tomicity（不可分性：全完了 or 全取消）、**C**onsistency（一貫性：制約を常に満たします）、**I**solation（独立性：並列実行しても互いに干渉しません）、**D**urability（永続性：コミット後はシステム障害でも消えません）。Python コードでは `session.commit()` と `session.rollback()` がこれに対応します。

**なぜ問題なのか：** ユーザーが 100 人いれば SQL が 101 回（1 + 100）発行されます。1000 人なら 1001 回です。ループの中で毎回 DB に問い合わせるため、件数が増えるほど応答時間が劇的に悪化します。DS の文脈ではあまり意識しませんが、Web API では重要なパフォーマンス問題です。

```python
# ❌ N+1 問題：ユーザー数だけ SQL が発行されます
users = session.query(User).all()          # SQL × 1
for user in users:
    orders = session.query(Order).filter(
        Order.user_id == user.id
    ).all()                                 # SQL × N 回！

# ✅ JOIN で一度に取得します
from sqlalchemy.orm import joinedload

users = session.query(User).options(
    joinedload(User.orders)
).all()                                     # SQL × 2（最適）
```

N+1問題は、「画面に表示する件数が増えたときだけ急に遅くなる」典型的な原因です。最初の `users` 取得で1回、その後ユーザーごとに注文を取りに行くため、ユーザー数に比例してSQLが増えます。`joinedload` などで関連データをまとめて取得すると、DBとの往復回数を減らせます。

---

## Docker で PostgreSQL を起動する

WebアプリとDBを一緒に開発するときは、Docker Compose でDBを起動しておくと環境差分が少なくなります。チーム全員が同じPostgreSQLバージョン、同じ初期DB名で作業できます。

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER:     myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB:       mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data  # データを永続化します

volumes:
  postgres_data:
```

```bash
docker compose up -d   # バックグラウンドで起動します

# 接続確認
DATABASE_URL=postgresql://myuser:mypassword@localhost:5432/mydb
```

`postgres_data` は名前付きボリュームです。コンテナを止めたり作り直したりしても、DBの中身を残すために使います。逆に、学習中に完全に初期化したい場合は、ボリュームを削除する必要があります。

---

## Streamlit × SQLite（手軽なデータ永続化）

Streamlit の小規模ツールでは SQLite が手軽です。

```python
import sqlite3
import streamlit as st
import pandas as pd

conn = sqlite3.connect("predictions.db", check_same_thread=False)
conn.execute("""
    CREATE TABLE IF NOT EXISTS logs (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        input TEXT,
        prediction INTEGER,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
""")

# 予測ログを保存します
def save_log(input_data: dict, prediction: int):
    import json
    conn.execute(
        "INSERT INTO logs (input, prediction) VALUES (?, ?)",
        (json.dumps(input_data), prediction)
    )
    conn.commit()

# ログを表示します
logs = pd.read_sql("SELECT * FROM logs ORDER BY created_at DESC LIMIT 50", conn)
st.dataframe(logs)
```

SQLite はサーバーを起動せず、1つのファイルにデータを保存できるDBです。小規模なStreamlitアプリで、予測結果や入力履歴を少し残したい場合に向いています。この例では、`CREATE TABLE IF NOT EXISTS` でテーブルがなければ作り、`save_log` で予測ログを保存し、最後に `pd.read_sql` で最新50件をDataFrameとして読み込んで表示しています。

---

## よくある疑問

**Q. DS で使う pandas の DataFrame と DB のテーブルの違いは？**  
A. DataFrame はメモリ上の一時的なデータ構造です。DB は永続化されており、複数のユーザー・サーバーから同時アクセスできます。Web アプリでは DB が必須ですが、分析作業には DataFrame が便利です。両者を組み合わせる（`pd.read_sql()` で DB から DataFrame へ）のが典型的なパターンです。

**Q. ORM を使うべき？生 SQL を書くべき？**  
A. 一般的な CRUD は ORM が便利です。複雑な集計・分析クエリや、パフォーマンスが重要な箇所は生 SQL（または SQLAlchemy の `text()`）を使うのが現実的な判断です。

---


## 確認問題

1. データベース × Web は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [セキュリティ](セキュリティ) — 接続情報の安全な管理（.env）
- [Docker](Docker) — PostgreSQL をローカルで動かす
- [Python × Web API（FastAPI）](FastAPI) — API との組み合わせ

---

[← ホームへ](Home)
