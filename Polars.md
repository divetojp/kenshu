# Polars

Rust 実装の高速 DataFrame ライブラリです。pandas の代替として 2024 年以降に急速に普及しており、大規模データでは pandas の **5〜50 倍**の速度が出ることもあります。遅延評価・Apache Arrow 形式・マルチスレッドが設計の核心です。

---

## はじめて読む人へ

pandas は使い慣れているが「データが大きくなると遅い」「メモリが足りない」という問題は Polars で解決できます。API は pandas に似ていますが「遅延評価」という発想が大きく異なります。まず「なぜ速いのか」を理解してから操作を覚えると定着が早いです。

### 読む前に押さえること

- [NumPy / pandas 基礎](pandas-sklearn) — DataFrame の基本操作

### 読み終えたら説明できること

- Polars が pandas より速い 3 つの理由を説明できる
- Eager API と Lazy API の使い分けを説明できる
- 主要な操作（select・filter・groupby・join）を書ける

---

## なぜ速いのか

### 1. Apache Arrow 形式

pandas の内部表現（NumPy）に対し、Polars は **Apache Arrow** の列指向メモリを使います。

!!! info ""
    ```text
    pandas（行指向メモリ）:
      行1: [Alice, 30, Tokyo]
      行2: [Bob,   25, Osaka]
      ↑ 列を集計するとき全行をスキャンする必要がある

    Polars（列指向メモリ）:
      名前列: [Alice, Bob]
      年齢列: [30, 25]      ← 集計はこの列だけ読めばいい
      都市列: [Tokyo, Osaka]
    ```
列指向は集計・フィルタリング・変換（データ分析の主要操作）が CPU キャッシュに乗りやすく、高速です。

### 2. マルチスレッド並列処理

Polars は自動的に CPU 全コアを使います。pandas は基本的にシングルスレッドです。

### 3. クエリオプティマイザ（遅延評価）

Lazy API では実行計画を最適化してから一度に実行します。

!!! info ""
    ```text
    pandas（即時実行）:
      df.filter(...)     ← 全データをフィルタ
      .select(...)       ← フィルタ後の全列を処理
      .groupby(...)      ← 処理

    Polars Lazy（最適化してから実行）:
      select → filter → groupby の順序を最適化
      使われない列は最初から読まない（述語プッシュダウン）
    ```
---

## 基本操作

### DataFrameの作成と読み込み

```python
import polars as pl

# 作成
df = pl.DataFrame({
    "name": ["Alice", "Bob", "Carol"],
    "age":  [30, 25, 35],
    "city": ["Tokyo", "Osaka", "Tokyo"],
})

# CSV 読み込み（pandas の 5〜10 倍速い）
df = pl.read_csv("data.csv")
df = pl.read_parquet("data.parquet")  # 推奨フォーマット
```

### select / filter

```python
# 列の選択（select）
df.select(["name", "age"])
df.select(pl.col("age") * 2)          # 列演算

# 行のフィルタリング（filter）
df.filter(pl.col("age") > 28)
df.filter((pl.col("city") == "Tokyo") & (pl.col("age") > 25))
```

### 新列の追加（with_columns）

```python
df.with_columns([
    (pl.col("age") + 1).alias("age_next_year"),
    pl.col("city").str.to_uppercase().alias("city_upper"),
])
```

### GroupBy と集計

```python
df.group_by("city").agg([
    pl.col("age").mean().alias("avg_age"),
    pl.col("name").count().alias("count"),
    pl.col("age").max().alias("max_age"),
])
```

### Join

```python
df_left.join(df_right, on="id", how="left")    # LEFT JOIN
df_left.join(df_right, on="id", how="inner")   # INNER JOIN
```

---

## Lazy API（遅延評価）

大規模データに対して Lazy API を使うと、クエリオプティマイザが実行計画を最適化してから処理します。

```python
# Lazy フレームを作成（何も実行されない）
lf = pl.scan_csv("large_data.csv")  # scan_* は Lazy

# 操作を定義（まだ実行されない）
result = (
    lf
    .filter(pl.col("age") > 25)
    .select(["name", "age", "city"])
    .group_by("city")
    .agg(pl.col("age").mean())
    .sort("age")
)

# .collect() で初めて実行（この時点で最適化）
df = result.collect()

# 実行計画を確認
print(result.explain())
```

**述語プッシュダウン：** `filter` が内部で「できるだけ早く」適用されるよう最適化されます。読み込むデータ量が減り大幅に速くなります。

---

## pandas との比較・移行

### API の対応表

| pandas | Polars |
|--------|--------|
| `df[df['col'] > 0]` | `df.filter(pl.col('col') > 0)` |
| `df['new'] = df['col'] * 2` | `df.with_columns(...)` |
| `df.groupby('col').sum()` | `df.group_by('col').agg(pl.sum('col'))` |
| `df.merge(df2, on='id')` | `df.join(df2, on='id')` |
| `pd.read_csv(...)` | `pl.read_csv(...)` |

### 主な違い（注意点）

| 観点 | pandas | Polars |
|------|--------|--------|
| インデックス | あり（整数・文字列） | なし（行番号のみ） |
| 変更 | ミュータブル（`df['col'] = ...`）| イミュータブル（`with_columns` で新 DF）|
| None/NaN | 混在 | `null`（Arrow の null）に統一 |
| 文字列操作 | `str.method()` | `pl.col('s').str.method()` |

---

## パフォーマンス比較

| 操作 | pandas | Polars | 倍率 |
|------|--------|--------|------|
| CSV 読み込み（1GB） | ~15 秒 | ~3 秒 | 5× |
| GroupBy 集計（1億行） | ~45 秒 | ~4 秒 | 11× |
| Join（大テーブル） | ~120 秒 | ~8 秒 | 15× |
| フィルタ + 集計 | ~20 秒 | ~1 秒 | 20× |

（ハードウェア・データ内容によって大きく異なります）

---

## いつ Polars を使うか

| 使う場面 | 理由 |
|---------|------|
| データが 100 万行以上 | pandas が遅くなり始める |
| メモリが制約 | Arrow 形式は効率的 |
| パイプラインの高速化 | Lazy API で最適化 |
| 本番データ処理 | Rust の型安全性・エラーが明示的 |

| 使わない場面 | 理由 |
|------------|------|
| scikit-learn と直接連携 | `to_pandas()` 変換が必要（ただし軽量）|
| matplotlib の入力 | 同上 |
| 小さなデータ（〜10 万行） | pandas で十分 |

---

## 確認問題

1. Polars の列指向メモリが「年齢列の平均を計算する」場合に pandas より速い理由を説明してください。
2. Lazy API が Eager より速い理由を「述語プッシュダウン」の観点から説明してください。
3. Polars に `inplace` 操作がない理由を「イミュータブル設計」の観点から説明してください。

---

## 関連ページ

- [NumPy / pandas 基礎](pandas-sklearn) — pandas の基本操作
- [DuckDB](DuckDB) — SQL でのデータ分析（Polars との相互運用）
- [データエンジニアリング](データエンジニアリング) — Apache Arrow・Parquet との接続
- [探索的データ分析（EDA）](EDA) — Polars を使った EDA

---

[← ホームへ](Home)
