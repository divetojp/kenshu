# PyTorch 入門

Python で最も使われている深層学習フレームワークです。NumPy に似た API と動的計算グラフ（実行時にグラフが構築される）が特徴で、研究・実務ともに標準的な選択肢です。

PyTorch を使うには、テンソル計算の本体である `torch` と、画像データセットや画像変換を扱う `torchvision` をインストールします。

```bash
pip install torch torchvision
```

深層学習では、ただの Python のリストではなく、GPU 上で高速に計算できるテンソルを使います。PyTorch は、そのテンソル計算、自動微分、ニューラルネットワークの部品をまとめて提供します。

---

## はじめて読む人へ

PyTorch は、深層学習モデルを実装するためのライブラリです。NumPy に似たテンソル計算に、自動微分とニューラルネットワークの部品が加わっています。


### 読む前に押さえること

- テンソルは、数値を多次元に並べたデータ構造です。
- autograd は、計算の流れから勾配を自動で求めます。
- 学習ループは、予測、損失計算、逆伝播、更新の繰り返しです。

### 読み終えたら説明できること

- テンソルと NumPy 配列の関係を説明できる。
- `nn.Module` でモデルを定義する流れを読める。
- 基本的な学習ループを理解できる。

---

## テンソル（Tensor）

NumPy の ndarray に GPU 対応・自動微分が加わったものです。

テンソルは、スカラー、ベクトル、行列、さらに高次元配列をまとめて扱うためのデータ構造です。画像なら「枚数・チャンネル・高さ・幅」、文章なら「バッチサイズ・系列長・埋め込み次元」のように、形状が意味を持ちます。

```python
import torch
import numpy as np

# テンソルの作成
t = torch.tensor([1.0, 2.0, 3.0])
m = torch.zeros(3, 4)
r = torch.randn(2, 3)    # 標準正規分布

# NumPy との相互変換
a = np.array([1.0, 2.0, 3.0])
t = torch.from_numpy(a)     # NumPy → Tensor
a2 = t.numpy()               # Tensor → NumPy

# GPU に移す（CUDA が使えるとき）
device = "cuda" if torch.cuda.is_available() else "cpu"
t = t.to(device)
print(f"使用デバイス: {device}")

# 形状操作
t = torch.randn(2, 3, 4)
print(t.shape)         # torch.Size([2, 3, 4])
t_flat = t.view(2, -1) # → (2, 12)
t_perm = t.permute(2, 0, 1)  # 軸の並べ替え → (4, 2, 3)
```

`torch.tensor` は値からテンソルを作り、`torch.zeros` や `torch.randn` は指定した形のテンソルを作ります。`shape` を確認する習慣は非常に重要で、深層学習のエラーの多くは「期待した形と違うテンソルを渡している」ことから起きます。

`view` はテンソルの形を変え、`permute` は軸の順番を入れ替えます。画像処理や時系列処理では、ライブラリごとに期待する軸の順番が違うことがあるため、形状操作を読めることが大切です。

---

## 自動微分（autograd）

計算グラフを自動で構築し、任意のパラメータへの勾配を自動計算します。

ニューラルネットワークの学習では、損失を小さくするために各パラメータをどちらへどれくらい動かせばよいかを知る必要があります。この「損失をパラメータで微分した値」が勾配です。

PyTorch の autograd は、テンソルに対する計算の履歴を記録し、`backward()` が呼ばれたときに連鎖律で勾配を計算します。

```python
# 勾配を計算するテンソルに requires_grad=True
x = torch.tensor(3.0, requires_grad=True)

# 順伝播
y = x ** 2 + 2 * x + 1   # y = (x+1)²

# 逆伝播（dy/dx を計算）
y.backward()

print(f"dy/dx at x=3: {x.grad}")  # 2x + 2 = 8
```

`requires_grad=True` を付けた `x` は、勾配を追跡する対象になります。`y.backward()` を呼ぶと、`y` を `x` で微分した値が `x.grad` に保存されます。

実際のニューラルネットでは、`x` の代わりに重みやバイアスが勾配を持ちます。optimizer はその勾配を使って、重みを少しずつ更新します。

---

## ニューラルネットの実装

PyTorch では、モデルを `nn.Module` を継承したクラスとして定義します。`__init__` で層を用意し、`forward` で入力がどの順番で層を通るかを書きます。

この分け方により、「モデルが持つ部品」と「データの流れ」を分けて読めます。エラーが出たときは、各層に入るテンソルの形と出るテンソルの形を確認することが大切です。

### nn.Module でモデルを定義

`nn.Module` は、PyTorch でモデルを作るときの基本クラスです。層を部品として持ち、入力テンソルがどの順番で流れるかを `forward` に書きます。

```python
import torch.nn as nn
import torch.nn.functional as F

class MLP(nn.Module):
    """多層パーセプトロン"""
    def __init__(self, input_dim, hidden_dim, output_dim):
        super().__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.bn1 = nn.BatchNorm1d(hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim)
        self.fc3 = nn.Linear(hidden_dim, output_dim)
        self.dropout = nn.Dropout(p=0.3)

    def forward(self, x):
        x = F.relu(self.bn1(self.fc1(x)))
        x = self.dropout(x)
        x = F.relu(self.fc2(x))
        x = self.fc3(x)          # 出力層には活性化関数をかけない
        return x

model = MLP(input_dim=28*28, hidden_dim=256, output_dim=10)
print(model)
```

この MLP は、入力を線形層に通し、BatchNorm、ReLU、Dropout を挟みながら、最後に 10 クラス分のスコアを出します。出力層に活性化関数をかけないのは、`CrossEntropyLoss` が内部で softmax 相当の計算を含むためです。

