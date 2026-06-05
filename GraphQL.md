# GraphQL

クライアントが「欲しいデータの形」を自分で指定できる API クエリ言語です。REST では「エンドポイントが返すデータを全部受け取る」のに対し、GraphQL では「名前とメールだけ欲しい」と宣言できます。フロントエンドとバックエンドの境界が多い開発で特に効果を発揮します。

---

## はじめて読む人へ

REST の問題を知っていると GraphQL の価値がわかります。「`/users/1` を叩いたら要らないフィールドが 20 個返ってきた」「ユーザー情報と投稿一覧を同時に取りたいのに 2 回リクエストが必要」——これを解決するのが GraphQL です。

### 読む前に押さえること

- [Web / API設計](WebAPI設計.md) の REST・エンドポイント・JSON レスポンスの概念
- [FastAPI](FastAPI.md) の基本的なルーティング

### 読み終えたら説明できること

- REST と GraphQL の使い分けを説明できる
- Query・Mutation・Subscription の違いを説明できる
- Python（Strawberry）で GraphQL API を実装できる

---

## REST vs GraphQL

!!! info ""
    ```text
    REST の問題例:

      画面に「ユーザー名」「プロフィール画像」だけ表示したい
      → GET /users/1 → { id, name, email, bio, avatar, created_at, ... } (20フィールド)
         要らないデータも全部来る（Over-fetching）

      画面に「ユーザー情報 + その投稿一覧」を表示したい
      → GET /users/1  (1回目)
      → GET /users/1/posts  (2回目)
         2回リクエストが必要（Under-fetching）

    GraphQL の解決:

      query {
        user(id: 1) {
          name        ← 欲しいものだけ指定
          avatar
          posts {
            title
            publishedAt
          }
        }
      }
      → 1回のリクエストで欲しいデータだけ返ってくる
    ```
| | REST | GraphQL |
|--|------|---------|
| エンドポイント | リソースごとに `/users`, `/posts` | `/graphql` の 1 つ |
| レスポンス | サーバーが決めた形 | クライアントが宣言した形 |
| Over-fetching | 起きやすい | 起きない |
| Under-fetching | 複数リクエストが必要 | 1 リクエストで解決 |
| 学習コスト | 低い | 高め |
| キャッシュ | HTTP レベルで自然にできる | 工夫が必要（Apollo Client 等）|
| 向いているケース | シンプルな API・公開 API | 複雑な UI・モバイル・BFF |

---

## 基本概念

### スキーマ（SDL：Schema Definition Language）

GraphQL API の「型定義書」です。クライアントはスキーマを見て何が取れるかを知ります。

```graphql
type User {
  id: ID!           # ! は必須（null にならない）
  name: String!
  email: String!
  posts: [Post!]!   # Post の配列
}

type Post {
  id: ID!
  title: String!
  content: String
  author: User!
  publishedAt: String
}

type Query {
  user(id: ID!): User        # 1 件取得
  users: [User!]!            # 一覧取得
  post(id: ID!): Post
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updatePost(id: ID!, title: String): Post!
  deletePost(id: ID!): Boolean!
}
```

### Query（データ取得）

```graphql
# ユーザー 1 件とその投稿を取得
query GetUserWithPosts {
  user(id: "1") {
    name
    email
    posts {
      title
      publishedAt
    }
  }
}

# 変数を使う（実際のアプリではこちらが多い）
query GetUser($id: ID!) {
  user(id: $id) {
    name
    email
  }
}
# 変数: { "id": "1" }
```

### Mutation（データ変更）

```graphql
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id
    name
    email
  }
}
# 変数: { "name": "田中", "email": "tanaka@example.com" }
```

---

## Python 実装（Strawberry + FastAPI）

```bash
pip install strawberry-graphql[fastapi]
```

