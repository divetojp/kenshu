# TypeScript

## TypeScript とは

**JavaScript に「型」を追加したプログラミング言語** です。Microsoft が開発しており、JavaScript の上位互換です。

TypeScript で書いたコードは、最終的に JavaScript に変換（ **トランスパイル**）されてブラウザで動きます。

### なぜ TypeScript を使うのか

> **情報工学メモ：静的型付けと動的型付けの違い、コンパイルとトランスパイルの違い**  
> プログラミング言語には「型をいつチェックするか」で2種類あります。 **静的型付け言語** （TypeScript・Java・C など）は実行前のコンパイル時に型を検査するため、バグを早期発見できます。 **動的型付け言語** （JavaScript・Python など）は実行時に型が決まるため、柔軟に書ける反面、型起因のバグが実行して初めて発覚します。なお、 **コンパイル** は「ソースコードを機械語や中間形式に変換すること」全般を指し、 **トランスパイル** は「同じ抽象レベルの別の言語に変換すること」（TypeScript → JavaScript など）を特に指します。TypeScript はブラウザで直接動かせないため、実行前に JavaScript へのトランスパイルが必要です。

JavaScript では、変数にどんな型の値でも入れられます。

次の JavaScript 例では、最初は数値だった `price` に、あとから文字列を入れています。実行するまで問題に気づきにくく、計算処理でバグになる可能性があります。

```js
// JavaScript：型がないので間違いに気づきにくい
let price = 1000;
price = "千円";  // 文字列を入れても文句を言わない → バグの温床
```

TypeScript では型を宣言することで、間違いをコードを書いている段階で検出できます。

TypeScript では、`price` は `number` だと宣言しているため、文字列を代入しようとすると型エラーになります。これは面倒な制約ではなく、実行前にバグを知らせる仕組みです。

```ts
// TypeScript：型を指定するとエラーを即座に検出
let price: number = 1000;
price = "千円";  // → エラー！number 型に string は入れられない
```

Astro は TypeScript を標準でサポートしており、`.astro` ファイル内でそのまま使えます。

---

## はじめて読む人へ

TypeScript は、JavaScript に型の仕組みを加えた言語です。変数や関数がどんな形の値を扱うかを先に書くことで、大きなコードでもミスを見つけやすくします。


### 読む前に押さえること

- 型は、値の形に名前を付ける仕組みです。
- 型エラーは失敗ではなく、実行前にバグを教えてくれる通知です。
- Astro やフロントエンド開発では、データ構造を明確にするために型が役立ちます。

### 読み終えたら説明できること

- JavaScript と TypeScript の違いを説明できる。
- 基本的な型、Union 型、interface を読める。
- 型エラーから修正すべき場所を推測できる。

---


> **Python で言うと：**  
> Python 3.5 以降の型ヒント `name: str` に相当します。Python の型ヒントは実行時に無視されますが、TypeScript の型チェックはコンパイル時に強制されます。

## 基本的な型

型は、値がどのような種類かを表すラベルです。TypeScript では、変数名の後ろに `: 型名` を付けて、その変数が受け取れる値を決めます。

```ts
// 基本型
const name: string = "近江テック";      // 文字列（Python: str）
const age: number = 3;                  // 数値（Python: int / float）
const isOpen: boolean = true;           // 真偽値（Python: bool）

// 配列
const items: string[] = ["Git", "GitHub", "Astro"];
const scores: number[] = [80, 90, 75];

// null になる可能性がある場合
const title: string | null = null;      // string か null
```

`string`、`number`、`boolean` は基本型です。`string[]` は文字列の配列を表します。`string | null` は、文字列または `null` のどちらかが入る可能性がある、という意味です。

---

## 型推論

値から型を自動的に推測してくれるため、毎回型を書く必要はありません。

型推論とは、TypeScript が代入された値から型を判断する仕組みです。明らかに文字列や数値だと分かる場合は、毎回型を書かなくても型チェックが働きます。

```ts
const name = "近江テック";    // 自動的に string と推論
const age = 3;               // 自動的に number と推論

// 明示的に型を書かなくても、間違いは検出してくれる
age = "三歳";  // → エラー！
```

型が明らかな場合は書かなくてよく、曖昧な場合だけ書くのが実際の運用です。

関数の引数、API から受け取るデータ、空配列から始める変数などは型が曖昧になりやすいため、明示的に型を書くと安全です。

---

## インターフェース（オブジェクトの型定義）

