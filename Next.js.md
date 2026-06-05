# Next.js

React でアプリを作るための**フレームワーク**です。React 単体では「JavaScript でUIを書く仕組み」しかありませんが、Next.js を使うと「ページのルーティング」「サーバーでのレンダリング」「API の作成」が一括で揃います。現在フロントエンド開発の事実上の標準です。

---

## はじめて読む人へ

React だけでアプリを作ると「ページ遷移をどう管理するか」「データをどのタイミングで取得するか」「SEO をどうするか」を自分で設計しなければなりません。Next.js はこれらの問題に対する答えをフレームワークとして提供します。

### 読む前に押さえること

- [React](React.md) のコンポーネント・useState・useEffect を知っていること
- [TypeScript](TypeScript.md) の基本的な型アノテーションを知っていること

### 読み終えたら説明できること

- ファイルベースルーティングの仕組みを説明できる
- Server Components と Client Components の違いを説明できる
- SSR・SSG・ISR の違いとそれぞれの用途を説明できる

---

## セットアップ

```bash
npx create-next-app@latest my-app --typescript --tailwind --app
cd my-app
npm run dev
```

`--app` フラグで App Router を使います（現在の推奨）。`http://localhost:3000` でアプリが起動します。

---

## ファイルベースルーティング

`app/` ディレクトリのファイル構造が URL に直接対応します。

!!! info ""
    app/
    ├── page.tsx          → /
    ├── about/
    │   └── page.tsx      → /about
    ├── blog/
    │   ├── page.tsx      → /blog
    │   └── [slug]/
    │       └── page.tsx  → /blog/any-article-name  ← 動的ルート
    └── layout.tsx        → 全ページ共通のレイアウト

```tsx
// app/page.tsx → http://localhost:3000/
export default function HomePage() {
  return <h1>トップページ</h1>;
}

// app/blog/[slug]/page.tsx → http://localhost:3000/blog/hello-world
export default function BlogPost({ params }: { params: { slug: string } }) {
  return <h1>記事: {params.slug}</h1>;
}
```

---

## Server Components と Client Components

App Router の最大の特徴です。**デフォルトですべてのコンポーネントは Server Components**（サーバーで実行）です。

| | Server Components | Client Components |
|--|-----------------|------------------|
| 実行場所 | サーバー（Next.js） | ブラウザ |
| データ取得 | `async/await` で直接 DB や API を呼べる | `useEffect` + `fetch` |
| useState / useEffect | 使えない | 使える |
| ファイルサイズ | JS をブラウザに送らない（高速）| JS をブラウザに送る |
| 使いどころ | 静的なUI・データ取得 | ボタン・フォーム・状態管理 |

```tsx
// app/users/page.tsx（Server Component）
// ← async で直接 fetch できる。ブラウザに JS を送らない
async function UsersPage() {
  const res = await fetch("https://jsonplaceholder.typicode.com/users");
  const users = await res.json();

  return (
    <ul>
      {users.map((u: { id: number; name: string }) => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}

export default UsersPage;
```

```tsx
// app/counter/page.tsx（Client Component）
// ← useState を使うには "use client" が必要
"use client";
import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>カウント: {count}</button>;
}
```

---

## データフェッチングと SSR / SSG / ISR

Next.js は「いつページを生成するか」を選べます。

| 方式 | 生成タイミング | 向いているページ |
|------|-------------|----------------|
| **SSG**（Static Site Generation）| ビルド時（1 回）| ブログ記事・ドキュメント |
| **SSR**（Server-Side Rendering）| リクエストごと | 常に最新データが必要なページ |
| **ISR**（Incremental Static Regeneration）| ビルド時 + N 秒ごとに再生成 | 更新頻度が低いが最新を返したい |
| **CSR**（Client-Side Rendering）| ブラウザで実行 | ユーザー固有データ（ログイン後） |

```tsx
// SSG: ビルド時に生成（デフォルト、fetch にオプションなし）
async function Page() {
  const data = await fetch("https://api.example.com/data");
  // ...
}

// SSR: リクエストのたびに実行
async function Page() {
  const data = await fetch("https://api.example.com/data", {
    cache: "no-store",  // ← これで SSR になる
  });
}

// ISR: 60 秒ごとに再生成
async function Page() {
  const data = await fetch("https://api.example.com/data", {
    next: { revalidate: 60 },
  });
}
```

---

## API Routes

`app/api/` ディレクトリに置いたファイルが API エンドポイントになります。フロントエンドと同じリポジトリで簡単なバックエンドを作れます。

```tsx
// app/api/hello/route.ts → GET /api/hello
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({ message: "こんにちは" });
}

export async function POST(request: Request) {
  const body = await request.json();
  return NextResponse.json({ received: body });
}
```

```tsx
// app/api/users/[id]/route.ts → GET /api/users/123
export async function GET(
  request: Request,
  { params }: { params: { id: string } }
) {
  const user = await db.getUser(params.id);  // DB アクセス（サーバー側）
  return NextResponse.json(user);
}
```

---

## layout.tsx（共通レイアウト）

```tsx
// app/layout.tsx（全ページ共通）
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "My App",
  description: "Next.js アプリ",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="ja">
      <body>
        <nav>
          <a href="/">ホーム</a>
          <a href="/about">About</a>
        </nav>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

`children` に各ページのコンテンツが入ります。ヘッダー・フッターはここに書くと全ページに適用されます。

---

## React 単体 vs Next.js

!!! info ""
    React だけ:
      - ルーティング → react-router-dom を別途インストール
      - データ取得 → useEffect + fetch（常に CSR）
      - SEO → 弱い（JS が実行されないと内容が見えない）
      - API → 別サーバーが必要（FastAPI 等）
    
    Next.js:
      - ルーティング → ファイル構造で自動
      - データ取得 → SSG/SSR/ISR を選べる
      - SEO → サーバーで HTML を作るので検索エンジンに届く
      - API → 同じリポジトリで作れる

---

## 確認問題

1. Server Components でできて Client Components でできないことは何ですか？逆は？
2. ブログサイトで「記事の一覧」ページには SSG・SSR・ISR のどれが適切ですか？理由も説明してください。
3. `app/products/[id]/page.tsx` にアクセスする URL の例を 2 つ書いてください。

---

## 関連ページ

- [React](React.md) — コンポーネント・useState・useEffect の基礎
- [TypeScript](TypeScript.md) — Props 型・API レスポンス型の定義
- [認証・認可](認証・認可.md) — Next.js での JWT・セッション管理
- [FastAPI](FastAPI.md) — Next.js から呼び出すバックエンド API
