# NumPy / pandas 基礎

データサイエンスの土台となる **NumPy** と **pandas** を学びます。NumPy は数値計算・行列演算の基盤、pandas は表形式データの読み込み・加工・集計の中心です。機械学習モデルへのデータ入力から可視化・Web API 連携まで、あらゆる場面で使います。

---

## はじめて読む人へ

NumPy と pandas は、データサイエンスの土台になる道具です。NumPy は数値配列を高速に扱い、pandas は表形式データを読み込み、加工し、集計するために使います。


### 読む前に押さえること

- NumPy の配列は、同じ種類の数値をまとめて高速に計算します。
- pandas の DataFrame は、行と列を持つ表としてデータを扱います。
- 分析では、読み込み・確認・抽出・欠損処理・集計の流れが基本です。

### 読み終えたら説明できること

- ndarray と DataFrame の違いを説明できる。
- 列選択、条件フィルタ、集計を読める。
- Web API や機械学習に渡す前のデータ準備を理解できる。

---

## NumPy

### ndarray（多次元配列）の基本

NumPy の中心は `ndarray` です。これは、同じ型の値を規則的な形で並べた配列で、Python のリストより高速に数値計算できます。データサイエンスでは、表の数値列、画像、ベクトル、行列などをこの形で扱います。

```python
import numpy as np

# リストから配列を作成
a = np.array([1, 2, 3, 4, 5])
b = np.array([[1, 2, 3], [4, 5, 6]])   # 2次元

print(a.shape)    # (5,)
print(b.shape)    # (2, 3)
print(b.dtype)    # int64

# 特殊な配列の作成
np.zeros((3, 4))      # すべて 0 の 3×4 配列
np.ones((2, 3))       # すべて 1
np.eye(3)             # 3×3 単位行列
np.arange(0, 10, 2)   # [0, 2, 4, 6, 8]
np.linspace(0, 1, 5)  # 0〜1 を等間隔に 5 点
```

`shape` は配列の形、`dtype` は要素の型を表します。`a.shape` が `(5,)` なら 1 次元に 5 個、`b.shape` が `(2, 3)` なら 2 行 3 列の配列です。

`zeros` や `ones` は初期値をそろえた配列、`arange` や `linspace` は規則的な数列を作る関数です。機械学習では、特徴量行列や重み行列を作るときに、この「形」を意識することが重要になります。

### スライスとインデックス

インデックスは、配列のどの位置の値を取り出すかを指定する方法です。NumPy では、行と列を `arr[行, 列]` の形で指定します。

```python
arr = np.array([[1, 2, 3],
                [4, 5, 6],
                [7, 8, 9]])

arr[0]        # [1, 2, 3]  1行目
arr[:, 1]     # [2, 5, 8]  2列目
arr[1:, :2]   # [[4, 5], [7, 8]]  部分配列

# ブールインデックス
arr[arr > 5]  # [6, 7, 8, 9]
```

`arr[:, 1]` の `:` は「すべて」を表すため、すべての行の 2 列目を取り出します。`arr > 5` は各要素について条件判定を行い、真になった要素だけを取り出すブールインデックスです。

### ブロードキャスト

**形状が異なる配列でも、自動的に次元を合わせて計算できる仕組み** です。

ブロードキャストは、NumPy の計算を短く書ける理由の 1 つです。行列の各行に同じベクトルを足す、列ごとに平均を引く、といった処理を Python の for ループなしで書けます。

```python
a = np.array([[1, 2, 3],
              [4, 5, 6]])   # shape (2, 3)

b = np.array([10, 20, 30])  # shape (3,) → 自動的に (2, 3) に拡張

a + b
# → [[11, 22, 33],
#    [14, 25, 36]]
```

この例では、`b` は 1 行分のベクトルですが、`a` の 2 行に合わせて自動的に各行へ足されます。実際に大きな配列をコピーしているわけではないため、効率よく計算できます。

> **情報工学メモ：ブロードキャストの仕組み**  
> NumPy は演算時に次元数が少ない配列の先頭に `1` を補完し、サイズが 1 の次元を繰り返しコピー（実際にはコピーせずポインタで管理）して形状を揃えます。これにより Python の for ループを使わずに C レベルの速度で行列演算が可能になります。

### 演算と線形代数

NumPy では、配列同士の演算は基本的に「対応する要素同士」で行われます。行列積や逆行列のような線形代数の計算は、専用の演算子や関数を使います。

```python
# 要素ごとの演算（Pythonのループより100倍以上速い）
a = np.array([1.0, 2.0, 3.0])
b = np.array([4.0, 5.0, 6.0])

a * b           # [4, 10, 18]   要素積
np.dot(a, b)    # 32.0          内積
np.sum(a)       # 6.0
np.mean(a)      # 2.0
np.std(a)       # 0.816...

# 行列演算
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])

A @ B           # 行列積（@ 演算子）
np.linalg.inv(A)       # 逆行列
np.linalg.eig(A)       # 固有値・固有ベクトル
np.linalg.svd(A)       # 特異値分解（SVD）
```

