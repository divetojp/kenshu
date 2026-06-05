# Tailwind CSS

「クラス名を HTML に直接書いてスタイルを当てる」ユーティリティファーストの CSS フレームワークです。`flex`・`p-4`・`text-blue-500` のような単機能クラスを組み合わせて UI を作ります。現在 React・Next.js・Astro と組み合わせて使われる CSS の主流です。

---

## はじめて読む人へ

従来の CSS は「`.card { padding: 1rem; }` のように名前をつけてスタイルを書く」スタイルでした。Tailwind は逆に「HTML の中に `class="p-4"` と書く」アプローチです。最初は冗長に見えますが、「ファイルをまたいで CSS を探す」手間がなくなり、コンポーネントとスタイルが一箇所に集まります。

### 読む前に押さえること

- [HTML / CSS](HTML-CSS.md) の基本的なプロパティ（padding・margin・flex）
- [React](React.md) か [Next.js](Next.js.md) のどちらかを読んでいること（Tailwind を使う文脈として）

### 読み終えたら説明できること

- ユーティリティファーストとは何か、従来の CSS との違いを説明できる
- レスポンシブデザインを `sm:` `md:` `lg:` で実装できる
- Next.js プロジェクトに Tailwind を導入できる

---

## セットアップ（Next.js）

`create-next-app` で `--tailwind` を付けると自動設定されます。

```bash
npx create-next-app@latest my-app --typescript --tailwind --app
```

手動で追加する場合：

```bash
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```js
// tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
  content: [
    "./app/**/*.{ts,tsx}",
    "./components/**/*.{ts,tsx}",
  ],
  theme: {
    extend: {},
  },
  plugins: [],
} satisfies Config;
```

```css
/* app/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

## 基本クラスの体系

### スペーシング（padding・margin）

!!! info ""
    p-{n}   → padding: n × 0.25rem（すべての辺）
    px-{n}  → padding 左右
    py-{n}  → padding 上下
    pt-{n}  → padding 上のみ
    m-{n}   → margin（同様）
    
    p-0  = 0px       p-1  = 4px
    p-2  = 8px       p-4  = 16px
    p-6  = 24px      p-8  = 32px
    p-12 = 48px      p-16 = 64px

```tsx
<div className="p-4 mx-auto mt-8">
  <p className="px-6 py-3">テキスト</p>
</div>
```

### 幅・高さ

!!! info ""
    w-{n}      → width: n × 0.25rem
    w-full     → width: 100%
    w-screen   → width: 100vw
    w-1/2      → width: 50%
    w-1/3      → width: 33.333%
    max-w-xl   → max-width: 36rem（sm/md/lg/xl/2xl など）
    h-{n}      → height（同様）

### テキスト

```
text-xs   → 12px    text-sm   → 14px
text-base → 16px    text-lg   → 18px
text-xl   → 20px    text-2xl  → 24px
text-3xl  → 30px    text-4xl  → 36px

font-normal   → font-weight: 400
font-medium   → 500
font-semibold → 600
font-bold     → 700

text-left / text-center / text-right
```

### 色

!!! info ""
    text-{color}-{shade}        → テキスト色
    bg-{color}-{shade}          → 背景色
    border-{color}-{shade}      → ボーダー色
    
    shade: 50 100 200 300 400 500 600 700 800 900 950
    color: slate gray red orange yellow green blue indigo violet pink
    
    例:
      text-blue-600   → 中程度の青
      bg-gray-100     → 薄いグレー背景
      text-white      → 白
      bg-black        → 黒

---

## Flexbox と Grid

```tsx
{/* Flexbox */}
<div className="flex items-center justify-between gap-4">
  <span>左</span>
  <span>右</span>
</div>

{/* 縦並び */}
<div className="flex flex-col gap-2">
  <div>上</div>
  <div>下</div>
</div>

