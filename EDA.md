# 探索的データ分析（EDA）

データを受け取ったとき、**いきなりモデルを作り始めてはいけません**。まずデータの形・分布・欠損・外れ値・相関を把握することが、分析の精度と信頼性を左右します。この一連の作業が **EDA（Exploratory Data Analysis）** です。

---

## はじめて読む人へ

「モデルの精度が上がらない」「予測がおかしい」の原因の多くは、データへの理解不足です。EDA は「データと対話するプロセス」であり、機械学習プロジェクトの最初の必須ステップです。

### 読む前に押さえること

- [NumPy / pandas](pandas-sklearn.md) — DataFrame の基本操作
- [データ可視化](データ可視化.md) — matplotlib・seaborn の基礎
- [Jupyter Notebook / Google Colab](Jupyter.md) — 対話的な作業環境

### 読み終えたら説明できること

- 新しいデータセットを受け取ったとき、何を最初に確認するかを説明できる
- 欠損値・外れ値・分布の歪みを発見し、対処方針を立てられる
- 変数間の相関関係を可視化して、特徴量選択の方針を立てられる

---

## EDA の全体フロー

```
データ受け取り
  ↓
① 形状・型・基本統計量の確認   ← 全体像を把握
  ↓
② 欠損値の確認                 ← 使えないデータの特定
  ↓
③ 各変数の分布確認             ← 偏り・外れ値を発見
  ↓
④ 変数間の関係確認             ← 相関・交互作用を発見
  ↓
⑤ 仮説の整理                   ← 何が目的変数に効くか？
  ↓
前処理 → モデリングへ
```

---

## ① 形状・型・基本統計量の確認

```python
import pandas as pd
import numpy as np

# サンプルデータ（タイタニック）
df = pd.read_csv('https://raw.githubusercontent.com/datasciencedojo/datasets/master/titanic.csv')

# ── 基本情報 ──────────────────────────────────────
print(df.shape)          # (891, 12) → 891 行 × 12 列
print(df.dtypes)         # 各列のデータ型
df.info()                # 型 + 非ヌル数 + メモリ使用量

# ── 最初と最後の数行 ──────────────────────────────
df.head()    # 先頭 5 行
df.tail()    # 末尾 5 行
df.sample(5) # ランダムに 5 行（偏りを防げる）

# ── 基本統計量 ────────────────────────────────────
df.describe()
# count / mean / std / min / 25% / 50% / 75% / max が一覧で確認できる
```

**確認するポイント：**

```
□ データ件数は期待通りか？（極端に少なくないか）
□ 数値型・文字列型・datetime 型が正しく認識されているか
□ 平均と中央値（50%）が大きく離れていないか（歪み・外れ値のサイン）
□ min / max に異常値がないか（年齢 = -5、気温 = 999 など）
```

---

## ② 欠損値の確認

```python
# ── 欠損数と割合を一覧表示 ────────────────────────
missing = pd.DataFrame({
    '欠損数': df.isnull().sum(),
    '欠損率(%)': (df.isnull().sum() / len(df) * 100).round(1)
}).query('欠損数 > 0').sort_values('欠損率(%)', ascending=False)

print(missing)
# Age     177  19.9
# Cabin   687  77.1
# Embarked  2   0.2
```

```python
import matplotlib.pyplot as plt
import seaborn as sns

# ── 欠損のヒートマップ（パターンを視覚化）────────
plt.figure(figsize=(12, 4))
sns.heatmap(df.isnull(), cbar=False, yticklabels=False, cmap='viridis')
plt.title('欠損値マップ（黄 = 欠損）')
plt.tight_layout()
plt.show()
```

**欠損への対処方針の目安：**

| 欠損率 | 典型的な対処 |
|--------|------------|
| 5% 以下 | 行削除 or 平均・中央値で補完 |
| 5〜30% | 予測補完（他の特徴量から推定）か、欠損フラグ列を追加 |
| 30% 以上 | 列削除を検討（情報量が少なすぎる可能性） |

---

## ③ 各変数の分布確認

### 数値変数

```python
# ── ヒストグラム（全数値列を一括） ──────────────
df.select_dtypes(include='number').hist(
    bins=30, figsize=(15, 8), layout=(3, 4)
)
plt.tight_layout()
plt.show()
```

```python
# ── 箱ひげ図（外れ値の発見）──────────────────────
fig, axes = plt.subplots(1, 3, figsize=(15, 4))
for ax, col in zip(axes, ['Age', 'Fare', 'SibSp']):
    sns.boxplot(y=df[col], ax=ax)
    ax.set_title(col)
plt.tight_layout()
plt.show()
```

```python
# ── 歪度・尖度（分布の形の数値化）────────────────
print(df[['Age', 'Fare']].agg(['skew', 'kurtosis']).T)
# Fare の歪度が大きければ対数変換を検討
```

