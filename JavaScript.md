# JavaScript 基礎

## JavaScript とは

**Web ページに動きや処理を加えるためのプログラミング言語** です。

HTML が「構造」、CSS が「見た目」を担当するのに対して、JavaScript は「動き・処理」を担当します。

現在では Web ブラウザの中だけでなく、サーバー（Node.js）やデスクトップアプリでも使われています。

---

## はじめて読む人へ

JavaScript は、Webページに動きを付けるための言語です。ボタンを押したら表示を変える、APIからデータを取る、入力内容に反応する、といった処理を担当します。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- HTML/CSS が画面を作り、JavaScript が動きを作ります。
- 変数・条件分岐・配列・関数は Python と似た考え方で読めます。
- 非同期処理は、通信やファイル読み込みの待ち時間を扱うために必要です。

### 読み終えたら説明できること

- 基本文法を Python と比較しながら読める。
- DOM 操作が何をしているか説明できる。
- async/await の目的を理解できる。

---

## まず動かしてみる

JavaScript はブラウザだけあれば今すぐ試せます。インストール不要です。

**手順：**

1. Chrome または Safari を開く
2. `F12`（Mac は `Option + Command + I`）で開発者ツールを開く
3. 「Console」タブをクリック
4. 以下を入力して `Enter`

最初のコードでは、JavaScript がブラウザのコンソール上で実行できることを確認します。`console.log` は、処理結果や変数の中身を開発者が確認するための出力です。

```js
console.log("Hello, World!");
```

`Hello, World!` と表示されれば成功です。このページのコード例はすべてコンソールで試せます。

> **`console.log()` とは？**  
> 値をコンソールに出力する命令です。プログラムの動作確認（デバッグ）に使います。

---

## 変数

**変数** とは、値に名前をつけて保存する箱です。

JavaScript では、変数を作るときに `const` か `let` を付けます。基本は「後から別の値を入れ直さないなら `const`、入れ直すなら `let`」です。

```js
const name = "田中 太郎";   // 変更しない値
let count = 0;              // 後で変更する値

console.log(name);   // "田中 太郎"
console.log(count);  // 0
```

| キーワード | 意味 | 使うとき |
|-----------|------|---------|
| `const` | 定数（変更不可） | ほとんどの場面 |
| `let` | 変数（変更可能） | カウンターなど値が変わる場合 |

`const` は「中身を絶対に変えられない」というより、「同じ変数名に別の値を再代入できない」という意味です。配列やオブジェクトの場合、中の要素やプロパティを変更できる点には注意してください。

```js
// const は再代入しようとするとエラーになる
const x = 10;
x = 20;  // TypeError: Assignment to constant variable.

// let は変更できる
let y = 10;
y = 20;  // OK
```

> **Python で言うと：**  
> Python に `const` / `let` の区別はなく、すべて `変数名 = 値` で書きます。JavaScript では「後で変更するか」を宣言時に明示します。

---

## データ型

JavaScript の主なデータ型です。

データ型は、値の種類を表します。数値、文字列、真偽値、値がない状態などを区別できると、比較や条件分岐の結果を予想しやすくなります。

```js
// 数値（整数と小数を区別しない）
const age = 21;
const pi = 3.14;

// 文字列
const message = "こんにちは";

// 真偽値
const isOpen = true;
const isClosed = false;

// null（意図的に「値がない」を表す）
const result = null;

// undefined（まだ値が入っていない）
let score;
console.log(score);  // undefined

// typeof で型を確認できる
console.log(typeof age);      // "number"
console.log(typeof message);  // "string"
console.log(typeof isOpen);   // "boolean"
```

`undefined` は「まだ値が代入されていない」状態、`null` は「値がないことを意図的に入れた」状態です。どちらも空っぽに見えますが、意味が違います。

> **Python で言うと：**  
> Python の `int` / `float` が JS では両方 `number` 型です。Python の `None` が JS では `null` に相当します。

---

## 文字列

文字列は、画面に表示するテキストや、API から受け取るメッセージ、ユーザー入力などを扱う型です。JavaScript では `+` で文字列を連結できます。

```js
const first = "田中";
const last = "太郎";

// 文字列の結合
const full = first + " " + last;
console.log(full);  // "田中 太郎"

// 長さ
console.log(full.length);  // 5
```

`length` は文字列の長さを表すプロパティです。関数のように `length()` と呼ぶのではなく、値の属性として `full.length` と書きます。

### テンプレートリテラル

