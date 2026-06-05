# GPU・CUDA 入門

深層学習・科学計算を高速化する GPU プログラミングの基礎です。CPU と GPU のアーキテクチャの違い・CUDA の並列計算モデル・PyTorch での GPU 活用・メモリ管理・混合精度学習を扱います。「なぜ GPU で学習が速いのか」の仕組みを理解します。

---

## はじめて読む人へ

「GPU を使うと学習が 100 倍速い」と聞きますが、なぜでしょうか？CPU は「1 つの複雑な計算を高速に」行い、GPU は「無数の単純な計算を同時に」行います。行列積（NN の中核計算）は後者に最適であるため、GPU が有効です。このページでは PyTorch での実践を含めて解説します。

### 読む前に押さえること

- [深層学習入門](深層学習入門) — NN の基礎・行列積の役割
- [PyTorch 入門](PyTorch入門) — テンソル・学習ループ

### 読み終えたら説明できること

- CPU と GPU のアーキテクチャの違いを説明できる
- CUDA のスレッド・ブロック・グリッドの階層を説明できる
- PyTorch での GPU 利用・メモリ管理の注意点を説明できる

---

## CPU vs GPU のアーキテクチャ

### CPU の設計哲学

**CPU（Intel Core i9）**

コア: 24 個（最新世代）
各コアは:
・複雑な分岐予測
・アウトオブオーダー実行
・大きな L1/L2 キャッシュ
→ 1 スレッドの逐次処理を最大化

向いている処理: if/else が多い・複雑なロジック・OS・データベース
### GPU の設計哲学

**GPU（NVIDIA A100）**

CUDA コア: 6,912 個（A100）
SM（Streaming Multiprocessor）: 108 個
各 SM:
・64 個の CUDA コア（FP32）
・大きなレジスタファイル
・高帯域幅 L1 キャッシュ
→ 数千スレッドの並列実行に特化

向いている処理: 同じ計算を大量データに適用（行列積・畳み込み）
### なぜ行列積に GPU が向いているか

$C = A \times B$（$N \times N$ 行列）の場合：

C[i][j] = Σ_k A[i][k] × B[k][j]  （N² 個の内積、各 N 回の乗算）

CPU: 1コアが順番に計算 → O(N³) の逐次処理
GPU: C[i][j] の計算を N² スレッドで同時に実行 → 理論上 O(N) に
1024×1024 の行列積では GPU が CPU の約 100 倍高速です。

---

## CUDA の実行モデル

### スレッド階層

Grid（グリッド）
└── Block（ブロック）× 多数
└── Thread（スレッド）× 最大 1024
↓
各スレッドが 1 つの計算を実行

**例：1024×1024 の行列要素を処理**

Grid: 4×4 ブロック（16 ブロック）
Block: 256×256 スレッド（65536 スレッド）
→ 合計 1,048,576 スレッドで並列処理
### CUDA カーネルの例（概念）

```c
// CUDA カーネル（GPU で実行される関数）
__global__ void vector_add(float* a, float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;  // スレッド ID
    if (i < n) {
        c[i] = a[i] + b[i];  // 各スレッドが 1 要素を担当
    }
}

// 呼び出し
int blocks = (n + 255) / 256;
vector_add<<<blocks, 256>>>(a, b, c, n);
```

`blockIdx`・`blockDim`・`threadIdx` でスレッドの位置を特定し、自分が担当する要素を計算します。

---

## PyTorch での GPU 利用

### デバイスの確認と移動

```python
import torch

# GPU の確認
print(torch.cuda.is_available())           # True/False
print(torch.cuda.get_device_name(0))       # "NVIDIA A100-SXM4-80GB"
print(torch.cuda.memory_allocated() / 1e9) # 使用中の VRAM (GB)

# テンソルの GPU 移動
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
x = torch.randn(1000, 1000).to(device)

# モデルの GPU 移動
model = MyModel().to(device)
```

### 学習ループでのベストプラクティス

```python
for batch in dataloader:
    # データをデバイスに移動
    inputs = batch["input"].to(device)
    labels = batch["label"].to(device)

    optimizer.zero_grad()

    with torch.cuda.amp.autocast():  # 混合精度（後述）
        outputs = model(inputs)
        loss = criterion(outputs, labels)

    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

---

## メモリ管理

### VRAM（GPU メモリ）の構成

VRAM 使用量の主な内訳：

| 要素 | サイズの目安 |
|------|------------|
| モデルパラメータ | $\sum(\text{各層のパラメータ数}) \times \text{dtype サイズ}$ |
| 勾配 | モデルパラメータと同サイズ |
| オプティマイザ状態 | Adam の場合はモデルの約 2 倍（$m$, $v$ の 2 種類） |
| アクティベーション | バッチサイズ × シーケンス長 × 隠れ次元 × 層数 |
### 主なエラーと対処

| エラー | 原因 | 対処 |
|--------|------|------|
| `CUDA OOM` | VRAM 不足 | バッチサイズを下げる・勾配チェックポイント |
| 学習が遅い | データ転送のボトルネック | `num_workers` を増やす・`pin_memory=True` |
| 勾配消失（fp16） | float16 のアンダーフロー | GradScaler を使う |

### 勾配チェックポイント

メモリと計算のトレードオフで、アクティベーションを保存せず逆伝播時に再計算します。

```python
from torch.utils.checkpoint import checkpoint

