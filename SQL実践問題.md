# SQL 実践問題

> [データベース詳解](データベース詳解) で理論を学んだあとに取り組んでください。データ系採用の選考テストで頻出するクエリパターンを実践形式でまとめています。

---

## はじめて読む人へ

SQL 実践問題では、基本の SELECT から一歩進んで、集計、CTE、ウィンドウ関数などを練習します。SQL は書いて覚える面が強いので、結果の表を想像しながら読むことが大切です。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- GROUP BY は、行をグループにまとめて集計します。
- CTE は、複雑なクエリに名前付きの途中結果を作ります。
- ウィンドウ関数は、行を残したまま順位や累積値を計算します。

### 読み終えたら説明できること

- 集計、サブクエリ、CTE の使い分けを説明できる。
- ウィンドウ関数が通常の集計と違う点を理解できる。
- 問題文から必要なテーブル操作を考えられる。

---

## 準備：練習用テーブル

以下のテーブルを前提に解説します。

SQL の実践問題では、まず「どんな表があり、各列が何を表すか」を把握することが重要です。クエリは表に対する操作なので、テーブル設計を読めないまま書き始めると、どの列で結合・集計すればよいか分からなくなります。

```sql
-- 社員テーブル
CREATE TABLE employees (
  id         INT PRIMARY KEY,
  name       VARCHAR(50),
  dept       VARCHAR(50),
  salary     INT,
  hired_on   DATE
);

-- 売上テーブル
CREATE TABLE sales (
  id          INT PRIMARY KEY,
  employee_id INT,
  amount      INT,
  sold_at     DATE
);
```

`employees` は社員の基本情報、`sales` は売上情報を表します。`sales.employee_id` は、どの社員がその売上を作ったかを示す列です。実際には外部キーとして定義することも多いですが、ここではクエリ練習のために関係を読み取れれば十分です。

---

## 集計・GROUP BY・HAVING

`GROUP BY` は、複数の行をグループにまとめてから集計するための構文です。たとえば社員を部署ごとにまとめると、部署ごとの平均給与や人数を計算できます。

`WHERE` と `HAVING` の違いは、SQL の実践でとても重要です。`WHERE` は行をグループ化する前に絞り込み、`HAVING` は集計した後のグループを絞り込みます。

**問題：** 部署ごとの平均給与を求め、平均給与が 400,000 円以上の部署だけを表示してください。

```sql
SELECT
  dept,
  AVG(salary) AS avg_salary
FROM employees
GROUP BY dept
HAVING AVG(salary) >= 400000
ORDER BY avg_salary DESC;
```

このクエリでは、まず `employees` を `dept` ごとにまとめ、各部署の `AVG(salary)` を計算しています。その後、`HAVING AVG(salary) >= 400000` で、平均給与が条件を満たす部署だけを残します。

**ポイント：** `WHERE` は集計前のフィルタ、`HAVING` は集計後のフィルタです。

---

## サブクエリ

サブクエリは、クエリの中に書く別のクエリです。ある値を先に計算し、その結果を外側のクエリの条件として使いたいときに便利です。

**問題：** 全社員の平均給与より高い給与をもらっている社員を一覧してください。

```sql
SELECT name, salary
FROM employees
WHERE salary > (
  SELECT AVG(salary) FROM employees
);
```

内側の `SELECT AVG(salary)` が全社員の平均給与を 1 つの値として返し、外側の `WHERE salary > (...)` がその値と各社員の給与を比較します。平均値を手で計算して書くのではなく、SQL の中で計算して使うのがポイントです。

**相関サブクエリ**（外側のクエリを参照するサブクエリ）：

相関サブクエリは、外側の行を 1 行ずつ見ながら内側のクエリを実行する考え方です。次の例では、社員ごとに「その社員と同じ部署の最高給与」を調べています。

```sql
-- 各社員について、同じ部署内で自分が最高給与かを判定
SELECT name, dept, salary
FROM employees e
WHERE salary = (
  SELECT MAX(salary)
  FROM employees
  WHERE dept = e.dept   -- 外側の e を参照
);
```

`WHERE dept = e.dept` の `e.dept` は、外側で今見ている社員の部署です。これにより、「全社の最高給与」ではなく「同じ部署の最高給与」と比較できます。