バッククォート（`` ` ``）を使うと、文字列の中に変数を `${}` で埋め込めます。

テンプレートリテラルは、HTML の断片やメッセージ文を作るときに便利です。`+` で何度も連結するより、文章の形を保ったまま変数を埋め込めます。

```js
const name = "太郎";
const age = 21;

const greeting = `こんにちは、${name}さん！ ${age}歳ですね。`;
console.log(greeting);
// → "こんにちは、太郎さん！ 21歳ですね。"
```

> **Python で言うと：**  
> Python の f-string（`f"こんにちは{name}"`）とほぼ同じ書き方です。

---

## 演算子

演算子は、計算、比較、論理判断を行うための記号です。比較演算子の結果は `true` または `false` になり、そのまま `if` 文の条件として使えます。

```js
// 算術演算子
console.log(10 + 3);   // 13
console.log(10 - 3);   // 7
console.log(10 * 3);   // 30
console.log(10 / 3);   // 3.333...
console.log(10 % 3);   // 1  （余り）
console.log(2 ** 3);   // 8  （べき乗）

// 比較演算子（結果は true か false）
console.log(5 > 3);    // true
console.log(5 < 3);    // false
console.log(5 === 5);  // true  （値と型が等しい）
console.log(5 !== 3);  // true  （等しくない）

// 論理演算子
console.log(true && false);  // false （かつ）
console.log(true || false);  // true  （または）
console.log(!true);          // false （否定）
```

`===` は値と型の両方を比較します。JavaScript では型変換が自動で起きる場面があるため、予想外の比較結果を避けるには `===` を使います。

> **`===` と `==` の違い：**  
> `==` は型を無視して比較するため予期しない結果になることがあります（例：`"5" == 5` が `true`）。JavaScript では常に `===` を使う習慣をつけましょう。

---

## 条件分岐（if / else）

ある条件が成立するときだけ処理を実行します。

条件分岐は、ユーザーの入力、API の結果、画面の状態に応じて処理を変えるための基本です。JavaScript ではブロックを `{}` で囲みます。

```js
const score = 75;

if (score >= 80) {
  console.log("優");
} else if (score >= 60) {
  console.log("良");  // ← これが実行される
} else {
  console.log("不可");
}
```

条件は上から順番に評価されます。この例では `score >= 80` は偽、`score >= 60` は真なので「良」が表示され、残りの `else` には進みません。

### 三項演算子

1行で条件分岐を書けます。

三項演算子は、条件に応じて値を選ぶための短い書き方です。処理を実行するというより、変数に入れる値を切り替えるときに向いています。

```js
const isAdult = age >= 18 ? "成人" : "未成年";
// 条件 ? 真のとき : 偽のとき
```

> **Python で言うと：**  
> Python の `"成人" if age >= 18 else "未成年"` と同じです。順番が違う点に注意。

---

## 繰り返し（for ループ）

繰り返しは、同じ処理を複数のデータに対して行うための構文です。配列の要素を順番に表示する、API から受け取った一覧を画面に並べる、といった場面で使います。

```js
// 基本の for ループ
for (let i = 0; i < 5; i++) {
  console.log(i);  // 0, 1, 2, 3, 4
}

// 配列の要素を順番に処理する（for...of）
const fruits = ["りんご", "バナナ", "みかん"];

for (const fruit of fruits) {
  console.log(fruit);
}
// → "りんご"
// → "バナナ"
// → "みかん"
```

基本の `for` ループは、カウンター `i` を自分で増やします。`for...of` は配列の要素を直接取り出せるため、一覧処理ではこちらのほうが読みやすいことが多いです。

> **Python で言うと：**  
> `for fruit in fruits:` と全く同じ発想です。JavaScript では `for (const fruit of fruits)` と書きます。

---

## 配列

**配列** とは、複数の値を順番に並べたリストです。

配列は、ユーザー一覧、商品一覧、点数一覧のような「複数の同種のデータ」を扱うための基本構造です。要素は 0 番目から数えます。

```js
const members = ["田中", "山田", "佐藤"];

// 特定の要素を取得（0始まり）
console.log(members[0]);  // "田中"
console.log(members[2]);  // "佐藤"

// 長さ
console.log(members.length);  // 3

