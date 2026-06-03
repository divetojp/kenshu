# npm / Node.js

## このプロジェクトでの前提

このプロジェクトでは **Docker を使って開発環境を起動します**。  
そのため、`npm run dev` などのコマンドは Docker が自動的にコンテナ内で実行してくれます。  
**普段の作業で npm コマンドを手動で叩くことはほとんどありません。**

ただし、npm・Node.js・package.json の仕組みを理解しておくと、エラーが起きたときや設定を変更したいときに役立ちます。

---

## はじめて読む人へ

npm は、JavaScript のライブラリを管理する道具です。Python の `pip` に近く、プロジェクトに必要なパッケージをインストールしたり、開発用コマンドを実行したりします。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- `package.json` は、プロジェクトの依存関係とコマンド一覧です。
- `node_modules` は、インストールされたライブラリ本体です。
- `npm run` は、よく使うコマンドに短い名前を付けて実行する仕組みです。

### 読み終えたら説明できること

- npm、Node.js、package.json の関係を説明できる。
- `npm install` と `npm run` の違いを理解できる。
- 依存関係が変わったときに何を確認するか分かる。

---

## Node.js とは

**JavaScript をブラウザの外（自分の PC 上）で動かすための実行環境** です。

本来、JavaScript はブラウザ内でのみ動く言語でした。Node.js の登場によって、PC 上でも JavaScript が実行できるようになり、Web 開発ツールの多くが JavaScript で作られるようになりました。

Astro もその一つです。`npm run dev` や `npm run build` といったコマンドは、Node.js を使って実行されています。

---

## npm とは

**npm** （Node Package Manager）は、 **Node.js のパッケージ（ライブラリ・ツール）を管理するツール** です。

Node.js をインストールすると、npm も一緒にインストールされます。

> **パッケージとは？**  
> 他の人が作った便利な機能のまとまりです。「車輪の再発明をしない」ために、世界中の開発者が公開しているコードを自分のプロジェクトで使えます。  
> たとえば Astro 自体も、npm でインストールされるパッケージです。

---

## package.json とは

`package.json` は、そのプロジェクトがどんな JavaScript プロジェクトなのかを説明する設定ファイルです。依存パッケージ、実行コマンド、プロジェクト名などがここにまとまります。

チーム開発では、`package.json` を見れば「このプロジェクトを動かすには何が必要か」が分かります。`scripts` に書かれたコマンドは `npm run` で実行できるため、長いコマンドを覚えなくても同じ手順を共有できます。

> **情報工学メモ：セマンティックバージョニング（SemVer: major.minor.patch）**  
> `"astro": "^4.0.0"` のようなバージョン番号は **SemVer（Semantic Versioning）** という規則に従っています。`major.minor.patch` の3つの数字で意味を表します。 **patch** （4.0.0 → 4.0.1）はバグ修正のみで後方互換性あり、 **minor** （4.0.0 → 4.1.0）は機能追加だが既存コードは壊れない、 **major** （4.0.0 → 5.0.0）は破壊的変更を含む可能性あり、という意味です。`^4.0.0` の `^` は「4.x.x の範囲で最新を使う（majorは変えない）」という意味です。これにより `npm install` したときのバージョンのばらつきを制御しています。

プロジェクトのルートフォルダにある `package.json` は、 **このプロジェクトで使うパッケージの一覧と設定を記録したファイル** です。

次の例では、`scripts` に開発・ビルド・プレビュー用のコマンドが定義され、`dependencies` に Astro が依存パッケージとして書かれています。

```json
{
  "name": "ota_hp",
  "scripts": {
    "dev": "astro dev",
    "build": "astro build",
    "preview": "astro preview"
  },
  "dependencies": {
    "astro": "^4.0.0"
  }
}
```

`npm run dev` を実行すると、`scripts.dev` に書かれている `astro dev` が実行されます。つまり、長いコマンドに短い名前を付けているのが `scripts` です。

| 項目 | 意味 |
|------|------|
| `name` | プロジェクト名 |
| `scripts` | `npm run xxx` で実行できるコマンドの定義 |
| `dependencies` | このプロジェクトが必要とするパッケージの一覧 |