---

## CTE（WITH 句）

複雑なクエリをステップごとに分割して読みやすくします。

CTE は、クエリの途中結果に名前を付ける仕組みです。長いクエリを一気に書くのではなく、「まず月別売上を作る」「その結果を使って前月差を出す」のように段階を分けられます。

**問題：** 月別売上合計を求め、前月との差も表示してください。

```sql
WITH monthly_sales AS (
  SELECT
    DATE_FORMAT(sold_at, '%Y-%m') AS month,
    SUM(amount)                   AS total
  FROM sales
  GROUP BY month
)
SELECT
  month,
  total,
  total - LAG(total) OVER (ORDER BY month) AS diff_from_prev
FROM monthly_sales
ORDER BY month;
```

`monthly_sales` は一時的な表のように扱えます。最初の `WITH` 部分で月別売上を作り、後半の `SELECT` ではその結果に対して `LAG` を使って前月の値を参照しています。

**メリット：** 同じ中間結果を複数回参照できます。サブクエリのネストより読みやすくなります。

---

## 再帰 CTE（WITH RECURSIVE）

通常の CTE は 1 回だけ実行しますが、`WITH RECURSIVE` を使うと結果を繰り返し自分自身に当てはめることができます。**木構造（階層データ）の全祖先・全子孫を取得**するときによく使います。

再帰 CTE は、「出発点」と「次の行をたどる規則」をセットで書きます。組織図、カテゴリ階層、フォルダ構造のように、親子関係が何段も続くデータを扱うときに便利です。

```sql
-- employees テーブルに manager_id カラムがある想定
-- 「社員 ID=5 の全上司（祖先）を辿る」クエリ
WITH RECURSIVE ancestors AS (
  -- ① 起点：対象の社員を選ぶ（非再帰部分）
  SELECT id, name, manager_id
  FROM employees
  WHERE id = 5

  UNION ALL

  -- ② 繰り返し：直前の結果の manager_id を使って 1 つ上を取得（再帰部分）
  SELECT e.id, e.name, e.manager_id
  FROM employees e
  INNER JOIN ancestors a ON e.id = a.manager_id
)
SELECT * FROM ancestors;
```

`UNION ALL` の上が起点、下が繰り返し部分です。最初に社員 ID 5 の行を取り、その社員の `manager_id` を使って上司の行を取り、さらにその上司の `manager_id` を使って上へたどります。

**実行イメージ：**

```
ステップ 1：id=5 の行を取得
ステップ 2：id=5 の manager_id を使って、その上司の行を取得
ステップ 3：その上司の manager_id を使って、さらに上の行を取得
  ... manager_id が NULL になるまで繰り返す
```

> 無限ループを防ぐため、PostgreSQL は `CYCLE` 句、MySQL 8.0 は最大再帰回数（デフォルト 1000）で自動停止します。

---

## ウィンドウ関数

集計しながらも行を保持できる強力な機能です。`GROUP BY` と違い、元の行が消えません。

通常の `GROUP BY` は、複数の行を 1 行にまとめます。一方、ウィンドウ関数は元の行を残したまま、順位、前後の値、累積合計などを追加できます。「明細を残したまま集計情報も見たい」ときに使います。

### RANK / DENSE_RANK / ROW_NUMBER

**問題：** 各社員の給与順位を部署内でつけてください。

次のクエリでは、`PARTITION BY dept` で部署ごとにグループを分け、その中で `ORDER BY salary DESC` によって給与の高い順に順位を付けます。

```sql
SELECT
  name,
  dept,
  salary,
  RANK()       OVER (PARTITION BY dept ORDER BY salary DESC) AS rank,
  DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC) AS dense_rank,
  ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC) AS row_num
FROM employees;
```

`OVER (...)` の中身が、順位を付ける範囲と順序を決めています。`PARTITION BY` を省略すると全社員を 1 つの集団として順位付けします。

| 関数 | 同順位が複数いる場合 |
|------|------------------|
| RANK | 1, 1, 3（2 を飛ばします） |
| DENSE_RANK | 1, 1, 2（飛ばしません） |
| ROW_NUMBER | 1, 2, 3（必ず連番） |