**確認するポイント：**

```
□ 正規分布に近いか、大きく歪んでいるか（対数変換が必要か）
□ 外れ値（箱ひげ図のひげ外の点）が多くないか
□ 単峰性か多峰性か（ラベルの混在のサイン）
```

### カテゴリ変数

```python
# ── 値ごとの頻度確認 ──────────────────────────────
for col in df.select_dtypes(include='object').columns:
    print(f"\n{col}:")
    print(df[col].value_counts(dropna=False))

# ── 棒グラフ ──────────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
df['Sex'].value_counts().plot(kind='bar', ax=axes[0], title='Sex')
df['Embarked'].value_counts().plot(kind='bar', ax=axes[1], title='Embarked')
plt.tight_layout()
plt.show()
```

**確認するポイント：**

```
□ カテゴリの種類数（ユニーク数）は多すぎないか
□ 特定のカテゴリに偏りすぎていないか（不均衡）
□ 想定外の値・スペルミスがないか
```

---

## ④ 変数間の関係確認

### 相関ヒートマップ

```python
# ── 数値変数間の相関 ──────────────────────────────
corr = df.select_dtypes(include='number').corr()

plt.figure(figsize=(8, 6))
sns.heatmap(corr, annot=True, fmt='.2f', cmap='coolwarm',
            center=0, square=True)
plt.title('相関行列ヒートマップ')
plt.tight_layout()
plt.show()
```

### 目的変数との関係

```python
# 生存率（目的変数）と各カテゴリ変数の関係
fig, axes = plt.subplots(1, 3, figsize=(15, 4))
for ax, col in zip(axes, ['Sex', 'Pclass', 'Embarked']):
    survival_rate = df.groupby(col)['Survived'].mean()
    survival_rate.plot(kind='bar', ax=ax, title=f'{col} × 生存率', rot=0)
    ax.set_ylabel('生存率')
plt.tight_layout()
plt.show()
```

```python
# 数値変数 × 目的変数の分布比較（KDE プロット）
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
for ax, col in zip(axes, ['Age', 'Fare']):
    for survived, label in [(0, '死亡'), (1, '生存')]:
        df[df['Survived'] == survived][col].dropna().plot(
            kind='kde', ax=ax, label=label
        )
    ax.set_title(f'{col} の分布（生存 vs 死亡）')
    ax.legend()
plt.tight_layout()
plt.show()
```

### ペアプロット（全変数の関係を一覧）

```python
# 変数が多すぎると重くなるため、絞り込んで使う
cols = ['Age', 'Fare', 'Pclass', 'Survived']
sns.pairplot(df[cols].dropna(), hue='Survived', plot_kws={'alpha': 0.5})
plt.show()
```

---

## ⑤ 自動 EDA ツール

### ydata-profiling（旧 pandas-profiling）

```python
# pip install ydata-profiling
from ydata_profiling import ProfileReport

profile = ProfileReport(df, title="タイタニック EDA レポート")
profile.to_file("report.html")   # HTML で詳細レポートを出力
# または Notebook 内で表示:
# profile.to_notebook_iframe()
```

1 行でヒストグラム・相関・欠損・外れ値の分析を一括で実行してくれます。素早い概観把握に便利ですが、**自動生成に頼りすぎず自分の目でも確認**することが大切です。

---

## EDA チェックリスト

```
【基本情報】
□ df.shape で行数・列数を確認した
□ df.dtypes で型が正しく認識されているか確認した
□ df.describe() で基本統計量を確認した

【欠損・異常値】
□ 欠損値の列と割合を確認した
□ min/max に異常な値がないか確認した
□ カテゴリ変数にスペルミスや想定外の値がないか確認した

【分布】
□ 数値変数のヒストグラムで歪みを確認した
□ 箱ひげ図で外れ値を確認した
□ カテゴリ変数の頻度バランスを確認した

【関係性】
□ 相関ヒートマップで多重共線性の候補を確認した
□ 目的変数と各特徴量の関係を可視化した
□ 仮説（「この変数が効きそう」）を 3 つ以上書き出した
```

---

## 確認問題

1. 新しいデータセットを受け取ったとき、最初にすべき 3 つの操作を説明してください。
2. ある数値変数の歪度（skewness）が 3.5 だった場合、どのような前処理を検討しますか？
3. 欠損率が 80% の列はどう扱うべきか、理由とともに説明してください。

---

## 関連ページ

- [NumPy / pandas](pandas-sklearn.md) — DataFrame 操作の基礎
- [データ可視化](データ可視化.md) — matplotlib・seaborn・plotly
- [特徴量エンジニアリング](特徴量エンジニアリング.md) — EDA の後の前処理
- [Jupyter Notebook / Google Colab](Jupyter.md) — 対話的な分析環境
