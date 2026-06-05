# エッジAI・TinyML

クラウドに頼らず、スマートフォン・マイコン・IoT デバイスなどのエッジデバイス上で AI 推論を実行する技術です。モデル圧縮（量子化・枝刈り・知識蒸留）・TensorFlow Lite・ONNX・MCU への展開を扱います。レイテンシ・プライバシー・コストの制約が厳しい実世界応用の核心技術です。

---

## はじめて読む人へ

「スマートスピーカーが『Hey Siri』を常に聞いているのにバッテリーが長持ちする理由」「自動車の衝突判定がクラウドに頼らず 1ms で動く理由」——これらはエッジ AI の賜物です。GPU サーバーではなく、手のひらサイズのデバイスで深層学習推論を動かすための技術を学びます。

### 読む前に押さえること

- [深層学習入門](深層学習入門) — NN の基本構造
- [GPU・CUDA入門](GPU-CUDA入門) — 精度とメモリのトレードオフ

### 読み終えたら説明できること

- エッジデバイスの制約（メモリ・計算量・電力）を定量的に説明できる
- 量子化・枝刈り・知識蒸留の仕組みの違いを説明できる
- TFLite・ONNX・CoreML の使い分けを説明できる

---

## クラウド AI vs エッジ AI

### なぜエッジで AI を動かすのか

| 観点 | クラウド AI | エッジ AI |
|------|-----------|---------|
| レイテンシ | 数十〜数百 ms（通信往復）| < 1 ms（ローカル処理）|
| プライバシー | データがクラウドに送られる | データがデバイスを離れない |
| オフライン | 通信が必要 | オフラインで動作 |
| コスト | API 課金 | 初期コストのみ |
| 電力 | サーバー側 | デバイス電池を消費 |

**用途別の使い分け：**

!!! info ""
    **エッジに適した用途**

    ・ウェイクワード検出（常時起動・低電力）
    ・自動車の衝突回避（超低レイテンシ）
    ・産業機器の異常検知（ネットワーク非依存）
    ・スマホのカメラAI（プライバシー保護）

    **クラウドに適した用途**

    ・大規模言語モデル（膨大な計算量）
    ・バッチ処理・学習
    ・精度最優先のタスク
---

## エッジデバイスの制約

### ハードウェアの現実

| デバイス | RAM | 演算能力 | 電力 |
|--------|-----|---------|------|
| Arduino Nano (MCU) | 2 KB | ~1 MFLOPS | < 50 mW |
| Raspberry Pi Zero | 512 MB | ~1 GFLOPS | 1 W |
| Raspberry Pi 5 | 8 GB | ~10 GFLOPS | 5 W |
| スマートフォン (NPU) | 4〜16 GB | ~10 TOPS | 3 W |
| NVIDIA Jetson Orin | 16 GB | 275 TOPS | 15 W |
| サーバー GPU (A100) | 80 GB | 312 TOPS | 400 W |

**MCU（マイクロコントローラ）向けの TinyML** は KB 単位のフラッシュ・RAM しかない環境で動かします。ResNet-50（25M パラメータ、100 MB）は到底動かず、数 KB〜数 MB のモデルが必要です。

---

## モデル圧縮手法

### 1. 量子化（Quantization）

float32（4 byte/値）を int8（1 byte/値）や int4 に変換し、メモリと演算を削減します。

$$
x_{\text{int8}} = \text{round}\!\left(\frac{x_{\text{float32}}}{s}\right) + z
$$

$s$：スケールファクター、$z$：ゼロポイント（浮動小数点 0 に対応する整数値）。

| 量子化の種類 | 手順 | 精度低下 | 用途 |
|-----------|------|---------|------|
| **PTQ（Post-Training）** | 学習後に統計から変換 | 小〜中 | シンプル・まず試す |
| **QAT（Quantization-Aware Training）** | 量子化誤差を学習中にシミュレート | 最小 | 精度重視 |
| **動的量子化** | 推論時に重みのみ量子化 | 最小 | NLP モデル |