// 末尾に追加
members.push("鈴木");
console.log(members);  // ["田中", "山田", "佐藤", "鈴木"]
```

`members[0]` が最初の要素です。`push` は末尾に要素を追加します。`const members` と宣言していても、配列そのものへの再代入ができないだけで、中身の追加はできます。

> **Python で言うと：**  
> JavaScript の配列（`Array`）は Python のリスト（`list`）に相当します。`.push()` は Python の `.append()` です。

### よく使う配列メソッド

**.map()：各要素を変換して新しい配列を作る**

`.map()` は、配列の各要素を別の値に変換して、新しい配列を作ります。元の配列を直接変更しないため、データの流れを追いやすい書き方です。

```js
const scores = [50, 70, 90];
const doubled = scores.map((score) => score * 2);
console.log(doubled);  // [100, 140, 180]
```

**.filter()：条件に合う要素だけ取り出す**

`.filter()` は、条件を満たす要素だけを残します。一覧から検索条件に合うデータを取り出すときによく使います。

```js
const scores = [45, 82, 67, 91, 38];
const passed = scores.filter((score) => score >= 70);
console.log(passed);  // [82, 91]
```

**.find()：条件に合う最初の要素を1つ取り出す**

`.find()` は、条件に合う最初の 1 件だけを返します。ID が一致するユーザーを探す、特定の設定を探す、といった用途に向いています。

```js
const members = [
  { name: "田中", role: "admin" },
  { name: "山田", role: "editor" },
];

const admin = members.find((m) => m.role === "admin");
console.log(admin.name);  // "田中"
```

**.reduce()：配列を1つの値にまとめる**

`.reduce()` は、配列の要素を積み上げて 1 つの値にまとめます。合計、最大値、グループ化などに使えますが、初学者には少し読みにくいため、まずは合計の例で考えます。

```js
const prices = [500, 1200, 800];
const total = prices.reduce((sum, price) => sum + price, 0);
console.log(total);  // 2500
```

> **Python で言うと：**  
> `.map(fn)` → リスト内包表記 `[fn(x) for x in arr]`  
> `.filter(fn)` → `[x for x in arr if fn(x)]`  
> `.reduce(fn, init)` → `functools.reduce(fn, arr, init)`

---

## 関数

**関数** とは、まとまった処理に名前をつけて繰り返し使えるようにしたものです。

関数は、入力を受け取り、処理を行い、結果を返す部品です。画面のイベント処理、データ変換、API 通信など、JavaScript ではほとんどの処理を関数として整理します。

```js
// 関数の定義
function greet(name) {
  return `こんにちは、${name}さん！`;
}

// 関数の呼び出し
console.log(greet("田中"));  // "こんにちは、田中さん！"
console.log(greet("山田"));  // "こんにちは、山田さん！"
```

`greet("田中")` の `"田中"` が引数として `name` に入り、`return` の結果が呼び出し元に返ります。同じ処理を名前だけ変えて再利用できるのが関数の利点です。

### アロー関数（短く書く方法）

JavaScript ではよりコンパクトに関数を書ける「アロー関数」が使われます。

アロー関数は、関数を値として扱う場面でよく使います。特に `.map()` や `.filter()` のように、別の関数へ処理を渡すときに自然に登場します。

```js
// 通常の関数
function add(a, b) {
  return a + b;
}

// アロー関数（同じ意味）
const add = (a, b) => {
  return a + b;
};

// 処理が1行なら return と {} を省略できる
const add = (a, b) => a + b;

console.log(add(3, 5));  // 8
```

最後の `const add = (a, b) => a + b;` は、「a と b を受け取り、a + b を返す関数」を表します。処理が 1 行だけなら `{}` と `return` を省略できます。

`.map()` などのメソッドでよく登場するのがこのアロー関数です。

```js
// scores.map((score) => score * 2) の (score) => score * 2 がアロー関数
const doubled = [1, 2, 3].map((n) => n * 2);  // [2, 4, 6]
```

> **Python で言うと：**  
> Python の `lambda x: x * 2` に相当します。ただし JavaScript のアロー関数は複数行の処理も書けます。

### デフォルト引数

引数が渡されなかったときのデフォルト値を設定できます。

デフォルト引数は、「指定されなかった場合の標準値」を関数側で決めておく仕組みです。呼び出し側が毎回すべての値を渡さなくてもよくなります。

```js
function greet(name, greeting = "こんにちは") {
  return `${greeting}、${name}さん！`;
}

console.log(greet("田中"));             // "こんにちは、田中さん！"
console.log(greet("田中", "おはよう")); // "おはよう、田中さん！"
```

---

## オブジェクト

**オブジェクト** とは、関連するデータをまとめた構造です。`キー: 値` の形で書きます。

オブジェクトは、1 つの対象が持つ複数の情報をまとめるために使います。ユーザー、商品、記事、設定値など、Web アプリのデータはオブジェクトとして扱われることが多いです。

```js
const person = {
  name: "山田 太郎",
  age: 21,
  isStudent: true,
};