---

## node_modules フォルダとは

`npm install` を実行すると、`node_modules/` フォルダが自動的に作られます。これは **`package.json` に書かれたパッケージの実体（コード本体）が入るフォルダ** です。

`node_modules` は、インストールされたライブラリ本体の置き場所です。自分で編集するフォルダではなく、npm が管理します。

```
ota_hp/
├── node_modules/    ← npm install で自動生成される（巨大）
├── src/
├── package.json
└── package-lock.json
```

> **node_modules は Git で管理しない**  
> `node_modules/` は何千ものファイルを含む巨大なフォルダです。  
> `.gitignore` ファイルに `node_modules/` と書くことで、Git の管理対象から除外されています。  
> `package.json` さえあれば `npm install` で誰でも再現できるため、除外しても問題ありません。

---

## よく使うコマンド

### パッケージをインストールする

`npm install` は、`package.json` と `package-lock.json` をもとに、必要なパッケージを `node_modules` へ入れるコマンドです。

```bash
npm install
```

> `package.json` に書かれたすべてのパッケージを `node_modules/` にダウンロードします。  
> リポジトリをクローンした直後や、`package.json` が変更されたときに実行します。

### 開発サーバーを起動する

開発中にサイトを確認するときは、`npm run dev` を使います。これは Astro の開発サーバーを起動し、保存した変更をブラウザに反映します。

```bash
npm run dev
```

> `package.json` の `scripts.dev` に書かれたコマンド（`astro dev`）を実行します。  
> ブラウザで `http://localhost:4321` を開くとサイトを確認できます。

### 本番用にビルドする

本番公開前には、Astro のソースを静的な HTML/CSS/JS に変換する必要があります。この変換処理がビルドです。

```bash
npm run build
```

> **ビルド** とは、Astro のコードをブラウザが読める HTML/CSS/JS ファイルに変換する作業です。  
> `dist/` フォルダに出力されます。Cloudflare Pages がデプロイ時に自動で実行してくれるので、手動で実行する機会は少ないです。

### 特定のパッケージを追加する

新しいライブラリを使いたいときは、パッケージ名を指定してインストールします。インストールすると、`package.json` と `package-lock.json` も更新されます。

```bash
npm install パッケージ名
```

> 新しいパッケージを追加します。`package.json` に自動で記録されます。

---

## Docker 環境での npm コマンド

このプロジェクトでは Docker を使うため、npm コマンドは **コンテナの中** で実行する必要があります。ローカルの PC に Node.js がなくても動作します。

Docker 環境では、Node.js と npm はコンテナ内にあります。そのため、依存関係を更新したいときは、コンテナの中で npm を実行する必要があります。

```bash
# 通常の開発（docker compose up で npm run dev が自動実行される）
docker compose up

# コンテナ内で npm install を手動実行（package.json が変わったとき）
docker compose exec web npm install

# コンテナを再ビルド（Dockerfile や package.json に大きな変更があったとき）
docker compose up --build
```

`docker compose exec web npm install` は、すでに起動している `web` コンテナの中で `npm install` を実行するコマンドです。`docker compose up --build` は、イメージを作り直してから起動します。

> **ローカルで `npm install` を実行しても意味がありません。**  
> Docker 環境の中に Node.js が入っているため、コンテナの外（ローカル PC）で npm コマンドを実行しても、コンテナには反映されません。

---

## よくある疑問

**Q. `npm install` と `npm i` は同じ？**  
A. はい、`i` は `install` の省略形です。どちらも同じ動作をします。

**Q. `node_modules` を誤って削除してしまった場合は？**  
A. `npm install` を実行すれば再作成されます。問題ありません。

**Q. `package-lock.json` って何？**  
A. パッケージの正確なバージョンを記録したファイルです。チームで全員が同じバージョンを使えるよう、Git で管理します。基本的に手動で編集しないでください。

---


## 確認問題

1. npm / Node.js は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [環境構築](環境構築) — Node.js のインストール方法
- [Astro](Astro) — npm を使って動かすフレームワーク
- [Docker](Docker) — コンテナ内での npm 実行

---

[← ホームへ](Home)
