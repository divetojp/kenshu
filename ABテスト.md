# A/Bテスト・実験設計

**「新しいUIの方が本当にクリック率が上がるのか」「このメール施策は効果があるのか」**——A/Bテストは施策の効果をデータで証明する実験設計の手法です。直感や経験則ではなく、統計的な証拠に基づいて意思決定します。

---

## はじめて読む人へ

マーケティング・製品開発・医療・政策評価——どの分野でも「この施策は効いたのか」を証明する必要があります。A/Bテストは[統計的因果推論](因果推論.md)で学んだ RCT（ランダム化比較試験）の実践版です。

### 読む前に押さえること

- [確率・統計基礎](確率・統計基礎.md) — p 値・検定・信頼区間
- [統計的因果推論](因果推論.md) — RCT の考え方

### 読み終えたら説明できること

- サンプルサイズを事前に計算できる
- p 値の正しい解釈と「落とし穴」を説明できる
- 多重比較問題に対処できる

---

## A/Bテストの全体フロー

```mermaid
flowchart LR
    A["仮説を立てる\n（何を変えると何が変わる？）"]
    --> B["指標を決める\n（主要指標 1 つ + 副次指標）"]
    --> C["サンプルサイズを計算\n（検出力・効果量・有意水準）"]
    --> D["ランダムに A/B に割り当て\n（同時並行で実施）"]
    --> E["データ収集"]
    --> F["統計検定\n（有意差があるか？）"]
    --> G["意思決定"]
```

**重要：** A と B は**同時並行**で実施します。「先週 A、今週 B」では季節性などの外部要因が混入します。

---

## サンプルサイズの計算

テストを始める前にサンプルサイズを決めることが必須です。「たまたま有意になったら止める」は多重比較問題を引き起こします。

### 必要な情報

| パラメータ | 意味 | 典型値 |
|----------|------|--------|
| 有意水準 $\alpha$ | 偽陽性（本当は差がないのに差があると判断）の許容確率 | 0.05 |
| 検出力 $1-\beta$ | 本当に差があるとき差を検出できる確率 | 0.80 |
| 効果量 | 検出したい最小の差 | ビジネス要件による |

```python
from scipy import stats
import numpy as np

def calc_sample_size(
    p_baseline: float,   # 対照群の基準コンバージョン率
    mde: float,          # Minimum Detectable Effect（検出したい最小差）
    alpha: float = 0.05, # 有意水準
    power: float = 0.80, # 検出力
) -> int:
    """2 比率の差の検定に必要なサンプルサイズを計算"""
    p_treatment = p_baseline + mde
    p_pooled = (p_baseline + p_treatment) / 2

    # 片側の z スコア
    z_alpha = stats.norm.ppf(1 - alpha / 2)  # 両側検定
    z_beta  = stats.norm.ppf(power)

    n = (
        (z_alpha * np.sqrt(2 * p_pooled * (1 - p_pooled))
         + z_beta * np.sqrt(p_baseline * (1 - p_baseline)
                            + p_treatment * (1 - p_treatment))
         ) ** 2
        / mde ** 2
    )
    return int(np.ceil(n))

# 例: クリック率 5% → 6% への改善（+1%pt）を検出したい
n = calc_sample_size(p_baseline=0.05, mde=0.01)
print(f"1グループあたり必要なサンプル数: {n:,}")
# → 約 14,764 件

# scipy の組み込み関数でも計算可能
from statsmodels.stats.power import NormalIndPower
analysis = NormalIndPower()
# Cohen's h（比率の効果量）を使う場合
from statsmodels.stats.proportion import proportion_effectsize
es = proportion_effectsize(0.06, 0.05)
n2 = analysis.solve_power(effect_size=es, alpha=0.05, power=0.80, alternative='two-sided')
print(f"statsmodels での計算: {int(np.ceil(n2)):,}")
```

---

## 検定の実施

### 比率の検定（クリック率・コンバージョン率）

```python
from scipy.stats import chi2_contingency, proportions_ztest
import numpy as np

# データ
n_A = 10000;  conv_A = 502   # A群: 10000人中502人がコンバージョン
n_B = 10000;  conv_B = 612   # B群: 10000人中612人がコンバージョン

rate_A = conv_A / n_A
rate_B = conv_B / n_B
print(f"A群のコンバージョン率: {rate_A:.2%}")
print(f"B群のコンバージョン率: {rate_B:.2%}")
print(f"相対的な改善: {(rate_B - rate_A) / rate_A:.1%}")

# z 検定（2比率の差）
count = np.array([conv_A, conv_B])
nobs  = np.array([n_A,    n_B])
z_stat, p_value = proportions_ztest(count, nobs)
print(f"\nz統計量: {z_stat:.3f}")
print(f"p値: {p_value:.4f}")
print("→", "有意差あり ✓" if p_value < 0.05 else "有意差なし")

# カイ二乗検定（同じ結果になる）
table = np.array([[conv_A, n_A - conv_A],
                  [conv_B, n_B - conv_B]])
chi2, p_chi2, _, _ = chi2_contingency(table)
print(f"\nカイ二乗統計量: {chi2:.3f}, p値: {p_chi2:.4f}")
```