**int8 の効果（理論値）：**
- メモリ：float32 比 **4 分の 1**
- 演算：int8 SIMD 命令で **2〜4 倍高速化**
- 精度低下：top-1 で **0.5〜1% 程度**（タスク依存）

### 2. 枝刈り（Pruning）

重要度の低い重みや接続をゼロに固定して取り除きます。

**重みの重要度指標：**

$$
\text{importance}(w_{ij}) = |w_{ij}| \quad \text{または} \quad |w_{ij}|^2 \cdot \|x_i\|^2
$$

**構造的枝刈り（Structured Pruning）：** チャンネル・フィルタ単位でまるごと削除。実際のレイテンシ削減に有効。

**非構造的枝刈り（Unstructured Pruning）：** 個々の重みをゼロに。スパース行列になるが、専用ハードがないと速度向上が難しい。

| 枝刈り率 | 精度低下目安 | メモリ削減 |
|---------|-----------|---------|
| 50% | 0〜1% | ~50% |
| 80% | 1〜3% | ~80% |
| 90%+ | 依存 | ~90% |

### 3. 知識蒸留（Knowledge Distillation）

大きな**教師モデル**（Teacher）の「ソフトな出力（確率分布）」を使って、小さな**生徒モデル**（Student）を学習させます。

!!! info ""
    ```text
    正解ラベル:    [0, 0, 1, 0, 0]     ← ハードラベル（情報が少ない）
    教師の出力: [0.01, 0.02, 0.95, 0.01, 0.01]  ← ソフトラベル（クラス間の類似性を含む）
    ```
**蒸留損失：**

$$
\mathcal{L}_{\text{KD}} = (1-\alpha) \mathcal{L}_{\text{CE}}(y, \hat{p}_S) + \alpha T^2 \mathcal{L}_{\text{CE}}(p_T^{(T)}, p_S^{(T)})
$$

- $T$：温度パラメータ（高いほどソフトな確率分布に）
- $\alpha$：蒸留の重み
- $p_T^{(T)}, p_S^{(T)}$：温度 $T$ でのスケールされた確率

**なぜソフトラベルが有効か：** 「この画像は 95% 猫、5% 犬」という教師の出力は、単純な「猫」というラベルより豊かな情報を含んでいます。生徒モデルはこの「暗黙的な知識」から効率よく学習できます。

### 手法の比較

| 手法 | 実装コスト | 精度低下 | 推論速度向上 | メモリ削減 |
|------|---------|---------|-----------|---------|
| 量子化（int8） | 低 | 小 | ◎ | ◎ |
| 枝刈り（構造的）| 中 | 中 | ○ | ○ |
| 知識蒸留 | 高（再学習）| 最小 | △（アーキテクチャ依存）| ○ |

---

## 軽量アーキテクチャ

### MobileNet（Depthwise Separable Convolution）

通常の畳み込みを「深さ方向の畳み込み（Depthwise）+ 1×1 畳み込み（Pointwise）」に分解し、演算量を約 **9 分の 1** に削減します。

$$
\frac{\text{Depthwise Separable の計算量}}{\text{標準畳み込みの計算量}} = \frac{1}{N} + \frac{1}{D_K^2}
$$

$N$：出力チャンネル数、$D_K$：カーネルサイズ（通常 3）。$N=256$ なら約 8〜9 倍削減。

### EfficientNet

モデルの深さ・幅・解像度を同時に最適にスケーリングする手法。

### SqueezeNet・ShuffleNet・MobileNet V3

すべて「精度をほぼ維持しながら演算量を大幅削減」を目的とした軽量アーキテクチャです。

---

## デプロイのフレームワーク

### TensorFlow Lite（TFLite）

Android・iOS・Linux 組み込みデバイス向け。`.tflite` 形式に変換して使います。