// 値の取り出し方（2通り）
console.log(person.name);      // "山田 太郎"
console.log(person["age"]);    // 21

// 値の変更
person.age = 22;
console.log(person.age);  // 22
```

ドット記法 `person.name` はよく使う基本形です。キー名が変数で決まる場合や、キーにハイフンなどが含まれる場合は、`person["age"]` のようなブラケット記法を使います。

> **Python で言うと：**  
> JavaScript のオブジェクト `{ key: value }` は Python の辞書（`dict`）`{"key": value}` に相当します。

### オブジェクトの配列

実際のデータはこの形がよく登場します。

API から返ってくるデータは、オブジェクトの配列であることが多いです。つまり「複数のレコードがあり、各レコードが名前付きの項目を持つ」という形です。

```js
const students = [
  { name: "田中", score: 85 },
  { name: "山田", score: 72 },
  { name: "佐藤", score: 91 },
];

// 全員の名前を表示
for (const student of students) {
  console.log(student.name);
}

// 80点以上の学生だけ取り出す
const highScorers = students.filter((s) => s.score >= 80);
console.log(highScorers);
// → [{ name: "田中", score: 85 }, { name: "佐藤", score: 91 }]
```

この例では、`students` が学生レコードの一覧、各 `{ name, score }` が 1 人分のデータです。`filter` を使うと、条件に合うレコードだけを新しい配列として取り出せます。

---

## 非同期処理（async / await）

JavaScript では、API 通信やファイル読み込みのように時間がかかる処理をよく扱います。これらを同期的に待ってしまうと、その間ページの操作が止まってしまいます。

非同期処理は、時間のかかる処理を待ちながらも、他の処理を止めないための仕組みです。`async` / `await` を使うと、非同期処理を上から順に読むような形で書けるため、Promise を直接つなぐより理解しやすくなります。

データの取得やファイル読み込みなど、**完了まで時間がかかる処理** を扱うときに使います。

> **なぜ必要？**  
> JavaScript はブラウザ上で **シングルスレッド** で動作します（同時に1つの処理しかできません）。サーバーへのデータ取得（数百ms かかる）を「待ちながら実行」すると、その間ブラウザ全体が固まってしまいます。  
> `async/await` を使うと「この処理が終わるまで待って」と指示しつつ、待っている間に他の処理を進められます。

次のコードでは、`fetch` で API にリクエストを送り、返ってきたレスポンスを JSON として読み取ります。`await` が付いている行は、非同期処理の完了を待ってから次へ進みます。

```js
// async をつけた関数の中で await が使える
async function fetchUserData(userId) {
  const response = await fetch(`https://api.example.com/users/${userId}`);
  const data = await response.json();
  return data;
}