複雑なオブジェクトの構造を定義するには **インターフェース** を使います。

> **Python で言うと：**  
> Python の `TypedDict`（辞書の型定義）や `@dataclass` に相当します。  
> ```python
> from typing import TypedDict
> class NewsItem(TypedDict):
>     title: str
>     date: str
>     published: bool
> ```

`interface` は、オブジェクトが持つべきプロパティ名と型を定義します。API レスポンス、記事データ、ユーザー情報など、形が決まっているデータに使います。

```ts
interface NewsItem {
  title: string;
  date: string;
  category: string;
  published: boolean;
}

const news: NewsItem = {
  title: "春のオープンキャンパス",
  date: "2024-03-01",
  category: "イベント",
  published: true,
};

// title プロパティが抜けているとエラーになる
```

`const news: NewsItem` と書くことで、`news` は `title`、`date`、`category`、`published` を必ず持つ必要があります。プロパティ名の打ち間違いや、必要な項目の入れ忘れを早く見つけられます。

---

## 関数の型定義

引数と戻り値に型をつけます。

関数の型定義では、「何を受け取り、何を返すか」を明示します。関数は他の場所から何度も呼ばれるため、入力と出力の型が分かると使い方を間違えにくくなります。

```ts
// 引数の型と戻り値の型を指定
function greet(name: string): string {
  return `こんにちは、${name}さん！`;
}

// 戻り値がない場合は void
function logMessage(message: string): void {
  console.log(message);
}

// オプション引数（?をつけると省略可能）
function createTitle(title: string, suffix?: string): string {
  return suffix ? `${title} - ${suffix}` : title;
}
```

`suffix?: string` の `?` は省略可能を表します。戻り値がない関数は `void` と書きます。ログ出力や DOM 操作のように、結果を返さず副作用だけを起こす関数で使います。

---

## Astro での TypeScript

Astro のフロントマターでは TypeScript をそのまま使えます。

Astro では、コンポーネントに渡される値を `Props` として型定義できます。これにより、ページやコンポーネントを使う側が必要な値を渡しているか確認できます。

```astro
---
interface Props {
  title: string;
  description?: string;
}

const { title, description = "説明なし" } = Astro.props;
---

<h1>{title}</h1>
<p>{description}</p>
```

`Astro.props` は、親コンポーネントやページから渡された値です。`description = "説明なし"` は、値が渡されなかった場合のデフォルト値です。

### コンテンツコレクションの型定義

`src/content/config.ts` でコンテンツの型を定義できます。

コンテンツコレクションでは、Markdown のフロントマターにどんな項目が必要かをスキーマとして定義します。記事データの入力ミスをビルド時に検出できます。

```ts
import { defineCollection, z } from 'astro:content';

const news = defineCollection({
  schema: z.object({
    title: z.string(),
    date: z.date(),
    category: z.string(),
    published: z.boolean().default(true),
  }),
});

export const collections = { news };
```

これにより、Markdown のフロントマターの型チェックが自動で行われます。

`z.string()` は文字列、`z.date()` は日付、`z.boolean().default(true)` は省略時に `true` を入れる真偽値です。コンテンツが増えるほど、このような型チェックが保守性を支えます。

---

## 型エラーの読み方

型エラーは、プログラムを実行する前に「値の形が期待と違う」と教えてくれる仕組みです。最初は赤い表示が怖く見えますが、実行時のバグを前倒しで見つけていると考えてください。

読むときは、エラー文の中から「期待されている型」と「実際に渡した型」を探します。多くの場合、どちらかの型定義が間違っているか、データの形を変換し忘れています。

TypeScript のエラーメッセージは最初難しく見えますが、パターンを覚えると読めるようになります。

次のエラー例では、左側が実際の型、右側が期待されている型です。エラー文を読むときは「何をどこへ渡そうとしているか」を確認します。

```
Type 'string' is not assignable to type 'number'
```
→ `number` 型の変数に `string` を入れようとしています

```
Property 'title' is missing in type '{}' but required in type 'NewsItem'
```
→ `NewsItem` 型に必須の `title` プロパティが足りていません

```
Argument of type 'null' is not assignable to parameter of type 'string'
```
→ `null` かもしれない値を、`null` を受け付けない関数に渡しています  
→ `if (value !== null)` でチェックしてから使うか、型を `string | null` にします

---

## Union 型・Literal 型

### Union 型（複数の型のいずれか）