# 通常（アクティベーションをすべて保存）
output = model(input)

# チェックポイント（VRAM を約 1/√L 削減、計算は約 33% 増加）
output = checkpoint(model, input)
```

---

## 混合精度学習（Mixed Precision）

### float32 vs float16 vs bfloat16

| データ型 | ビット数 | 精度 | 最大値 | 用途 |
|---------|--------|------|--------|------|
| float32 | 32 bit | 約 7 桁 | 3.4×10³⁸ | 標準 |
| float16 | 16 bit | 約 3 桁 | 65,504 | GPU 高速化 |
| bfloat16 | 16 bit | 約 2 桁（範囲広）| 3.4×10³⁸ | TPU・Ampere 以降 |

### AMP（Automatic Mixed Precision）

```python
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch in dataloader:
    with autocast():              # forward は fp16 で実行（高速）
        output = model(input)
        loss = criterion(output, target)

    scaler.scale(loss).backward() # 勾配をスケーリングして fp32 で蓄積
    scaler.step(optimizer)        # fp32 でパラメータ更新
    scaler.update()               # スケール係数を更新
```

**AMP の仕組み：**
- 順伝播・一部の backward：fp16（高速・メモリ削減）
- 勾配累積・パラメータ更新：fp32（精度保持）
- GradScaler：fp16 のアンダーフロー（勾配が 0 になる）を防ぐためスケーリング

---

## マルチGPU 学習

### DataParallel（シンプル）

```python
model = torch.nn.DataParallel(model)  # 全 GPU を自動使用
```

バッチを各 GPU に分割し、勾配を統合します。シンプルですが非効率（GPU 0 にボトルネック）。

### DistributedDataParallel（推奨）

各 GPU が独立したプロセスとして動き、NCCL で勾配を同期します。現在の標準的な方法です。

```bash
# 4 GPU でマルチGPU 学習
torchrun --nproc_per_node=4 train.py
```

---

## プロファイリング

```python
# PyTorch Profiler
with torch.profiler.profile(
    activities=[torch.profiler.ProfilerActivity.CPU,
                torch.profiler.ProfilerActivity.CUDA],
    record_shapes=True,
    with_flops=True
) as prof:
    model(input)

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))
```

`nvidia-smi` コマンドで GPU 使用率・VRAM・温度をリアルタイムに確認できます。

---

## 数学的導出

### Roofline モデル

GPU 演算の上限を「演算ピーク性能」と「メモリ帯域幅」の 2 つから考える分析モデルです。

$$
\text{性能} = \min\!\left(P_{\text{peak}},\; I \times B\right)
$$

- $P_{\text{peak}}$：GPU の理論演算性能 [FLOPS]
- $I$：演算強度（算術演算数 ÷ メモリアクセス量）[FLOP/Byte]
- $B$：メモリ帯域幅 [Byte/s]

**行列積の演算強度：** $N \times N$ 行列積の演算数は $2N^3$、メモリアクセスは $3N^2 \times 4$ (bytes)。$N=4096$ なら $I = 2N \approx 8192$ FLOP/Byte で、A100 のルーフライン上では演算バウンドになります（行列積は GPU を効率よく使える操作）。

---

## 確認問題

1. 行列積が GPU に向いている理由を「並列性」の観点から説明してください。
2. 混合精度学習で GradScaler が必要な理由を fp16 のオーバーフロー・アンダーフローと関連づけて説明してください。
3. 勾配チェックポイントが「メモリと計算のトレードオフ」である理由を説明してください。
4. DataParallel より DistributedDataParallel が推奨される理由を説明してください。

---

## 関連ページ

- [PyTorch 入門](PyTorch入門) — テンソル・学習ループの基礎
- [深層学習入門](深層学習入門) — NN・バックプロパゲーション
- [ファインチューニング詳解](ファインチューニング詳解) — QLoRA によるVRAM削減
- [数値計算](数値計算) — 浮動小数点・混合精度の数学的背景
- [MLOps概要](MLOps概要) — GPU クラスターの管理・コスト最適化

---

[← ホームへ](Home)