`a * b` は要素ごとの掛け算、`np.dot(a, b)` は内積です。2 次元配列では、`A @ B` が行列積を表します。機械学習や統計では、特徴量行列と重みベクトルの計算にこの考え方が出てきます。

### 乱数生成

乱数は、シミュレーション、データ分割、モデルの初期化などで使います。再現性を保つためには、乱数生成器に seed を指定します。

```python
rng = np.random.default_rng(seed=42)  # 再現性のために seed を固定

rng.normal(loc=0, scale=1, size=(100, 3))   # 正規分布
rng.uniform(low=0, high=1, size=50)          # 一様分布
rng.integers(0, 10, size=20)                 # 整数乱数
```

同じ seed を使うと、同じ乱数列が生成されます。実験結果を再現したいとき、授業で全員が同じ結果を確認したいときに重要です。

---

## pandas

pandas の DataFrame は、行と列を持つ表としてデータを扱うための構造です。Excel の表に似ていますが、Python のコードで読み込み、絞り込み、集計、結合を再現できる点が重要です。

分析では、まず `head()`、`info()`、`describe()` などでデータの様子を確認します。列の型や欠損値を見ないまま処理を進めると、数値だと思っていた列が文字列だった、欠損が混ざっていた、という問題に後から気づくことがあります。

### DataFrame の基本操作

DataFrame は、行と列を持つ表です。各列には名前があり、列ごとに型があります。まずは読み込んだ直後に、行数・列数・型・欠損値を確認します。

```python
import pandas as pd

# CSV を読み込む
df = pd.read_csv("data.csv", encoding="utf-8")

# 基本情報を確認
df.shape          # (行数, 列数)
df.dtypes         # 各列のデータ型
df.describe()     # 数値列の統計量
df.isnull().sum() # 欠損値の数

# 列の選択
df["name"]               # Series として取得
df[["name", "age"]]      # DataFrame として取得

# 条件フィルタ
df[df["age"] >= 20]
df[(df["age"] >= 20) & (df["score"] >= 80)]

# 欠損値の処理
df.dropna()              # 欠損行を削除
df.fillna(0)             # 欠損値を 0 で埋める
df["age"].fillna(df["age"].mean())  # 平均値で補完
```

`df["name"]` は 1 列だけを Series として取り出し、`df[["name", "age"]]` は複数列を DataFrame として取り出します。条件フィルタでは、条件を満たす行だけが残ります。

欠損値の扱いは分析結果に大きく影響します。削除するのか、0 で埋めるのか、平均値で補完するのかは、データの意味に合わせて選ぶ必要があります。

### 集計・グループ化

`groupby` は、カテゴリごとに行をまとめて集計するための機能です。SQL の `GROUP BY` と同じ発想で、学科別平均、曜日別売上、ユーザー別購入回数などを計算できます。

```python
# グループごとの集計
df.groupby("category")["score"].mean()
df.groupby("category").agg({"score": ["mean", "std"], "age": "count"})

# ピボットテーブル
df.pivot_table(values="score", index="category", aggfunc="mean")

# 並べ替え
df.sort_values("score", ascending=False).head(10)
```

`agg` を使うと、複数の列に複数の集計方法をまとめて適用できます。`sort_values(...).head(10)` は、上位 10 件を確認する典型的なパターンです。

### データの結合

複数の表を組み合わせるときは、どの列をキーにするか、どの行を残すかを決めます。結合は便利ですが、行数が意図せず増減しやすいので、実行後に `shape` を確認する習慣が大切です。

```python
# 内部結合（共通キーのある行のみ）
merged = pd.merge(df1, df2, on="user_id", how="inner")

# 左外部結合（左側のすべての行を保持）
merged = pd.merge(df1, df2, on="user_id", how="left")

# 縦方向に結合
combined = pd.concat([df1, df2], ignore_index=True)
```

`inner` は両方にキーがある行だけを残し、`left` は左側の表の行をすべて残します。`concat` は同じ形の表を縦に積み重ねるときに使います。

### NumPy との連携

pandas の内部は NumPy で動いています。相互変換は簡単です。

機械学習ライブラリの多くは、入力として NumPy 配列を受け取ります。そのため、pandas で前処理した DataFrame を、最後に `.to_numpy()` で数値配列へ変換する場面がよくあります。

```python
# DataFrame → NumPy 配列
X = df[["age", "score", "hours"]].to_numpy()  # shape (n, 3)

# NumPy 配列 → DataFrame
df_new = pd.DataFrame(X, columns=["age", "score", "hours"])

# 列ごとの統計量（NumPy で計算）
np.corrcoef(df["age"], df["score"])   # 相関係数
np.percentile(df["score"], [25, 50, 75])  # 四分位数
```

