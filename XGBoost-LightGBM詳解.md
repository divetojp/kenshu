# XGBoost / LightGBM 詳解

勾配ブースティング決定木（GBDT）の数学的基礎から、XGBoost の正則化・LightGBM の高速化・CatBoost のカテゴリ処理まで詳解します。表形式データの機械学習コンペで 10 年以上トップに君臨する手法群であり、実務でも最頻出アルゴリズムです。

---

## はじめて読む人へ

[アンサンブル学習](アンサンブル学習) ではブースティングの概念と XGBoost の概要を学びました。このページでは「なぜ XGBoost は強いのか」「LightGBM はどう高速化しているのか」を数式レベルで理解します。Kaggle 上位入賞者のほとんどがこの手法を使いこなしています。

### 読む前に押さえること

- [アンサンブル学習](アンサンブル学習) — GBDT・XGBoost の概要
- [決定木](決定木) — 木の分割アルゴリズム・不純度
- [微分・最適化基礎](微分・最適化基礎) — テイラー展開・勾配降下法

### 読み終えたら説明できること

- GBDT の損失関数をテイラー展開して葉の重みを最適化できる理由を説明できる
- LightGBM のヒストグラムベース学習と GOSS の仕組みを説明できる
- XGBoost と LightGBM の使い分け基準を説明できる

---

## GBDT の数学的基礎

### ブースティングの一般形

$M$ 本の木 $f_1, f_2, \ldots, f_M$ を順番に追加し、各ステップで**残差（残っている誤差）を新しい木で学習**します。

$$
\hat{y}_i^{(m)} = \hat{y}_i^{(m-1)} + \eta \cdot f_m(x_i)
$$

$\eta$：学習率（shrinkage）。各木の貢献を小さく抑えることで過学習を防ぎます。

### 損失のテイラー展開（XGBoost の核心）

$m$ ステップ目の目標：

$$
\mathcal{L}^{(m)} = \sum_{i=1}^n \ell(y_i,\, \hat{y}_i^{(m-1)} + f_m(x_i)) + \Omega(f_m)
$$

$\Omega(f_m)$：木の複雑さへの正則化項。$f_m(x_i)$ を小さい量として 2 次テイラー展開：

$$
\mathcal{L}^{(m)} \approx \sum_{i=1}^n \left[g_i f_m(x_i) + \frac{1}{2} h_i f_m(x_i)^2\right] + \Omega(f_m)
$$

- **$g_i = \partial_{\hat{y}} \ell(y_i, \hat{y}_i^{(m-1)})$：1 次微分（勾配）**
- **$h_i = \partial_{\hat{y}}^2 \ell(y_i, \hat{y}_i^{(m-1)})$：2 次微分（ヘシアン）**

どんな損失関数でも $g_i, h_i$ を計算すれば同じ最適化フレームワークで解けます。

### 葉の重みの最適解

木の構造を固定して葉 $j$ の重み $w_j$ を最適化します。葉 $j$ に属するサンプル集合を $I_j$ として：

$$
\mathcal{L}_{\text{leaf}} = \sum_{j=1}^T \left[\left(\sum_{i \in I_j} g_i\right) w_j + \frac{1}{2}\left(\sum_{i \in I_j} h_i + \lambda\right) w_j^2\right] + \gamma T
$$

$\lambda$：L2 正則化（葉の重みを小さくする）、$\gamma$：葉の数に対するペナルティ。

最適重みは単純に導出できます（2 次関数の最小化）：

$$
\boxed{w_j^* = -\frac{\sum_{i \in I_j} g_i}{\sum_{i \in I_j} h_i + \lambda}}
$$

この最適重みを代入した最小損失（**木の品質スコア**）：

$$
\mathcal{L}^* = -\frac{1}{2} \sum_{j=1}^T \frac{\left(\sum_{i \in I_j} g_i\right)^2}{\sum_{i \in I_j} h_i + \lambda} + \gamma T
$$

### 分割の評価（Gain）

あるノードを特徴量 $k$・閾値 $v$ で左右に分割したときの**利得（Gain）**：

$$
\text{Gain} = \frac{1}{2}\left[\frac{G_L^2}{H_L + \lambda} + \frac{G_R^2}{H_R + \lambda} - \frac{(G_L+G_R)^2}{H_L+H_R+\lambda}\right] - \gamma
$$

