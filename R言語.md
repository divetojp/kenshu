# R 言語入門

統計解析とデータ可視化のために設計された言語です。**統計学者が作った言語**なので、検定・回帰・多変量解析が Python より少ないコードで書けます。授業・研究・医療・社会科学で広く使われています。

---

## はじめて読む人へ

「Python はもう学んだけど R も使えないといけない授業がある」「R で書かれた研究コードを読む必要がある」——このページでは Python との対比を意識しながら R の基本を習得します。

### 読む前に押さえること

- [Python基礎](Python.md) を学んでいると対比で理解が速まりますが必須ではありません

### 読み終えたら説明できること

- R の基本的なデータ操作（vector・data.frame）ができる
- ggplot2 でグラフを作成できる
- t 検定・回帰分析を R で実行できる
- Python と R の使い分けを説明できる

---

## Python vs R の使い分け

| 観点 | Python | R |
|------|--------|---|
| 得意分野 | ML・Web・自動化・汎用プログラミング | 統計解析・学術研究・可視化 |
| 統計の書きやすさ | 標準的 | **非常に簡潔** |
| グラフの品質 | matplotlib（細かい調整が必要） | **ggplot2（宣言的・美しい）** |
| ML ライブラリ | 豊富（scikit-learn・PyTorch 等） | caret・tidymodels |
| パッケージ数 | 多い | CRAN: 2 万以上（統計系が充実） |
| 実行環境 | Jupyter Notebook | **RStudio**（専用 IDE が使いやすい） |
| 業界 | IT・AI・Web | **学術・医療・行政・金融** |

**おすすめの方針：**
- データ分析・機械学習 → Python
- 統計的検定・論文用のグラフ・生命統計 → R
- 両方できるとデータサイエンティストとして強い

---

## インストールと RStudio

```bash
# R 本体（https://cran.r-project.org/ からダウンロードも可）
# Mac（Homebrew）
brew install r

# RStudio（無料の統合開発環境）を推奨
# https://posit.co/downloads/ からダウンロード
```

**RStudio の主要なペイン：**

RStudio は 4 つのペインで構成されています。左上が **スクリプト**（`.R` ファイルの編集）、右上が **環境変数**（定義済み変数の一覧）、左下が **コンソール**（対話実行）、右下が **ファイル・グラフ・ヘルプ** の表示エリアです。
---

## 基本構文

### 変数・代入

```r
# R では <- が代入演算子（= も使えるが <- が慣習）
x <- 42
name <- "田中"
flag <- TRUE

# 型の確認
class(x)      # "numeric"
class(name)   # "character"
class(flag)   # "logical"
```

### ベクトル（R の基本データ型）

R では**単一の値もベクトル（長さ1のベクトル）**です。

```r
# ベクトルの作成
scores <- c(85, 72, 91, 68, 79)   # c() で結合

# 基本演算（ベクトル全体に適用される）
scores * 2    # [1] 170 144 182 136 158
scores > 75   # [1]  TRUE FALSE  TRUE FALSE  TRUE

# 統計量
mean(scores)   # 79
sd(scores)     # 9.273618（標準偏差）
median(scores) # 79
summary(scores)
# Min. 1st Qu. Median  Mean 3rd Qu.  Max.
# 68.0   72.0   79.0  79.0   85.0   91.0
```

### インデックス（1始まり）

```r
v <- c(10, 20, 30, 40, 50)

v[1]      # 10（Python と違い 1 始まり）
v[2:4]    # 20 30 40（スライス）
v[c(1,3)] # 10 30（複数指定）
v[-1]     # 20 30 40 50（1番目を除く）
v[v > 25] # 30 40 50（条件でフィルタ）
```

---

## データフレーム（data.frame）

```r
# data.frame の作成（Python の pandas.DataFrame に相当）
df <- data.frame(
  name  = c("Alice", "Bob", "Carol"),
  score = c(85, 72, 91),
  group = c("A", "B", "A")
)

# 基本操作
nrow(df)        # 行数: 3
ncol(df)        # 列数: 3
dim(df)         # c(3, 3)
head(df)        # 先頭6行
str(df)         # 構造を表示
summary(df)     # 統計量

# 列のアクセス
df$score        # c(85, 72, 91)
df[["name"]]    # c("Alice", "Bob", "Carol")
df[1, 2]        # 1行2列: 85

# 条件でフィルタ
df[df$score >= 80, ]
# name score group
#    Alice    85     A
#    Carol    91     A
```

### tidyverse（モダンな R の標準）

```r
# install.packages("tidyverse")
library(tidyverse)

df %>%
  filter(score >= 80) %>%         # 条件でフィルタ
  select(name, score) %>%         # 列を選択
  arrange(desc(score)) %>%        # 降順ソート
  mutate(grade = if_else(score >= 90, "A", "B"))  # 列の追加

# %>% は「パイプ演算子」（前の結果を次の関数の第1引数に渡す）
# Python の df[df.score >= 80][['name','score']].sort_values('score', ascending=False)
```

### CSV の読み書き