`|`（パイプ）で複数の型を組み合わせます。

> **Python で言うと：**  
> `string | null` は Python の `Optional[str]`（または `str | None`）に相当します。Python 3.10 以降は `str | int` のように書けます。

Union 型は、値が複数の候補のどれかになることを表します。API から値が返ることもあれば `null` のこともある、状態がいくつかの文字列に限られる、という場面で使います。

```ts
// string か null のいずれか
let title: string | null = null;
title = "記事タイトル";  // OK
title = 42;             // → エラー！

// 特定の文字列だけ許可（Literal 型）
type Status = "draft" | "published" | "archived";
let status: Status = "published";
status = "deleted";  // → エラー！"deleted" は Status に含まれない
```

Astro のコンテンツ管理でよく使います。

```ts
// Literal 型でカテゴリを限定する
type Category = "イベント" | "お知らせ" | "採用";

interface NewsItem {
  title: string;
  category: Category;  // この3つ以外は型エラー
}
```

Literal 型で状態を限定すると、存在しない状態名を誤って使うミスを防げます。`"deleted"` がエラーになるのは、許可された候補に含まれていないからです。

### Intersection 型（複数の型をすべて満たす）

`&` で型を合成します。

Intersection 型は、複数の型を合体させ、すべてのプロパティを持つ型を作ります。共通の型を組み合わせて新しいデータ構造を作るときに使います。

```ts
interface Named {
  name: string;
}

interface Dated {
  date: string;
}

type NewsEntry = Named & Dated;  // name と date 両方が必要

const entry: NewsEntry = {
  name: "イベント",
  date: "2024-03-01",
};
```

---

## tsconfig.json（TypeScript の設定）

プロジェクトルートにある `tsconfig.json` が TypeScript の動作設定です。Astro が自動生成するので基本的に触りませんが、重要なオプションを把握しておきましょう。

`tsconfig.json` は、TypeScript がどのくらい厳しく型チェックするか、どの形式の JavaScript に変換するか、import のパスをどう解釈するかを決める設定ファイルです。

```json
{
  "compilerOptions": {
    "strict": true,          // 厳格モード（推奨）。型チェックを最大限に有効化
    "target": "ES2022",      // 出力する JavaScript のバージョン
    "module": "ESNext",      // モジュール形式
    "jsx": "preserve",       // JSX の処理方法（Astro が設定）
    "baseUrl": ".",          // import のベースパス
    "paths": {
      "@components/*": ["src/components/*"],  // パスのエイリアス
      "@layouts/*": ["src/layouts/*"]
    }
  }
}
```

`strict: true` は、初学者には厳しく感じることがありますが、`null` や `any` に関するバグを早めに見つけてくれます。チーム開発では、型の曖昧さを減らすために有効にすることが多いです。

### `strict` モードで有効になる主な検査

| チェック | 意味 |
|---------|------|
| `strictNullChecks` | `null` / `undefined` の不正な使用を検出 |
| `noImplicitAny` | 型推論できない変数に `any` を暗黙的に使うことを禁止 |
| `strictFunctionTypes` | 関数の引数の型チェックを厳格化 |

---

## JavaScript との違い まとめ

| | JavaScript | TypeScript |
|-|-----------|-----------|
| 型宣言 | 不要 | 任意（推奨） |
| 型チェック | なし | コンパイル時にチェック |
| ファイル拡張子 | `.js` / `.jsx` | `.ts` / `.tsx` |
| Astro での使用 | 可 | 可（標準） |
| 学習コスト | 低い | やや高い |

---

## よくある疑問

**Q. TypeScript は必須？**  
A. 必須ではありませんが、Astro プロジェクトでは標準サポートされており、型安全性が上がるため使うことを推奨します。既存の JavaScript コードはそのまま動くので、徐々に型を追加していくことも可能です。

**Q. `.ts` ファイルはそのままブラウザで動く？**  
A. 動きません。`tsc`（TypeScript コンパイラ）や Astro のビルドプロセスで JavaScript に変換されます。

**Q. `any` 型って何？**  
A. すべての型を受け入れる型で、TypeScript の型チェックを無効化します。型が分からないときの逃げ道ですが、多用するとTypeScriptの意味がなくなるため避けましょう。

---


## 確認問題

1. TypeScript は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [JavaScript 基礎](JavaScript) — TypeScript の前提知識
- [Astro](Astro) — TypeScript を使う主な場所

---

[← ホームへ](Home)