- $G_L, G_R$：左右のサンプルの勾配の和
- $H_L, H_R$：左右のサンプルのヘシアンの和

**Gain > 0 のときだけ分割する**（$\gamma$ が大きいほど分割を抑制 → 過学習防止）。

---

## XGBoost の工夫

### 正則化パラメータ

| パラメータ | 役割 |
|-----------|------|
| `lambda` ($\lambda$) | 葉の重みへの L2 正則化 |
| `alpha` | 葉の重みへの L1 正則化（スパース解）|
| `gamma` | 分割のための最小 Gain（木を浅くする）|
| `max_depth` | 木の最大深さ |
| `min_child_weight` | 葉のヘシアン和の最小値（下限 $H_j \geq \text{min\_child\_weight}$）|

### 列サブサンプリングと行サブサンプリング

- `subsample`：各木の学習に使う行のサンプリング率（Random Forest のバギングに相当）
- `colsample_bytree` / `colsample_bylevel`：列（特徴量）のサンプリング率

多様な木を作ることで汎化性能が向上します。

### 欠損値の自動処理

分割評価時に欠損値を左右どちらに入れるかも同時に最適化します（Sparsity-Aware Split Finding）。

---

## LightGBM の高速化

XGBoost は全データを使って各特徴量の全分割点を評価するため、大規模データでは遅い問題がありました。LightGBM は 3 つの工夫で劇的に高速化しています。

### 1. ヒストグラムベース GBDT（Histogram-based Tree Learning）

連続値をビン（例：256 ビン）に離散化し、**ヒストグラムを計算してから分割点を評価**します。

!!! info ""
    通常の分割探索: O(n × d)  （n: サンプル数, d: 特徴量数）
    
    ヒストグラム法:
      1. 各特徴量をビン化: O(n × d)  ← 1 回だけ
      2. ヒストグラム計算: O(n × d)
      3. 分割探索: O(bins × d)  ← bins << n
      
    → 大規模データで数十倍高速

**子ノードのヒストグラム差分：** 親 - 大きい子 = 小さい子 のヒストグラムが得られるため、小さい側の計算をスキップできます。

### 2. GOSS（Gradient-based One-Side Sampling）

大きな勾配を持つサンプル（まだうまく学習できていないサンプル）は**全て保持**し、小さな勾配のサンプルはランダムに**間引く**ことで計算量を削減します。

!!! info ""
    勾配が大きい top_rate% のサンプル → 全て使う
    勾配が小さい残りのサンプル → other_rate% をランダムサンプリング
    
    間引いたサンプルに (1 - top_rate) / other_rate の重みを付けて
    分布を補正（ヒストグラムの精度を維持）

### 3. EFB（Exclusive Feature Bundling）

相互に排他的な特徴量（同時に 0 でない場合が少ない）を**バンドル化**して特徴量次元を削減します。スパースな特徴量（One-hot 後のカテゴリ変数等）で特に有効です。

### Leaf-wise vs Level-wise

| 成長戦略 | 説明 | 特徴 |
|---------|------|------|
| **Level-wise**（XGBoost デフォルト）| 同じ深さの全ノードを展開 | 過学習しにくい |
| **Leaf-wise**（LightGBM デフォルト）| 最も Gain の大きい葉だけを展開 | 精度が高いが過学習しやすい |

---

## CatBoost

### カテゴリ変数の問題

通常の Label Encoding や One-Hot Encoding はカテゴリ変数を数値に変換しますが、目的変数を使った Target Encoding はデータリーケージが起きる可能性があります。

### Ordered Boosting と Ordered Target Statistics

!!! info ""
    時系列順（ランダム順）に並べた場合、
    i 番目のサンプルの Target Statistics を計算するとき
    → i 番目のサンプル自体は使わず、1〜i-1 番目のデータのみ使う
    → データリーケージを防ぐ

$$
\hat{x}_{i, \text{TS}} = \frac{\sum_{j=1}^{i-1} \mathbb{1}[x_j = x_i] \cdot y_j + \alpha \cdot P_0}{\sum_{j=1}^{i-1} \mathbb{1}[x_j = x_i] + \alpha}
$$

$P_0$：全体の平均（事前分布）、$\alpha$：平滑化パラメータ。カテゴリの出現頻度が少ない場合は全体平均に近づきます。

---

## 三者の比較