{/* Grid（3カラム） */}
<div className="grid grid-cols-3 gap-4">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>
```

| クラス | 意味 |
|--------|------|
| `flex` | display: flex |
| `items-center` | align-items: center |
| `justify-between` | justify-content: space-between |
| `justify-center` | justify-content: center |
| `gap-4` | gap: 1rem |
| `flex-col` | flex-direction: column |
| `grid-cols-3` | grid-template-columns: repeat(3, 1fr) |

---

## レスポンシブデザイン

`sm:` `md:` `lg:` `xl:` プレフィックスでブレークポイントを指定します。**モバイルファースト**設計なので、プレフィックスなしがモバイル向けです。

| プレフィックス | ブレークポイント |
|-------------|-------------|
| （なし）    | 0px〜（全デバイス） |
| `sm:`       | 640px 以上 |
| `md:`       | 768px 以上 |
| `lg:`       | 1024px 以上 |
| `xl:`       | 1280px 以上 |

```tsx
{/* モバイル: 1列、タブレット以上: 2列、PC: 3列 */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  ...
</div>

{/* モバイル: 非表示、md以上: 表示 */}
<nav className="hidden md:flex gap-4">
  <a href="/">ホーム</a>
  <a href="/about">About</a>
</nav>

{/* テキストサイズをデバイスで変える */}
<h1 className="text-2xl md:text-4xl lg:text-5xl font-bold">
  タイトル
</h1>
```

---

## ホバー・フォーカス・ダークモード

```tsx
{/* ホバー時にスタイル変化 */}
<button className="bg-blue-500 hover:bg-blue-700 text-white px-4 py-2 rounded transition-colors">
  ボタン
</button>

{/* フォーカス時（アクセシビリティ） */}
<input className="border border-gray-300 focus:outline-none focus:ring-2 focus:ring-blue-500 rounded px-3 py-2" />

{/* ダークモード（tailwind.config で darkMode: 'class' の設定が必要） */}
<div className="bg-white dark:bg-gray-900 text-black dark:text-white">
  コンテンツ
</div>
```

---

## コンポーネントパターン

### カード

```tsx
function Card({ title, description }: { title: string; description: string }) {
  return (
    <div className="bg-white rounded-xl shadow-md p-6 hover:shadow-lg transition-shadow">
      <h2 className="text-xl font-semibold text-gray-800 mb-2">{title}</h2>
      <p className="text-gray-600 text-sm leading-relaxed">{description}</p>
    </div>
  );
}
```

### ボタン（バリアント）

```tsx
type ButtonProps = {
  variant?: "primary" | "secondary" | "danger";
  children: React.ReactNode;
  onClick?: () => void;
};

const variants = {
  primary:   "bg-blue-600 hover:bg-blue-700 text-white",
  secondary: "bg-gray-200 hover:bg-gray-300 text-gray-800",
  danger:    "bg-red-600 hover:bg-red-700 text-white",
};

function Button({ variant = "primary", children, onClick }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      className={`${variants[variant]} px-4 py-2 rounded-lg font-medium transition-colors`}
    >
      {children}
    </button>
  );
}
```

### バッジ・タグ

```tsx
<span className="inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium bg-green-100 text-green-800">
  完了
</span>
```

---

## `@apply` によるクラス抽出

同じクラスの組み合わせを繰り返す場合、`@apply` で CSS クラスにまとめられます。ただし**多用すると Tailwind のメリットが薄れる**ため、ボタンなど本当に繰り返すコンポーネントのみに限定します。

```css
/* globals.css */
@layer components {
  .btn-primary {
    @apply bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg font-medium transition-colors;
  }
}
```

```tsx
<button className="btn-primary">送信</button>
```

---

## 確認問題

1. `className="p-4 md:p-8"` はどのような挙動をしますか？
2. `flex items-center justify-between` を CSS プロパティで書き直してください。
3. Tailwind のクラスは `class` ではなく `className` と書く理由は何ですか？（JSX の仕様）

---

## 関連ページ

- [HTML / CSS](HTML-CSS.md) — CSS の基礎（display・box model・flex）
- [React](React.md) — コンポーネントに Tailwind クラスを適用する
- [Next.js](Next.js.md) — Next.js + Tailwind の標準構成
- [Astro](Astro.md) — Astro + Tailwind での静的サイト