### LAG / LEAD（前後の行を参照）

`LAG` は前の行、`LEAD` は次の行を参照する関数です。時系列データで「前回との差」「次回の値」を見たいときによく使います。

```sql
SELECT
  employee_id,
  sold_at,
  amount,
  LAG(amount)  OVER (PARTITION BY employee_id ORDER BY sold_at) AS prev_amount,
  LEAD(amount) OVER (PARTITION BY employee_id ORDER BY sold_at) AS next_amount
FROM sales;
```

この例では、社員ごとに売上日順で並べ、前回売上と次回売上を同じ行に表示しています。`PARTITION BY employee_id` があるため、別の社員の売上と混ざりません。

### SUM / AVG の累積

累積合計は、時系列で「ここまでの合計」を見るときに使います。月次売上なら、各月単体の売上だけでなく、年初からその月までの積み上げを確認できます。

```sql
-- 月次売上の累積合計
SELECT
  month,
  total,
  SUM(total) OVER (ORDER BY month ROWS UNBOUNDED PRECEDING) AS cumulative
FROM monthly_sales;
```

`ROWS UNBOUNDED PRECEDING` は「先頭行から現在行まで」を表します。つまり、各月の行で、その月までの `total` をすべて足した値が `cumulative` になります。

---

## 自己結合

同じテーブルを 2 つの視点で使います。

自己結合は、1 つのテーブルを別々の役割として 2 回使う方法です。社員テーブルの中に「社員」と「上司」がどちらも入っているような場合、同じテーブルを社員側と上司側に分けて考えます。

**問題：** 上司（manager_id）の名前も一緒に表示してください。

```sql
-- employees テーブルに manager_id カラムがある想定
SELECT
  e.name        AS employee,
  m.name        AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

`employees e` は社員としての見方、`employees m` は上司としての見方です。`LEFT JOIN` にしているため、上司がいない社員も結果に残ります。

---

## NULL の扱い

`NULL` は「値がない」ことを表し、通常の比較演算子（`=`, `!=`）では判定できません。

`NULL` は 0 や空文字とは違い、「不明」「未登録」「存在しない」を表す特別な値です。そのため、`manager_id = NULL` のように等号で比較しても期待通りに判定できません。

```sql
-- NG：NULL は = で比較できません
WHERE manager_id = NULL

-- OK
WHERE manager_id IS NULL
WHERE manager_id IS NOT NULL

-- NULL を他の値に置き換えます
SELECT COALESCE(manager_id, 0) FROM employees;
```

`IS NULL` は NULL である行を探し、`IS NOT NULL` は NULL ではない行を探します。`COALESCE` は、最初に NULL ではない値を返す関数で、欠損値を表示用の値に置き換えるときによく使います。

---

## よく出る面接パターン集

| 問題パターン | 使うもの |
|------------|---------|
| N 番目に高い給与 | DENSE_RANK + HAVING / サブクエリ |
| 連続する X 日間 | LAG / LEAD + 日付差 |
| 各グループのトップ N | ROW_NUMBER + WHERE row_num <= N |
| 重複レコードの検出 | GROUP BY + HAVING COUNT(*) > 1 |
| 木構造の全祖先・全子孫 | WITH RECURSIVE（再帰 CTE） |
| 売上がある月・ない月を両方表示 | LEFT JOIN + COALESCE |

**練習サイト：**
- [SQLZoo](https://sqlzoo.net/) — 段階的な問題、無料
- [HackerRank SQL](https://www.hackerrank.com/domains/sql) — 就活によく使われるレベル
- [LeetCode Database](https://leetcode.com/problemset/database/) — FAANG 系企業の過去問多数

---


## 確認問題

1. SQL 実践問題 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [データベース基礎](データベース基礎) — SQL 基本文法・テーブル設計
- [データベース詳解](データベース詳解) — インデックス・トランザクション・正規化
- [データベース × Web](データベース-Web) — SQLAlchemy・FastAPI との統合
- [特徴量エンジニアリング](特徴量エンジニアリング) — SQL 集計を ML の特徴量に活かす

---

[← ホームへ](Home)