`forward` は自分で直接呼ぶより、`model(x)` の形で呼ぶのが一般的です。PyTorch が内部で `forward` を呼び、フックや autograd などの仕組みも正しく動きます。

### Dataset と DataLoader

`Dataset` は「1 件のデータをどう取り出すか」、`DataLoader` は「それをミニバッチにまとめてどう渡すか」を担当します。大きなデータを一度に全部モデルへ入れるのではなく、少しずつ分けて学習します。

```python
from torch.utils.data import Dataset, DataLoader

class TabularDataset(Dataset):
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.long)

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

dataset = TabularDataset(X_train.to_numpy(), y_train.to_numpy())
loader = DataLoader(dataset, batch_size=32, shuffle=True)
```

`__len__` はデータ件数、`__getitem__` は指定された番号のデータを返します。`DataLoader` はこれを使って、`batch_size=32` なら 32 件ずつ取り出し、`shuffle=True` なら学習ごとに順番を混ぜます。

---

## 学習ループ

学習ループは、深層学習の中心です。1 回のミニバッチごとに「予測する → 損失を計算する → 勾配を計算する → パラメータを更新する」という流れを繰り返します。

```python
model = MLP(input_dim=X_train.shape[1], hidden_dim=128, output_dim=num_classes)
model = model.to(device)

criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)

def train_epoch(model, loader, optimizer, criterion):
    model.train()   # 学習モード（Dropout・BN が有効）
    total_loss = 0
    for X_batch, y_batch in loader:
        X_batch, y_batch = X_batch.to(device), y_batch.to(device)
        optimizer.zero_grad()        # 勾配をリセット
        outputs = model(X_batch)     # 順伝播
        loss = criterion(outputs, y_batch)
        loss.backward()              # 逆伝播
        optimizer.step()             # パラメータ更新
        total_loss += loss.item()
    return total_loss / len(loader)

def evaluate(model, loader, criterion):
    model.eval()    # 推論モード（Dropout・BN が固定）
    total_loss, correct = 0, 0
    with torch.no_grad():           # 勾配を計算しない（メモリ節約）
        for X_batch, y_batch in loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            outputs = model(X_batch)
            total_loss += criterion(outputs, y_batch).item()
            correct += (outputs.argmax(1) == y_batch).sum().item()
    return total_loss / len(loader), correct / len(loader.dataset)

# 学習実行
for epoch in range(50):
    train_loss = train_epoch(model, train_loader, optimizer, criterion)
    val_loss, val_acc = evaluate(model, val_loader, criterion)
    if epoch % 10 == 0:
        print(f"Epoch {epoch:3d}: loss={train_loss:.4f}  val_acc={val_acc:.3f}")
```

`model.train()` は Dropout や BatchNorm を学習モードにし、`model.eval()` は推論モードにします。評価時に `torch.no_grad()` を使うのは、勾配計算を止めてメモリと計算時間を節約するためです。

`optimizer.zero_grad()` を毎回呼ぶ理由は、PyTorch では勾配が累積される仕様だからです。前回の勾配を消してから `loss.backward()` し、`optimizer.step()` でパラメータを更新します。

---

## モデルの保存と読み込み

学習したモデルは、後で推論や再学習に使えるよう保存します。PyTorch では、モデル全体ではなく、重みの辞書である `state_dict` を保存する方法がよく使われます。

```python
# 保存（state_dict のみ保存するのが推奨）
torch.save(model.state_dict(), "model.pth")

# 読み込み
model2 = MLP(input_dim=..., hidden_dim=128, output_dim=10)
model2.load_state_dict(torch.load("model.pth", map_location=device))
model2.eval()
```

読み込むときは、同じ構造のモデルを先に作ってから、保存した重みを流し込みます。`map_location=device` を指定すると、GPU で保存したモデルを CPU 環境で読むときの問題を避けやすくなります。

---

## MNIST で試してみる（手書き数字分類）

MNIST は、0〜9 の手書き数字画像を分類する定番データセットです。深層学習の入門では、データ読み込み、前処理、モデル学習、評価の一連の流れを確認するためによく使われます。

```python
from torchvision import datasets, transforms

transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

train_data = datasets.MNIST("./data", train=True, download=True, transform=transform)
test_data = datasets.MNIST("./data", train=False, transform=transform)

train_loader = DataLoader(train_data, batch_size=64, shuffle=True)
test_loader = DataLoader(test_data, batch_size=64)

model = MLP(28*28, 256, 10).to(device)
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()

for epoch in range(10):
    train_epoch(model, train_loader, optimizer, criterion)
    _, acc = evaluate(model, test_loader, criterion)
    print(f"Epoch {epoch+1}: test acc = {acc:.3f}")
# → 10 エポックで約 97〜98% の精度
```

`transforms.ToTensor()` は画像を PyTorch のテンソルに変換し、`Normalize` は画素値を標準化します。`datasets.MNIST(..., download=True)` は、必要ならデータセットを自動でダウンロードします。

この例では、28×28 ピクセルの画像を 1 次元に伸ばして MLP に入力しています。画像の空間構造をより活かす方法として、次の [CNN](CNN) で畳み込みニューラルネットワークを学びます。

---


## 確認問題

1. PyTorch 入門 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [深層学習入門](深層学習入門) — ニューラルネットの理論
- [CNN（画像認識）](CNN) — 画像に特化したアーキテクチャ
- [RNN・時系列](RNN-時系列) — 系列データの扱い
- [実験管理](実験管理) — 学習ログの記録

---

[← ホームへ](Home)