```python
import tensorflow as tf

# 学習済みモデルを TFLite に変換（量子化込み）
converter = tf.lite.TFLiteConverter.from_saved_model("saved_model/")
converter.optimizations = [tf.lite.Optimize.DEFAULT]  # int8 量子化
tflite_model = converter.convert()

with open("model.tflite", "wb") as f:
    f.write(tflite_model)
```

### ONNX（Open Neural Network Exchange）

PyTorch・TensorFlow など複数フレームワーク間のモデル変換フォーマット。ONNX Runtime で高速推論。

```python
import torch
import onnx

# PyTorch → ONNX 変換
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(model, dummy_input, "model.onnx", opset_version=13)
```

### 主要デプロイフレームワーク

| フレームワーク | 対象デバイス | 特徴 |
|-------------|-----------|------|
| **TFLite** | Android・iOS・Linux MCU | Google エコシステム |
| **ONNX Runtime** | クロスプラットフォーム | フレームワーク非依存 |
| **CoreML** | Apple デバイス（iPhone・Mac）| Apple Neural Engine 活用 |
| **TensorRT** | NVIDIA GPU | 最高の推論速度 |
| **Edge Impulse** | MCU（Arduino・STM32）| MCU 向けノーコードツール |

---

## TinyML の実例

### ウェイクワード検出

「Hey Siri」を常時監視するには 0.5 mW 以下の電力で動く必要があります。数 KB のモデルが 10〜20 ms の音声フレームをローカルで分類します。

**DSP（Fourier 変換）→ MFCC → 小型 CNN/RNN** のパイプラインを MCU 上で実行します。

### 異常検知（製造業）

工場の機械の振動センサーから取得した加速度データを MCU 上でオートエンコーダにかけ、通常とは異なるパターンを異常として検知します。

### ビジョンタスク（組み込みカメラ）

Raspberry Pi・NVIDIA Jetson などで MobileNet ベースの物体検出・人物検出を動かします。

---

## 数学的導出

### Depthwise Separable Convolution の計算量削減の導出

**標準畳み込み（$D_K \times D_K \times C_{in} \times C_{out}$ の重み）の計算量：**

$$
D_K^2 \cdot C_{in} \cdot C_{out} \cdot D_F^2
$$

（$D_F$：出力特徴マップのサイズ）

**Depthwise + Pointwise の計算量：**

$$
D_K^2 \cdot C_{in} \cdot D_F^2 + C_{in} \cdot C_{out} \cdot D_F^2
$$

比を取ると：

$$
\frac{D_K^2 \cdot C_{in} \cdot D_F^2 + C_{in} \cdot C_{out} \cdot D_F^2}{D_K^2 \cdot C_{in} \cdot C_{out} \cdot D_F^2} = \frac{1}{C_{out}} + \frac{1}{D_K^2}
$$

$D_K = 3$、$C_{out} = 256$ のとき $\approx 1/9$。

---

## 確認問題

1. int8 量子化がメモリを 4 分の 1 にできる理由を float32 との型サイズ比較から説明してください。
2. 知識蒸留で「ソフトラベル」がハードラベルより情報量が多い理由を説明してください。
3. Depthwise Separable Convolution が標準畳み込みより約 9 倍効率的な理由を計算量から説明してください。
4. エッジデバイスでクラウドに頼らず推論することの利点と欠点を 2 つずつ挙げてください。

---

## 関連ページ

- [GPU・CUDA入門](GPU-CUDA入門) — 混合精度・量子化の数学的背景
- [深層学習入門](深層学習入門) — NN の基礎構造
- [CNN（画像認識）](CNN) — 軽量 CNN アーキテクチャ（MobileNet）
- [音声処理基礎](音声処理基礎) — ウェイクワード検出のオーディオパイプライン
- [フェデレーテッドラーニング](フェデレーテッドラーニング) — エッジデバイスでの分散学習

---

[← ホームへ](Home)
