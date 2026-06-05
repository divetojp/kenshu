# Jupyter Notebook / Google Colab 入門

データサイエンスの標準的な作業環境です。**コードを書いてすぐ実行し、グラフや表をその場で確認できる**対話型の環境で、実験・分析・レポートを 1 つのファイルに統合できます。

---

## はじめて読む人へ

「Python を書いてすぐ動かしたい」「グラフを見ながらコードを直したい」というニーズに応えるのが Jupyter です。授業のレポート・卒業研究・kaggle のコンペ——すべてにおいてデータサイエンティストの日常的な作業環境です。

### 読む前に押さえること

- [プログラミング入門](プログラミング入門.md) — Python の基本的な書き方
- [環境構築](環境構築.md) — Python のインストール

### 読み終えたら説明できること

- Jupyter Notebook と Google Colab の使い分けを説明できる
- セルの種類と基本操作（実行・追加・削除）を使いこなせる
- ノートブックを分析レポートとして整理できる

---

## Jupyter Notebook とは

通常の Python スクリプト（`.py`）はファイル全体を一度に実行しますが、Jupyter は **「セル」単位で実行できる** ドキュメントです。

!!! info ""
    ```
    .py スクリプト:
      コードを全部書く → python script.py で一括実行
      → 途中の確認が難しい
    
    Jupyter Notebook (.ipynb):
      セル 1: データ読み込み → 実行して中身を確認
      セル 2: 前処理 → 実行して結果を確認
      セル 3: グラフ描画 → その場で表示
      → 一歩一歩確認しながら進められる
    ```

### Notebook と Colab の比較

| | Jupyter Notebook | Google Colab |
|---|---|---|
| 実行環境 | 自分の PC | Google のクラウド |
| セットアップ | 必要（pip install） | 不要（ブラウザで即開始） |
| GPU | PC のスペック依存 | 無料の GPU が使える |
| ファイルの保存先 | ローカル | Google Drive |
| インターネット接続 | 不要 | 必要 |
| **おすすめの用途** | 普段の分析・オフライン作業 | 深層学習・すぐ試したいとき |

---

## Google Colab をすぐ使う

環境構築なしで今すぐ始められます。

1. [colab.research.google.com](https://colab.research.google.com) を開く
2. 「ノートブックを新規作成」をクリック
3. セルに Python を書いて **Shift + Enter** で実行

```python
# ← このセルを実行してみましょう
print("Hello, Colab!")
import sys
print(f"Python {sys.version}")
```

### GPU を有効にする（深層学習用）

**ランタイム** → **ランタイムのタイプを変更** → **T4 GPU** を選択

```python
import torch
print(torch.cuda.is_available())  # True なら GPU が使える
```

---

## Jupyter Notebook のインストールと起動

```bash
# JupyterLab（推奨）をインストール
pip install jupyterlab

# 起動（ブラウザが自動で開く）
jupyter lab
```

VS Code を使っている場合は拡張機能「Jupyter」をインストールすれば `.ipynb` ファイルを直接開いて編集できます（別途ターミナルでの起動不要）。

---

## セルの種類と基本操作

### 2 種類のセル

| セルの種類 | 使い方 |
|-----------|--------|
| **コードセル** | Python コードを書いて実行する |
| **Markdown セル** | 見出し・説明文・数式を書く |

セルの種類は上部のドロップダウン（「Code」/「Markdown」）または **M キー** で切り替えます。

### よく使うショートカット

まず **Esc** でコマンドモード（青枠）に入ってから操作します。

| キー | 動作 |
|------|------|
| `Shift + Enter` | セルを実行して次のセルへ |
| `Ctrl + Enter` | セルを実行（その場に留まる） |
| `A` | 上にセルを追加 |
| `B` | 下にセルを追加 |
| `D D` | セルを削除 |
| `M` | Markdown セルに変更 |
| `Y` | コードセルに変更 |
| `Z` | 削除を取り消す |

---

## Markdown セルで分析レポートを書く

Markdown セルを使うと、コードとその説明を一緒に書けます。

````markdown
## データの前処理

欠損値のある列を確認し、中央値で補完します。

```python
df.isnull().sum()
```
````

**数式も書けます**（KaTeX 対応）：

```markdown
回帰係数の最小二乗推定量は
$$\hat{\beta} = (X^\top X)^{-1} X^\top y$$
```

---

## 実践：データ分析の流れ

```python
# ── セル 1：ライブラリの読み込み ──────────────────
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

print("ライブラリ読み込み完了")
```

```python
# ── セル 2：データ読み込みと概観 ──────────────────
# Google Colab でドライブからファイルを読む場合:
# from google.colab import drive
# drive.mount('/content/drive')
# df = pd.read_csv('/content/drive/MyDrive/data.csv')

# サンプルデータを使う場合
from sklearn.datasets import load_iris
import pandas as pd

data = load_iris()
df = pd.DataFrame(data.data, columns=data.feature_names)
df['target'] = data.target

print(df.shape)   # (150, 5)
df.head()         # 最初の 5 行を表示
```

```python
# ── セル 3：基本統計量 ──────────────────────────────
df.describe()
```

```python
# ── セル 4：可視化 ──────────────────────────────────
fig, axes = plt.subplots(1, 2, figsize=(12, 4))

# ヒストグラム
df['sepal length (cm)'].hist(ax=axes[0], bins=20)
axes[0].set_title('がく片の長さの分布')

# 散布図
for target in df['target'].unique():
    subset = df[df['target'] == target]
    axes[1].scatter(subset['sepal length (cm)'],
                    subset['sepal width (cm)'],
                    label=data.target_names[target], alpha=0.7)
axes[1].legend()
axes[1].set_title('がく片の長さ vs 幅')

plt.tight_layout()
plt.show()
```

---

## Colab 便利機能

### ファイルのアップロード

```python
from google.colab import files
uploaded = files.upload()   # ダイアログが開いてファイルを選択できる
```

### 外部ライブラリのインストール

```python
!pip install japanize-matplotlib   # ! を付けるとシェルコマンドが実行できる
```

### GPU メモリの確認

```python
!nvidia-smi
```

### Google Drive のマウント

```python
from google.colab import drive
drive.mount('/content/drive')

# マウント後は通常のファイルパスとして使える
df = pd.read_csv('/content/drive/MyDrive/Colab Notebooks/data.csv')
```

---

## ノートブックを Git で管理するときの注意

`.ipynb` ファイルは JSON 形式で、セルの出力（グラフの画像など）も保存されます。そのままコミットすると diff が巨大になるため、**出力をクリアしてからコミット**するのが慣習です。

```bash
# コミット前にすべての出力をクリアする
jupyter nbconvert --clear-output --inplace notebook.ipynb
```

または `nbstripout` をインストールすると自動化できます：

```bash
pip install nbstripout
nbstripout --install   # このリポジトリの git hook に登録
```

---

## 確認問題

1. Jupyter Notebook と Python スクリプト（`.py`）の最大の違いを説明してください。
2. Google Colab で GPU を有効にするにはどの設定を変えますか？
3. ノートブックを Git で管理するとき、なぜ「出力をクリアしてからコミット」することが推奨されますか？

---

## 関連ページ

- [環境構築](環境構築.md) — Python・pip のインストール
- [プログラミング入門](プログラミング入門.md) — Python の基礎
- [NumPy / pandas](pandas-sklearn.md) — データ操作の実践
- [探索的データ分析（EDA）](EDA.md) — Notebook を使った分析のワークフロー
- [データ可視化](データ可視化.md) — matplotlib・seaborn・plotly