```python
# main.py
import strawberry
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from typing import Optional

# --- 型定義 ---
@strawberry.type
class Post:
    id: strawberry.ID
    title: str
    content: Optional[str]

@strawberry.type
class User:
    id: strawberry.ID
    name: str
    email: str
    posts: list[Post]

# --- 仮のデータ ---
USERS = {
    "1": {"id": "1", "name": "田中", "email": "tanaka@example.com"},
    "2": {"id": "2", "name": "山田", "email": "yamada@example.com"},
}
POSTS = [
    {"id": "1", "title": "GraphQL入門", "content": "...", "user_id": "1"},
    {"id": "2", "title": "FastAPI Tips", "content": "...", "user_id": "1"},
]

# --- Resolver ---
@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: strawberry.ID) -> Optional[User]:
        u = USERS.get(str(id))
        if not u:
            return None
        user_posts = [
            Post(id=p["id"], title=p["title"], content=p["content"])
            for p in POSTS if p["user_id"] == str(id)
        ]
        return User(id=u["id"], name=u["name"], email=u["email"], posts=user_posts)

    @strawberry.field
    def users(self) -> list[User]:
        return [
            User(id=u["id"], name=u["name"], email=u["email"], posts=[])
            for u in USERS.values()
        ]

@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, name: str, email: str) -> User:
        new_id = str(len(USERS) + 1)
        USERS[new_id] = {"id": new_id, "name": name, "email": email}
        return User(id=new_id, name=name, email=email, posts=[])

# --- アプリ組み立て ---
schema = strawberry.Schema(query=Query, mutation=Mutation)
graphql_app = GraphQLRouter(schema)

app = FastAPI()
app.include_router(graphql_app, prefix="/graphql")

# http://localhost:8000/graphql でブラウザから GraphiQL IDE が使える
```

---

## Subscription（リアルタイム更新）

WebSocket を使ってサーバーからリアルタイムにデータをプッシュします。

```python
import asyncio
import strawberry
from typing import AsyncGenerator

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def count_up(self) -> AsyncGenerator[int, None]:
        for i in range(10):
            yield i
            await asyncio.sleep(1)

schema = strawberry.Schema(
    query=Query,
    mutation=Mutation,
    subscription=Subscription
)
```

```graphql
# クライアントから
subscription {
  countUp
}
# → 1秒ごとに 0, 1, 2, ... が届く
```

---

## N+1 問題と DataLoader

GraphQL でよく起きるパフォーマンス問題です。

!!! info ""
    ユーザー一覧 (10件) を取得すると…
    → ユーザー 1 の投稿を取得
    → ユーザー 2 の投稿を取得
    … (10回のDB クエリ)
    = 合計 11 回のクエリ（N+1 問題）
```python
from strawberry.dataloader import DataLoader

async def load_posts_by_user_ids(user_ids: list[str]) -> list[list[Post]]:
    # user_ids をまとめて 1 回のクエリで取得
    all_posts = await db.fetch_posts_for_users(user_ids)
    posts_by_user = {uid: [] for uid in user_ids}
    for post in all_posts:
        posts_by_user[post["user_id"]].append(post)
    return [posts_by_user[uid] for uid in user_ids]

# DataLoader がリクエストをバッチにまとめる
posts_loader = DataLoader(load_fn=load_posts_by_user_ids)

@strawberry.type
class User:
    id: strawberry.ID

    @strawberry.field
    async def posts(self, info: strawberry.types.Info) -> list[Post]:
        return await info.context["posts_loader"].load(str(self.id))
```

---

## 確認問題

1. REST の Over-fetching と Under-fetching をそれぞれ具体例で説明してください。
2. Query と Mutation の違いを説明してください。
3. N+1 問題とは何ですか？DataLoader はどうやってこれを解決しますか？

---

## 関連ページ

- [Web / API設計](WebAPI設計.md) — REST の設計原則との比較
- [FastAPI](FastAPI.md) — GraphQL のベースとなる Python Web フレームワーク
- [WebSocket・リアルタイム通信](WebSocket.md) — Subscription が内部で使う技術
- [認証・認可](認証・認可.md) — GraphQL API への JWT 認証
