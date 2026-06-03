# CNN（画像認識）

**畳み込みニューラルネットワーク（Convolutional Neural Network）** は画像・音声・時系列データに強いアーキテクチャです。「局所的な特徴を検出する」という画像の性質を反映した設計になっています。

---

## はじめて読む人へ

CNN は、画像のような格子状データから特徴を取り出すニューラルネットワークです。小さなフィルタで画像の一部を見ながら、線・角・模様・物体の形を段階的に学習します。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- 畳み込み層は、画像の局所的な特徴を検出します。
- プーリングは、位置の細かい違いを吸収しながらサイズを小さくします。
- 転移学習では、学習済みモデルを自分のデータに合わせて使います。

### 読み終えたら説明できること

- 入力テンソルの形 `(batch, channel, height, width)` を説明できる。
- 畳み込みとプーリングの役割を理解できる。
- ResNet などの学習済みモデルを使う理由を説明できる。

---

## 畳み込み層

### なぜ畳み込みが必要か

全結合層（普通のニューラルネットワーク）で 224×224 ピクセルの画像を扱うと：

```
224 × 224 × 3（RGB）= 150,528 個の入力値
隠れ層を 1,000 ノードとすると、接続数 = 150,528 × 1,000 = 約 1.5 億パラメータ！

問題 1：パラメータが多すぎて計算・学習が現実的でない
問題 2：「犬が左上にいる画像」と「犬が右下にいる画像」で全く別の特徴を学習してしまう
```

**CNN の解決策**：「小さなフィルターを全体に滑らせる」

「犬の耳」は画像のどこに映っていても「犬の耳」です。左上にあっても右下にあっても同じ形をしています。全結合層の場合、左上の「犬の耳パターン」と右下の「犬の耳パターン」はそれぞれ別の重みで学習されますが、CNN では同一のフィルタを全体に適用するため、どこに出ても同じフィルタが検出します。これが**位置不変性**という性質です。さらにパラメータ数も劇的に減ります。

```
3×3 フィルタ 1 つのパラメータ = 3×3 = 9 個だけ！
同じフィルタを画像全体に適用するので、どこにエッジがあっても同じフィルタが検出できる
→ 位置に依らず特徴を認識（位置不変性）
```

---

### STEP 1：畳み込みを手計算で理解する

5×5 の画像に 3×3 フィルタを当てる最も単純な例です。

```
入力画像（5×5）:           フィルタ（3×3）:

  1  0  1  0  0              1  0 -1
  0  1  0  1  0              1  0 -1
  1  0  1  0  1              1  0 -1
  0  1  0  1  0
  1  0  1  0  0          このフィルタは「縦エッジ検出器」

畳み込みの計算（左上の 3×3 领域から）:
  入力の左上 3×3:     フィルタ:
  1  0  1             1  0 -1
  0  1  0         ×   1  0 -1
  1  0  1             1  0 -1

  = 1×1 + 0×0 + 1×(-1)
  + 0×1 + 1×0 + 0×(-1)
  + 1×1 + 0×0 + 1×(-1)
  = 1 + 0 + (-1) + 0 + 0 + 0 + 1 + 0 + (-1)
  = 0   ← 出力の (0,0) の値

フィルタを 1 つ右にずらして同様に計算 → 出力の (0,1) の値
... と繰り返すと 3×3 の出力特徴マップが得られる
```

```python
import numpy as np

# 手計算を Python で確認
image  = np.array([[1,0,1,0,0],
                   [0,1,0,1,0],
                   [1,0,1,0,1],
                   [0,1,0,1,0],
                   [1,0,1,0,0]], dtype=float)

kernel = np.array([[ 1, 0,-1],  # 縦エッジ検出フィルタ
                   [ 1, 0,-1],
                   [ 1, 0,-1]], dtype=float)

# 畳み込みを手動で実装（ストライド=1、パディングなし）
h, w = image.shape
kh, kw = kernel.shape
out_h, out_w = h - kh + 1, w - kw + 1  # = (3, 3)

output = np.zeros((out_h, out_w))
for i in range(out_h):
    for j in range(out_w):
        region = image[i:i+kh, j:j+kw]  # 3×3 の切り出し
        output[i, j] = (region * kernel).sum()

print("出力（特徴マップ）:")
print(output)
```

