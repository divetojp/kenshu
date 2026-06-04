# React

UI を「コンポーネント」という小さな部品に分けて組み立てるための JavaScript ライブラリです。状態（state）が変わると自動的に画面が更新される仕組みを持ちます。現在フロントエンド開発で最も広く使われています。

---

## はじめて読む人へ

React を使うと、「どの要素をどう更新するか」を自分で書かずに、「この状態のとき画面はこう見えるべき」を宣言的に書けます。DOM 操作を手書きしなくてよくなります。

### 読む前に押さえること

- [JavaScript 基礎](JavaScript.md) の関数・配列・アロー関数を知っていること
- [TypeScript](TypeScript.md) の型アノテーション（省略可だが推奨）

### 読み終えたら説明できること

- コンポーネントと props の関係を説明できる
- `useState` でカウンターを実装できる
- `useEffect` の実行タイミングを説明できる

---

## セットアップ

```bash
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

Vite を使うと TypeScript + React の環境が数十秒で起動します。

---

## コンポーネント

React アプリは**コンポーネント**の組み合わせで作ります。コンポーネントは「UI の一部分を返す関数」です。

```tsx
// src/components/Greeting.tsx
type Props = {
  name: string;
};

export function Greeting({ name }: Props) {
  return <h1>こんにちは、{name}さん！</h1>;
}
```

```tsx
// src/App.tsx
import { Greeting } from "./components/Greeting";

export function App() {
  return (
    <div>
      <Greeting name="田中" />
      <Greeting name="山田" />
    </div>
  );
}
```

`<Greeting name="田中" />` という記法が **JSX** です。JavaScript の中に HTML に似た構文を書けます。`{}` の中に JavaScript の式を埋め込めます。

---

## props（親から子へデータを渡す）

**props** はコンポーネントに渡す引数です。親コンポーネントから子コンポーネントへ、一方向にのみデータが流れます。

```tsx
type CardProps = {
  title: string;
  score: number;
  isHighScore?: boolean;  // ? は省略可能
};

export function Card({ title, score, isHighScore = false }: CardProps) {
  return (
    <div style={{ border: "1px solid gray", padding: "1rem" }}>
      <h2>{title}</h2>
      <p>スコア: {score}</p>
      {isHighScore && <span>🏆 ハイスコア</span>}
    </div>
  );
}
```

`{isHighScore && <span>...</span>}` は「`isHighScore` が true のときだけ表示する」という条件付きレンダリングです。

---

## useState（状態管理）

コンポーネントが「変わる値」を持つには `useState` を使います。状態が変わると React が自動的に再レンダリングします。

```tsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);  // 初期値 0

  return (
    <div>
      <p>カウント: {count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(0)}>リセット</button>
    </div>
  );
}
```

`useState(0)` は `[現在の値, 値を更新する関数]` を返します。`setCount(新しい値)` を呼ぶと再レンダリングが起きます。

**直接変更は動きません：**

```tsx
// NG: count を直接変えても再レンダリングされない
count = count + 1;

// OK: setState 関数を使う
setCount(count + 1);
```

---

## フォームと状態

テキスト入力を React で管理するパターンです。

```tsx
import { useState } from "react";

export function SearchBox() {
  const [query, setQuery] = useState("");

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="検索ワードを入力"
      />
      <p>入力中: {query}</p>
    </div>
  );
}
```

`onChange` で `setQuery` を呼ぶことで、入力のたびに `query` が更新されます。`value={query}` で input の表示内容を React が管理します（制御コンポーネント）。

---

## リストのレンダリング

配列から要素の一覧を表示するには `.map()` を使います。

```tsx
type Todo = {
  id: number;
  text: string;
  done: boolean;
};

export function TodoList({ todos }: { todos: Todo[] }) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id} style={{ textDecoration: todo.done ? "line-through" : "none" }}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**`key` は必須です。** React がどの要素が変わったかを判断するために使います。一意な ID を渡してください（配列のインデックスは避けるのが基本）。

---

## useEffect（副作用）

API 呼び出し、タイマー、イベントリスナーの登録など「レンダリング以外の処理」は `useEffect` で行います。

```tsx
import { useState, useEffect } from "react";

export function UserProfile({ userId }: { userId: number }) {
  const [user, setUser] = useState<{ name: string } | null>(null);

  useEffect(() => {
    // userId が変わるたびに実行される
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then((data) => setUser(data));
  }, [userId]);  // 依存配列: ここに書いた値が変わると再実行

  if (!user) return <p>読み込み中...</p>;
  return <p>ユーザー: {user.name}</p>;
}
```

| 依存配列 | 実行タイミング |
|---------|-------------|
| `[]` | マウント時のみ（1回）|
| `[userId]` | マウント時 + `userId` 変更時 |
| なし | 毎レンダリング後（ほぼ使わない）|

---

## コンポーネントの分割と構成

```tsx
// 小さなコンポーネントを組み合わせてページを作る例
export function DashboardPage() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <Header title="ダッシュボード" />
      <main>
        <Counter count={count} onIncrement={() => setCount(count + 1)} />
        <UserList />
      </main>
    </div>
  );
}
```

**コンポーネントを小さく保つ指針：**
- 1 コンポーネントは 1 つのことだけを担う
- 100 行を超えたら分割を検討する
- props が 5 つ以上になったら設計を見直す

---

## 確認問題

1. `useState` と通常の変数の違いを「再レンダリング」という言葉を使って説明してください。
2. `useEffect` の依存配列を `[]` にすると何が起きますか？
3. props はなぜ「一方向」にしか流れないのですか？双方向にしたい場合はどうしますか？

---

## 関連ページ

- [TypeScript](TypeScript.md) — React の型付け（Props 型・イベント型）
- [JavaScript基礎](JavaScript.md) — `.map()` `.filter()` など React でよく使う記法
- [関数型プログラミング](関数型プログラミング.md) — 状態を直接変えない考え方
- [FastAPI](FastAPI.md) — React から呼び出す API の実装