```r
# 読み込み
df <- read.csv("data.csv")           # base R
df <- read_csv("data.csv")           # readr（tidyverse、推奨）

# 書き出し
write.csv(df, "output.csv", row.names = FALSE)
write_csv(df, "output.csv")          # readr 版
```

---

## ggplot2 でデータ可視化

**「データ・軸・グラフの種類を層として積み重ねる」**宣言的な文法です。

```r
library(ggplot2)

# 基本構造: ggplot(データ, aes(x軸, y軸)) + geom_グラフ種類()
ggplot(mpg, aes(x = displ, y = hwy, color = class)) +
  geom_point(alpha = 0.7, size = 2) +
  labs(
    title    = "エンジン排気量と燃費の関係",
    subtitle = "車種別に色分け",
    x        = "排気量 (L)",
    y        = "高速燃費 (mpg)",
    color    = "車種"
  ) +
  theme_minimal()
```

### よく使う geom_

| geom | グラフの種類 | 使用例 |
|------|------------|--------|
| `geom_point` | 散布図 | 2変数の関係 |
| `geom_line` | 折れ線 | 時系列 |
| `geom_bar` | 棒グラフ | カテゴリの頻度 |
| `geom_histogram` | ヒストグラム | 分布の確認 |
| `geom_boxplot` | 箱ひげ図 | グループ間の分布比較 |
| `geom_smooth` | 回帰線・平滑曲線 | トレンドの可視化 |

```r
# ヒストグラム + 密度曲線
ggplot(data.frame(x = rnorm(1000)), aes(x)) +
  geom_histogram(aes(y = after_stat(density)), bins = 30,
                 fill = "steelblue", alpha = 0.7) +
  geom_density(color = "red", linewidth = 1) +
  labs(title = "正規分布のサンプル", x = "値", y = "密度")

# グループ別の箱ひげ図
ggplot(iris, aes(x = Species, y = Sepal.Length, fill = Species)) +
  geom_boxplot() +
  theme_classic()
```

---

## 統計解析

### t 検定

```r
# 対応なし t 検定（2グループの平均が異なるか）
group_a <- c(82, 74, 88, 91, 79)
group_b <- c(70, 66, 72, 68, 75)

result <- t.test(group_a, group_b)
print(result)

# Output:
#   Welch Two Sample t-test
#   t = 3.01, df = 7.8, p-value = 0.017
#   alternative hypothesis: true difference in means is not equal to 0
#   95 percent confidence interval: 3.0  24.2
```

### 線形回帰

```r
# 回帰モデルの構築
model <- lm(mpg ~ wt + hp, data = mtcars)
summary(model)
# Coefficients:
#              Estimate Std. Error t value Pr(>|t|)
# (Intercept)  37.2273     1.5988   23.28   < 2e-16 ***
# wt           -3.8778     0.6327   -6.13  1.12e-06 ***
# hp           -0.0318     0.0090   -3.52   0.00145 **
# R-squared: 0.827

# 予測
predict(model, newdata = data.frame(wt = 3.5, hp = 120))
```

### 相関分析

```r
# ピアソン相関行列
cor(mtcars[, c("mpg", "wt", "hp", "qsec")])

# 相関ヒートマップ（corrplot パッケージ）
library(corrplot)
corrplot(cor(mtcars), method = "color", type = "upper",
         addCoef.col = "black", tl.srt = 45)
```

---

## R Markdown で分析レポートを作る

Python の Jupyter Notebook に相当します。コード・出力・文章を 1 ファイルにまとめて HTML・PDF・Word に出力できます。

````markdown
---
title: "データ分析レポート"
author: "田中太郎"
date: "`r Sys.Date()`"
output: html_document
---

## データの概要

```{r setup, include=FALSE}
library(tidyverse)
df <- read_csv("data.csv")
```

データセットは `r nrow(df)` 行 × `r ncol(df)` 列です。

```{r summary}
summary(df)
```

```{r plot, fig.width=8, fig.height=5}
ggplot(df, aes(x, y)) + geom_point()
```
````

---

## Python との連携（reticulate）

Python と R を同じプロジェクトで使えます。

```r
library(reticulate)

# Python のコードを R から実行
py_run_string("import numpy as np; result = np.mean([1,2,3,4,5])")
cat(py$result)  # 3.0

# pandas の DataFrame を R の data.frame に変換
py_run_file("analysis.py")   # Python スクリプトを実行
df_r <- py$df_python          # Python の変数を R で使う
```

---

## 確認問題

1. `v <- c(10, 20, 30, 40, 50)` で `v[v > 25]` の結果を説明してください（Python のリストと何が違いますか？）。
2. `ggplot()` の `aes()` に書く引数の役割を説明し、`color = class` と `fill = class` の違いを説明してください。
3. `lm(mpg ~ wt + hp, data = mtcars)` の `~` は何を意味しますか？`summary()` の出力の `Pr(>|t|)` は何を表しますか？

---

## 関連ページ

- [確率・統計基礎](確率・統計基礎.md) — 検定・相関の数学的背景
- [探索的データ分析（EDA）](EDA.md) — Python での EDA との比較
- [データ可視化](データ可視化.md) — Python の matplotlib・seaborn
- [統計的因果推論](因果推論.md) — 回帰分析の先の因果推論