フィルタの係数は学習で自動的に決まります。上の「縦エッジ検出フィルタ」は人が設計しましたが、CNN の学習では損失を小さくする方向にフィルタの値が更新されていきます。その結果、最初の層には「エッジ・色」を検出するフィルタが、中間層には「テクスチャ・パーツ」を検出するフィルタが、後の層には「顔・車の形」を検出するフィルタが自然と現れます——これが「特徴を自動で学習する」という意味です。

---

### STEP 2：PyTorch の畳み込み層

```python
import torch
import torch.nn as nn

# 畳み込み層の定義
conv = nn.Conv2d(
    in_channels=3,   # 入力チャンネル数（RGB なら 3）
    out_channels=32, # フィルタの種類数（32 種類の特徴を検出）
    kernel_size=3,   # フィルタのサイズ（3×3）
    padding=1        # 入力の周囲を 0 で埋める（出力サイズを入力と同じに保つ）
)

# テンソルの形：(バッチ数, チャンネル数, 高さ, 幅)
x = torch.randn(8, 3, 224, 224)   # 8 枚の 224×224 RGB 画像
out = conv(x)
print(f"入力: {x.shape}  → 出力: {out.shape}")
# 入力: torch.Size([8, 3, 224, 224])  → 出力: torch.Size([8, 32, 224, 224])
```

```
テンソルの形の読み方：
  (8, 32, 224, 224)
   │   │    │    └─ 幅 W
   │   │    └────── 高さ H
   │   └─────────── チャンネル数（フィルタの種類）
   └─────────────── バッチサイズ（一度に処理する画像枚数）
```

> **情報工学メモ：なぜ全結合層より良いか**  
> 224×224×3 = 150,528 次元の入力を全結合すると、第 1 層だけで数億パラメータが必要です。畳み込みは「同じフィルタを画像全体に適用（重み共有）」するため、3×3×3 = 27 パラメータのフィルタ一つで画像全体のエッジを検出できます。これが **位置不変性（Translation Invariance）** です。

### プーリング層

特徴マップを縮小します。位置のわずかなズレに不変になります。

Max Pooling は、小さな領域の中で最も強く反応した値を残します。これにより、画像中の特徴の位置が少しずれても、同じような特徴として扱いやすくなります。

```python
pool = nn.MaxPool2d(kernel_size=2, stride=2)  # 2×2 の最大値を取り、1/2 に縮小
```

`kernel_size=2, stride=2` では、縦横のサイズがおおよそ半分になります。計算量を減らしつつ、重要な特徴を残すために使います。

---

## 基本的な CNN の実装

次のモデルは、畳み込み層で画像特徴を抽出し、最後に全結合層でクラス分類する基本的な CNN です。前半の `features` が特徴抽出、後半の `classifier` が分類器です。

```python
class SimpleCNN(nn.Module):
    def __init__(self, num_classes=10):
        super().__init__()
        self.features = nn.Sequential(
            nn.Conv2d(1, 32, 3, padding=1),  # 28×28 → 28×28
            nn.BatchNorm2d(32),
            nn.ReLU(),
            nn.MaxPool2d(2),                 # 28×28 → 14×14
            nn.Conv2d(32, 64, 3, padding=1), # 14×14 → 14×14
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),                 # 14×14 → 7×7
        )
        self.classifier = nn.Sequential(
            nn.Flatten(),                    # (N, 64, 7, 7) → (N, 3136)
            nn.Linear(64 * 7 * 7, 256),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(256, num_classes),
        )

    def forward(self, x):
        return self.classifier(self.features(x))

# MNIST（28×28 グレースケール）で試す
model = SimpleCNN(num_classes=10)
```

MNIST は 1 チャンネルの 28×28 画像なので、最初の `Conv2d` は `in_channels=1` です。2 回の MaxPool により、28×28 が 14×14、さらに 7×7 に縮小され、`Flatten` で 1 次元にして分類器へ渡します。

---

## 転移学習（Transfer Learning）

大規模データセット（ImageNet）で学習済みのモデルを流用します。少ないデータでも高精度を達成できます。