`X` の形が `(n, 3)` なら、n 件のデータに対して 3 つの特徴量があるという意味です。機械学習では、この「行がサンプル、列が特徴量」という形を何度も使います。

---

## Web との接続

Web API とデータ分析をつなぐときは、JSON と DataFrame の変換がよく出てきます。API から受け取った JSON を表にし、分析後にまた JSON として返す、という流れです。

```python
# API から JSON を取得して DataFrame に変換
import requests
response = requests.get("https://api.example.com/data")
df = pd.DataFrame(response.json())

# DataFrame を JSON に変換して API レスポンスとして返す
data = df.to_dict(orient="records")   # [{"col": val, ...}, ...]

# FastAPI で DataFrame のデータを返す例
from fastapi import FastAPI
app = FastAPI()

@app.get("/data")
def get_data():
    df = pd.read_csv("data.csv")
    return df.to_dict(orient="records")
```

`to_dict(orient="records")` は、DataFrame の各行を辞書にし、それをリストにまとめます。これは JSON と相性がよく、FastAPI のレスポンスとして返しやすい形です。

ただし、大きな DataFrame をそのまま API で返すとレスポンスが重くなります。実務では、ページング、列の絞り込み、集計済みデータの返却などを設計します。

---

## 時系列データ処理

pandas は時系列データ（DatetimeIndex）の操作に特化した機能を持ちます。

```python
import pandas as pd
import numpy as np

# DatetimeIndex を持つ DataFrame を作成
dates = pd.date_range("2024-01-01", periods=365, freq="D")
df = pd.DataFrame({
    "sales": np.random.randint(50, 200, 365),
    "temp":  np.random.normal(15, 8, 365),
}, index=dates)

# ── リサンプリング（集約粒度を変える） ─────────────────────
weekly  = df["sales"].resample("W").sum()    # 週次集計
monthly = df["sales"].resample("ME").mean()  # 月次平均

# ── ローリング集計 ─────────────────────────────────────────
df["sales_7d_avg"]  = df["sales"].rolling(window=7).mean()    # 7 日移動平均
df["sales_7d_std"]  = df["sales"].rolling(window=7).std()     # 7 日標準偏差
df["sales_30d_max"] = df["sales"].rolling(window=30).max()    # 30 日最大値

# ── シフト（過去の値を特徴量に） ─────────────────────────────
df["sales_lag1"]  = df["sales"].shift(1)   # 1 日前
df["sales_lag7"]  = df["sales"].shift(7)   # 1 週前
df["diff_1"]      = df["sales"].diff(1)    # 1 日変化量

# ── 日時インデックスから特徴量を抽出 ────────────────────────
df["month"]       = df.index.month
df["dayofweek"]   = df.index.dayofweek
df["is_weekend"]  = df["dayofweek"].isin([5, 6]).astype(int)
df["quarter"]     = df.index.quarter

# ── 時間範囲でフィルタリング ─────────────────────────────────
q1 = df["2024-01-01":"2024-03-31"]
recent = df[df.index >= "2024-06-01"]

print(df.tail())
```

| 操作 | 意味 |
|------|------|
| `resample("W")` | 週単位で集約（W=Weekly, ME=月末, QE=四半期末） |
| `rolling(n)` | 直近 n ステップの窓集計 |
| `shift(k)` | k 期分シフト（正=過去、負=未来） |
| `diff(k)` | k 期差分（`x[t] - x[t-k]`） |

---

## よくある疑問

**Q. pandas Series と NumPy 配列の違いは？**  
A. Series はインデックス（ラベル）を持ち、欠損値（NaN）を扱えます。NumPy 配列はラベルを持たない純粋な数値配列で、計算速度が速いです。機械学習モデルへの入力は NumPy 形式が多いため、前処理後に `.to_numpy()` で変換するのが一般的です。

**Q. for ループと NumPy の速度差は？**  
A. 要素数 100 万のリストで合計を計算する場合、Python ループは約 100ms、NumPy は約 1ms です（100 倍以上）。NumPy は C/Fortran で書かれたベクトル演算を使うためです。

---


## 確認問題

1. NumPy / pandas 基礎 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [正規表現](正規表現) — pandas の文字列列（`str.match`・`str.extract`）に使います
- [データ可視化](データ可視化) — DataFrame のデータを matplotlib/seaborn/plotly でグラフ化
- [確率・統計基礎](確率・統計基礎) — pandas で計算する統計量の意味
- [特徴量エンジニアリング](特徴量エンジニアリング) — ML のためのデータ前処理
- [教師あり学習](教師あり学習) — sklearn でのモデル訓練

---

[← ホームへ](Home)