// 呼び出し
fetchUserData(1).then((data) => {
  console.log(data);
});
```

`fetchUserData` はすぐにデータそのものを返すのではなく、将来完了する処理を表す Promise を返します。そのため、外側では `.then(...)` で結果を受け取っています。別の `async` 関数の中なら、`const data = await fetchUserData(1)` のようにも書けます。

> **Python で言うと：**  
> Python の `async def` / `await`（asyncio）とほぼ同じです。FastAPI もこの仕組みを使っています。

### エラーハンドリング（try / catch）

通信エラーなど、失敗する可能性がある処理をまとめて囲みます。

ネットワーク通信は、URL の間違い、サーバー障害、認証エラーなどで失敗します。失敗時の動きを決めておかないと、ユーザーには何も起きていないように見えたり、画面が壊れたりします。

```js
async function fetchData(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTPエラー: ${response.status}`);
    }
    const data = await response.json();
    return data;
  } catch (error) {
    // 失敗したときの処理
    console.error("データ取得に失敗しました:", error.message);
    return null;
  } finally {
    // 成功・失敗に関わらず必ず実行される
    console.log("処理完了");
  }
}
```

`response.ok` は HTTP ステータスが成功範囲かどうかを表します。失敗している場合は `throw new Error(...)` で例外を発生させ、`catch` でまとめて処理します。

---

## DOM 操作（ブラウザでのインタラクション）

**DOM**（Document Object Model）とは、ブラウザが HTML を解析して作るデータ構造です。JavaScript から DOM を操作することで、ページの見た目や動作をリアルタイムに変更できます。

> **仕組み：** ブラウザは HTML を読み込むと、要素の親子関係を木構造（DOM ツリー）として内部に持ちます。JavaScript はこのツリーを書き換えることで、再読み込みなしにページを更新できます。

### 要素を取得する

DOM 操作では、まず変更したい HTML 要素を JavaScript から取得します。取得した要素は変数に入れて、文字やスタイル、クラスを変更できます。

```js
// id で取得（1つ）
const header = document.getElementById("header");

// セレクターで取得（最初の1つ）
const button = document.querySelector(".btn-primary");

// セレクターで取得（全部）
const items = document.querySelectorAll(".nav-item");
```

`getElementById` は ID で 1 つの要素を取得します。`querySelector` は CSS セレクターと同じ書き方で要素を探せるため、実務ではよく使われます。

### 要素を操作する

取得した要素に対して、テキスト、スタイル、クラスを変更できます。CSS で定義したクラスを付け外しする方法は、見た目の変更を管理しやすいのでおすすめです。

```js
// テキストを変更
button.textContent = "クリックされました";

// スタイルを変更
header.style.backgroundColor = "#333";

// クラスを追加・削除
button.classList.add("active");
button.classList.remove("hidden");
button.classList.toggle("open");   // あれば削除、なければ追加
```

直接 `style` を変更することもできますが、複数の見た目をまとめて切り替える場合は `classList.add/remove/toggle` を使うと整理しやすくなります。

### イベントリスナー

ボタンのクリックや入力など、ユーザーの操作に反応する処理です。

イベントリスナーは、「何かが起きたらこの関数を実行する」という登録です。クリック、入力、送信、スクロールなど、ブラウザ上の操作はイベントとして扱われます。

```js
const button = document.querySelector("#submit-btn");

button.addEventListener("click", () => {
  console.log("ボタンがクリックされました");
  button.textContent = "送信済み";
  button.disabled = true;
});

// フォームの入力内容を取得
const input = document.querySelector("#name-input");
input.addEventListener("input", (event) => {
  console.log("入力値:", event.target.value);
});
```

`addEventListener("click", ...)` は、ボタンがクリックされたときの処理を登録しています。入力欄の `input` イベントでは、`event.target.value` から現在の入力値を取得できます。

---

## Astro での JavaScript の使い方

> このセクションは [Astro](Astro) を使う場合に読んでください。

Astro では、ページファイル（`.astro`）の上部に **フロントマター** という領域があり、ここに JavaScript を書きます。

フロントマターは、HTML を生成する前に実行される JavaScript です。データを用意してテンプレートへ埋め込む用途に向いています。

```astro
---
// フロントマター（ビルド時に実行される JavaScript）
const title = "会社概要";
const members = ["田中", "山田", "佐藤"];
---

<!-- HTML テンプレート -->
<h1>{title}</h1>

<ul>
  {members.map((name) => <li>{name}</li>)}
</ul>
```

`{title}` や `{members.map(...)}` は、JavaScript の値を HTML テンプレート内に埋め込んでいます。配列からリスト項目を作る発想は、React などのフロントエンドでもよく使います。

フロントマターはサーバー側（ビルド時）で実行されるため、DOM 操作はできません。DOM 操作は `<script>` タグの中に書きます。

クリックなどユーザー操作に反応するコードは、ブラウザで実行される必要があります。そのため、次のように `<script>` タグの中へ書きます。

```astro
<button id="toggle-btn">メニューを開く</button>
<nav id="nav-menu" class="hidden">...</nav>

<script>
  // ここはブラウザで実行される
  const btn = document.getElementById("toggle-btn");
  const nav = document.getElementById("nav-menu");

  btn.addEventListener("click", () => {
    nav.classList.toggle("hidden");
  });
</script>
```

このコードでは、ボタンをクリックするたびに `hidden` クラスを付け外ししています。CSS 側で `.hidden { display: none; }` のように定義しておけば、メニューの表示・非表示を切り替えられます。

---

## よくある疑問

**Q. セミコロン（`;`）は必要？**  
A. 省略可能ですが、書く方が安全です。このプロジェクトでは書くことを推奨します。

**Q. `==` と `===` はどう違う？**  
A. `===` は値と型の両方が一致するときだけ `true`。`==` は型を変換してから比較するため予期しない結果になることがあります。常に `===` を使いましょう。

**Q. `var` というキーワードも見たけど？**  
A. 古い書き方です。スコープの扱いに落とし穴があるため、現在は `const` と `let` を使います。

---


## 確認問題

1. JavaScript 基礎 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [HTML / CSS](HTML-CSS) — Web ページの基本構造
- [TypeScript](TypeScript) — JavaScript に型をつけた言語
- [Astro](Astro) — JavaScript を使う Web フレームワーク
- [npm / Node.js](npm) — JavaScript のパッケージ管理

---

[← ホームへ](Home)