転移学習では、すでに大規模画像データで学んだ特徴抽出器を使い、最後の出力層だけを自分のクラス数に合わせます。少ないデータでゼロから CNN を学習するより安定しやすい方法です。

```python
import torchvision.models as models
import torch.nn as nn

# ImageNet で学習済みの ResNet-18 を使う
base_model = models.resnet18(weights=models.ResNet18_Weights.DEFAULT)

# 出力層だけ自分のクラス数に変更
num_features = base_model.fc.in_features  # 512
base_model.fc = nn.Linear(num_features, num_classes)

# ファインチューニング：全層を学習（少データなら出力層だけでもよい）
optimizer = torch.optim.Adam([
    {"params": base_model.layer4.parameters(), "lr": 1e-4},  # 後半層：小さい学習率
    {"params": base_model.fc.parameters(), "lr": 1e-3},      # 出力層：大きい学習率
])
```

`base_model.fc` は ResNet の最後の分類層です。ここを自分の `num_classes` に合わせて置き換えます。後半層は小さい学習率、出力層は大きい学習率にすることで、既存の特徴を壊しすぎず新しい分類に適応させます。

### データオーグメンテーション

少ないデータを水増しします。

データオーグメンテーションは、画像を少し変形して訓練データのバリエーションを増やす手法です。左右反転や回転をしてもラベルが変わらないタスクでは、過学習を減らす助けになります。

```python
from torchvision import transforms

train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(p=0.5),      # 左右反転
    transforms.RandomRotation(degrees=15),        # ±15° 回転
    transforms.ColorJitter(brightness=0.3, contrast=0.3),  # 明度・コントラスト変動
    transforms.RandomCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406],
                         std=[0.229, 0.224, 0.225]),  # ImageNet の平均・標準偏差
])

val_transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
])
```

訓練時にはランダムな変換を入れますが、検証時にはランダム性を避け、Resize と CenterCrop のような固定の変換を使います。評価条件が毎回変わると、性能比較が不安定になるからです。

---

## ResNet の仕組み（残差接続）

152 層などの超深層ネットが学習できる理由は **残差接続（Skip Connection）** です。

深いネットワークでは、層を重ねるほど勾配が伝わりにくくなり、学習が難しくなることがあります。残差接続は、入力を後の層へ直接足すことで、情報と勾配の通り道を作ります。

```python
class ResidualBlock(nn.Module):
    def __init__(self, channels):
        super().__init__()
        self.conv1 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn1 = nn.BatchNorm2d(channels)
        self.conv2 = nn.Conv2d(channels, channels, 3, padding=1)
        self.bn2 = nn.BatchNorm2d(channels)

    def forward(self, x):
        residual = x                   # 入力を「スキップ」しておく
        out = nn.ReLU()(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out = out + residual           # スキップ接続：入力を直接加算
        return nn.ReLU()(out)
        # 効果：最悪でも入力をそのまま流すので勾配消失が起きにくい
```

`out + residual` によって、畳み込み層を通した変換結果に元の入力を足しています。これにより、層が何も学ばない場合でも入力をそのまま流せるため、深いモデルを学習しやすくなります。

---

## 学習ループの実装（CIFAR-10）

CNN 固有の学習ループの例です。訓練フェーズと検証フェーズを分け、各エポックで精度を記録します。