### 平均の比較（購入金額・滞在時間）

```python
from scipy import stats
import numpy as np

rng = np.random.default_rng(42)
# A群: 平均1000円、B群: 平均1080円
revenue_A = rng.normal(1000, 400, 10000)
revenue_B = rng.normal(1080, 420, 10000)

# Welch の t 検定（等分散を仮定しない）
t_stat, p_value = stats.ttest_ind(revenue_A, revenue_B, equal_var=False)
ci = stats.t.interval(0.95, df=len(revenue_A)-1,
                      loc=revenue_B.mean() - revenue_A.mean(),
                      scale=stats.sem(revenue_B - revenue_A[:len(revenue_B)]))

print(f"A群平均: {revenue_A.mean():.1f}円")
print(f"B群平均: {revenue_B.mean():.1f}円")
print(f"差分: {revenue_B.mean() - revenue_A.mean():.1f}円")
print(f"p値: {p_value:.4f}")
```

---

## よくある落とし穴

### 1. ピーキング（覗き見問題）

!!! info ""
    ```
    ❌ よくやってしまうこと:
      実験中に毎日 p 値を確認して、有意になったら止める
    
      「月曜: p=0.08（続行）」「火曜: p=0.04（有意！止めよう）」
      → これは誤り。複数回チェックすることで偽陽性率が上がる（多重比較）
    
    ✅ 正しいやり方:
      事前に決めたサンプルサイズに達するまで待つ
      またはシーケンシャルテスト（逐次検定）を使う
    ```

### 2. 多重比較問題

```python
# 複数の指標を同時に検定するとき
metrics = ["クリック率", "購入率", "滞在時間", "直帰率", "再訪率"]
# → 5つ全部で α=0.05 の検定をすると、
#   少なくとも1つ偽陽性になる確率 = 1-(0.95)^5 ≈ 23%

# ✅ ボンフェロー二補正
alpha_bonferroni = 0.05 / len(metrics)   # 0.01 に補正
print(f"補正後の有意水準: {alpha_bonferroni:.3f}")

# より統計的に優れた方法（FDR制御）
from statsmodels.stats.multitest import multipletests
p_values = [0.03, 0.04, 0.002, 0.08, 0.01]  # 仮の p 値
reject, p_corrected, _, _ = multipletests(p_values, alpha=0.05, method='fdr_bh')
print("Benjamini-Hochberg 補正後:")
for m, p, p_c, r in zip(metrics, p_values, p_corrected, reject):
    print(f"  {m}: p={p:.3f} → 補正後p={p_c:.3f} {'✓' if r else '✗'}")
```

### 3. ネットワーク効果（SUTVA 違反）

!!! info ""
    ```
    SNS・マーケットプレイス・紹介プログラムなどで起きる問題:
      A群のユーザーが B群のユーザーに影響を与えてしまう
    
    例: 「友達紹介機能」の A/B テスト
      A群のユーザーがB群のユーザーを招待 → 効果が汚染される
    
    対処:
      クラスター単位でランダム化（個人でなく地域・グループで割り当て）
    ```

---

## ベイズ A/Bテスト

頻度論的検定の代わりに、**「B が A より良い確率」を直接計算**できます。

```python
import numpy as np
from scipy.stats import beta

# ベータ分布を事前分布として使う（ベータ-二項共役）
# 事前: Beta(1, 1) = 一様分布（無情報事前）
alpha_prior, beta_prior = 1, 1

# 観測データ
conv_A, n_A = 502, 10000
conv_B, n_B = 612, 10000

# 事後分布（解析的に更新）
post_A = beta(alpha_prior + conv_A, beta_prior + n_A - conv_A)
post_B = beta(alpha_prior + conv_B, beta_prior + n_B - conv_B)

# モンテカルロで「B が A を上回る確率」を計算
samples = 100_000
prob_B_wins = np.mean(post_B.rvs(samples) > post_A.rvs(samples))
print(f"P(B > A) = {prob_B_wins:.1%}")
# → "B が A より良い確率は 99.2%"という解釈ができる（頻度論との違い）
```

**ベイズ A/Bテストの利点：** サンプルサイズを事前に固定しなくてよい・「B が A より X% 良い確率」という直感的な解釈ができる。

---

## 確認問題

1. 「昨日 p=0.06 だったが今日 p=0.04 になったので実験を止めた」の何が問題か、多重比較の観点で説明してください。
2. サンプルサイズを事前計算するとき「検出したい最小の効果量」をどう決めますか？ビジネス的な観点も含めて説明してください。
3. ベイズ A/Bテストで「P(B>A) = 95%」という結果は、頻度論的な「p<0.05」とどう違いますか？

---

## 関連ページ

- [確率・統計基礎](確率・統計基礎.md) — p 値・検定・信頼区間の基礎
- [統計的因果推論](因果推論.md) — RCT との接続・交絡の問題
- [ベイズ理論](ベイズ理論.md) — ベイズ A/Bテストの背景
- [データ倫理・AI倫理](データ倫理.md) — A/Bテストの倫理的問題（同意・公平性）