| 項目 | XGBoost | LightGBM | CatBoost |
|------|---------|---------|---------|
| 学習速度 | 中 | **最速** | 中〜遅い |
| カテゴリ変数 | 手動処理 | 組み込みあり | **ネイティブ対応** |
| メモリ | 多い | **少ない** | 中 |
| 数値精度 | 高い | 高い | 高い |
| GPU 対応 | ○ | ○ | ○ |
| ハイパーパラメータ感度 | 高 | 高 | **低（デフォルトが強い）**|

---

## 重要なハイパーパラメータと調整指針

```python
import lightgbm as lgb

params = {
    # 木の構造
    "n_estimators": 1000,          # 木の数（early_stopping と合わせて使う）
    "learning_rate": 0.05,         # 低いほど良いがその分 n_estimators を増やす
    "max_depth": -1,               # -1: 制限なし（num_leaves で制御）
    "num_leaves": 31,              # 2^max_depth より小さく設定（過学習防止）

    # 正則化
    "reg_alpha": 0.1,              # L1
    "reg_lambda": 1.0,             # L2
    "min_child_samples": 20,       # 葉の最小サンプル数

    # サンプリング（過学習防止 + 高速化）
    "subsample": 0.8,              # 行サンプリング
    "colsample_bytree": 0.8,       # 列サンプリング
    "subsample_freq": 5,           # 何ラウンドごとにサンプリングするか

    "objective": "regression",     # タスクに応じて変更
    "metric": "rmse",
}
```

**調整の優先順：**
1. `learning_rate` + `n_estimators`（Early Stopping で自動決定）
2. `num_leaves` / `max_depth`（木の複雑さ）
3. `subsample`, `colsample_bytree`（サンプリング）
4. `min_child_samples`, `reg_alpha`, `reg_lambda`（正則化）

---

## 数学的導出

### 最適葉の重みの導出

葉 $j$ に対する目的関数（定数項を除く）：

$$
L_j(w_j) = G_j w_j + \frac{1}{2}(H_j + \lambda) w_j^2 + \gamma
$$

$G_j = \sum_{i \in I_j} g_i$、$H_j = \sum_{i \in I_j} h_i$ として微分してゼロとおく：

$$
\frac{d L_j}{d w_j} = G_j + (H_j + \lambda) w_j = 0 \implies w_j^* = -\frac{G_j}{H_j + \lambda}
$$

代入した最小値：

$$
L_j^* = G_j \cdot \left(-\frac{G_j}{H_j+\lambda}\right) + \frac{1}{2}(H_j+\lambda)\left(\frac{G_j}{H_j+\lambda}\right)^2 = -\frac{G_j^2}{2(H_j+\lambda)}
$$

分割 Gain はこの「分割後の最小損失の和」から「分割前の損失」を引いたものです。

### 二項分類における $g_i, h_i$

ロジスティック損失 $\ell = -[y_i \log p_i + (1-y_i)\log(1-p_i)]$（$p_i = \sigma(\hat{y}_i)$）の場合：

$$
g_i = \frac{\partial \ell}{\partial \hat{y}_i} = p_i - y_i, \quad h_i = \frac{\partial^2 \ell}{\partial \hat{y}_i^2} = p_i(1-p_i)
$$

直感：$g_i = p_i - y_i$ は「予測確率 - 正解ラベル」、つまり**残差**です。これが次の木が学習するターゲットになります。

---

## 確認問題

1. GBDT でテイラー展開が有用な理由を「任意の損失関数への対応」の観点から説明してください。
2. 葉の最適重み $w_j^* = -G_j/(H_j + \lambda)$ で $\lambda$ が大きいと重みが小さくなる理由を説明してください。
3. LightGBM の GOSS で「小さい勾配のサンプルに重みを付ける」理由を説明してください。
4. CatBoost の Ordered Target Statistics がデータリーケージを防ぐ仕組みを説明してください。

---

## 関連ページ

- [アンサンブル学習](アンサンブル学習) — Random Forest・GBDT の概要
- [決定木](決定木) — 木の分割アルゴリズム・不純度の基礎
- [微分・最適化基礎](微分・最適化基礎) — テイラー展開・勾配の概念
- [説明可能AI（XAI）](説明可能AI) — SHAP によるモデルの解釈
- [モデル評価・チューニング](モデル評価-チューニング) — Early Stopping・Optuna による調整

---

[← ホームへ](Home)