```python
import torch
import torch.nn as nn
import torchvision
import torchvision.transforms as transforms
from torch.utils.data import DataLoader

# CIFAR-10（32×32 カラー画像、10 クラス）
transform = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.RandomCrop(32, padding=4),
    transforms.ToTensor(),
    transforms.Normalize((0.5,), (0.5,)),
])
train_set = torchvision.datasets.CIFAR10(root="./data", train=True, download=True, transform=transform)
val_set   = torchvision.datasets.CIFAR10(root="./data", train=False, download=True,
                                          transform=transforms.Compose([transforms.ToTensor(),
                                                                        transforms.Normalize((0.5,),(0.5,))]))
train_loader = DataLoader(train_set, batch_size=128, shuffle=True,  num_workers=2)
val_loader   = DataLoader(val_set,   batch_size=256, shuffle=False, num_workers=2)

device = "cuda" if torch.cuda.is_available() else "cpu"
model  = SimpleCNN(num_classes=10).to(device)  # 前の節で定義

criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=30)

best_val_acc = 0.0

for epoch in range(30):
    # ── 訓練フェーズ ──────────────────────────────
    model.train()
    train_loss, correct, total = 0.0, 0, 0
    for images, labels in train_loader:
        images, labels = images.to(device), labels.to(device)
        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        train_loss += loss.item() * images.size(0)
        correct += (outputs.argmax(1) == labels).sum().item()
        total += labels.size(0)
    train_acc = correct / total

    # ── 検証フェーズ ──────────────────────────────
    model.eval()
    val_loss, val_correct, val_total = 0.0, 0, 0
    with torch.no_grad():
        for images, labels in val_loader:
            images, labels = images.to(device), labels.to(device)
            outputs = model(images)
            val_loss += criterion(outputs, labels).item() * images.size(0)
            val_correct += (outputs.argmax(1) == labels).sum().item()
            val_total += labels.size(0)
    val_acc = val_correct / val_total

    scheduler.step()

    print(f"Epoch {epoch+1:2d} | train acc {train_acc:.3f} | val acc {val_acc:.3f}")

    # ベストモデルを保存
    if val_acc > best_val_acc:
        best_val_acc = val_acc
        torch.save(model.state_dict(), "best_cnn.pt")

print(f"最良検証精度: {best_val_acc:.3f}")
```

`model.train()` にすると Dropout と BatchNorm が訓練モードになり、`model.eval()` + `torch.no_grad()` で検証/推論モードに切り替わります。この切り替えを忘れると精度が不安定になります。

---

## Grad-CAM（視覚的説明）

Grad-CAM は、CNN がどの領域を見てクラスを判断したかを**ヒートマップ**で可視化する手法です。最後の畳み込み層の勾配を使います。

```python
import torch
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

class GradCAM:
    def __init__(self, model, target_layer):
        self.model = model
        self.gradients = None
        self.activations = None

        target_layer.register_forward_hook(
            lambda m, i, o: setattr(self, "activations", o))
        target_layer.register_backward_hook(
            lambda m, gi, go: setattr(self, "gradients", go[0]))

    def generate(self, x, class_idx=None):
        self.model.eval()
        output = self.model(x)
        if class_idx is None:
            class_idx = output.argmax(1).item()

        self.model.zero_grad()
        output[0, class_idx].backward()

        # チャンネル方向の勾配の平均重み
        weights = self.gradients.mean(dim=[2, 3], keepdim=True)
        cam = (weights * self.activations).sum(dim=1, keepdim=True)
        cam = F.relu(cam)

        # 入力サイズにリサイズして 0-1 正規化
        cam = F.interpolate(cam, size=x.shape[2:], mode="bilinear", align_corners=False)
        cam = (cam - cam.min()) / (cam.max() - cam.min() + 1e-8)
        return cam.squeeze().detach().numpy()


# モデルの最後の畳み込み層を指定
grad_cam = GradCAM(model, model.features[-3])  # SimpleCNN の場合の例

# 1 枚の画像で試す
img_tensor = images[0:1].to(device)
heatmap = grad_cam.generate(img_tensor)

plt.imshow(heatmap, cmap="jet", alpha=0.5)
plt.colorbar()
plt.title("Grad-CAM ヒートマップ")
plt.savefig("gradcam.png", dpi=150)
```

赤い領域ほどモデルが強く注目した部分です。分類ミスがあったときに「どこを見ていたか」を確認するデバッグ手法として使われます。

---

## 代表的なアーキテクチャ

| モデル | 特徴 | 用途 |
|--------|------|------|
| ResNet-18/50 | 残差接続。転移学習の定番 | 汎用画像分類 |
| EfficientNet | 精度・速度のバランスが良い | モバイル・本番 |
| ViT（Vision Transformer）| Transformer を画像に適用 | 大規模学習 |

---


## 確認問題

1. CNN（画像認識） は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [PyTorch 入門](PyTorch入門) — テンソル・学習ループの基礎
- [深層学習入門](深層学習入門) — バッチ正規化・ドロップアウトの理論
- [Hugging Face 入門](HuggingFace入門) — ViT など Transformer 系の利用

---

[← ホームへ](Home)
