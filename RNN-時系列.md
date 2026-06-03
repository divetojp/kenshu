# RNN・時系列

**過去の情報を記憶しながら系列データを処理する** ニューラルネットワークです。株価・気温・センサーデータなど「時間の順序に意味がある」データに使います。現在は多くのタスクで Transformer に置き換えられていますが、軽量モデルとして今も実用的です。

時系列データでは、データの順番そのものが情報になります。普通の表データでは行の順番を入れ替えても意味が変わらないことが多いですが、時系列では昨日、今日、明日という順序を壊すと予測問題として成り立たなくなります。

この章では、RNN・LSTM・GRU の細かい数式よりも先に、**時系列データをどのように学習データへ変換し、どの形でモデルに渡すか** を重視します。ここが分かると、PyTorch のコードも「暗記するもの」ではなく「形を合わせている処理」として読めるようになります。

## この章で学ぶこと

- 時系列データを `(サンプル数, 系列長, 特徴量数)` の形に変換する
- スライディングウィンドウで「過去から未来を予測する」データセットを作る
- LSTM と GRU が普通の RNN より長い依存関係を扱いやすい理由を理解する
- PyTorch で LSTM 予測モデルを定義し、学習ループを読む
- 古典的なラグ特徴量と深層学習モデルの使い分けを理解する

---

## はじめて読む人へ

RNN・時系列モデルは、順番に意味があるデータを扱うための方法です。過去の値から未来を予測するには、データを時間順に切り出し、モデルへ渡せる形に整える必要があります。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- window は過去何ステップを見るか、horizon は何ステップ先を予測するかです。
- 時系列では、未来の情報を訓練に混ぜないようにします。
- LSTM や GRU は、長い系列の情報を扱いやすくするためのモデルです。

### 読み終えたら説明できること

- 時系列データを3次元テンソルに変換する理由を説明できる。
- スライディングウィンドウで学習データを作れる。
- ラグ特徴量と深層学習モデルの使い分けを理解できる。

---

## 時系列予測で最初に決めること

時系列予測では、モデルを選ぶ前に次の 3 つを決めます。

| 決めること | 意味 | 例 |
|------------|------|----|
| window | 過去何ステップを見るか | 過去 30 日分 |
| horizon | 何ステップ先を予測するか | 次の 7 日分 |
| features | 各時点で使う特徴量 | 気温、湿度、曜日、祝日 |

たとえば「過去 30 日の気温から、次の 7 日の気温を予測する」なら、`window=30`、`horizon=7` です。1 日ごとに気温だけを使うなら特徴量数は 1、気温・湿度・気圧を使うなら特徴量数は 3 になります。

注意点は、未来の情報を学習時に使わないことです。時系列では、ランダムに train/test split すると未来のデータが訓練側に混ざることがあります。基本的には古い期間を訓練データ、新しい期間を検証・テストデータにします。

---

## 時系列データの形式

PyTorch の RNN 系モデルは、入力を 3 次元テンソルとして受け取ります。`batch_first=True` にすると、形は `(batch, seq, feature)` です。

- `batch` は一度に処理するサンプル数です
- `seq` は系列長、つまり過去何ステップを見るかです
- `feature` は各時点に含まれる特徴量数です

```python
import numpy as np
import pandas as pd
import torch

# 時系列データの典型的な形状
# (バッチサイズ, 系列長, 特徴量数)
# 例：30 日分の気温を 10 サンプル
data = torch.randn(10, 30, 1)  # (10, 30, 1)

# 多変量時系列（気温・湿度・気圧）
data_multi = torch.randn(10, 30, 3)  # (10, 30, 3)
```

この例では、`data` は「10 個のサンプルがあり、それぞれ 30 日分の気温 1 つを持つ」データです。`data_multi` は、各時点に気温・湿度・気圧の 3 つの特徴量があるケースです。

### スライディングウィンドウで学習データを作成

時系列予測では、1 本の長い時系列から、たくさんの学習サンプルを切り出します。これをスライディングウィンドウと呼びます。

たとえば 1,000 日分の気温データがあるとします。過去 30 日から次の 7 日を予測するなら、1〜30 日目を入力、31〜37 日目を正解にします。次に 2〜31 日目を入力、32〜38 日目を正解にします。このように窓を 1 日ずつずらして学習データを作ります。

```python
def create_sequences(data: np.ndarray, window: int, horizon: int):
    """
    data: (T,) の 1 次元時系列
    window: 何ステップを見て予測するか
    horizon: 何ステップ先を予測するか
    """
    X, y = [], []
    for i in range(len(data) - window - horizon + 1):
        X.append(data[i : i + window])
        y.append(data[i + window : i + window + horizon])
    return np.array(X)[..., np.newaxis], np.array(y)  # (N, window, 1), (N, horizon)

# 例：過去 30 日 → 次の 7 日予測
temps = np.sin(np.linspace(0, 100, 1000)) + np.random.randn(1000) * 0.1
X, y = create_sequences(temps, window=30, horizon=7)
```

返り値の `X` は `(N, 30, 1)`、`y` は `(N, 7)` になります。`N` は作れたサンプル数です。`X` の最後の `1` は特徴量数を表します。PyTorch の LSTM に渡すため、1 次元の時系列でも `[..., np.newaxis]` で特徴量次元を追加しています。

時系列の分割では、作った `X` と `y` をランダムに混ぜる前に注意が必要です。検証やテストでは未来側のサンプルを使い、実際の予測に近い条件で評価します。

---

## LSTM（Long Short-Term Memory）

LSTM は、系列の中で長く残すべき情報と忘れてよい情報を分けて扱うためのモデルです。普通のRNNでは、時間が進むにつれて昔の情報が薄れやすくなります。

LSTM はセル状態という長期記憶の通り道を持ち、ゲートによって情報を通す量を調整します。これにより、少し前の値だけでなく、より長い時間前の傾向も予測に使いやすくなります。

単純な RNN は長い系列で勾配消失が起きます。LSTM はゲート機構で「記憶をどれだけ保持・忘れるか」を学習します。

普通の RNN は、系列を 1 ステップずつ読みながら隠れ状態を更新します。しかし系列が長くなると、昔の情報が徐々に薄れていきます。たとえば「30 日前の気温が今週の傾向に効いている」ような場合、単純な RNN ではその情報を保ちにくくなります。

LSTM は、短期的な隠れ状態とは別に、長期記憶にあたるセル状態を持ちます。忘れる情報、追加する情報、出力に使う情報をゲートで調整することで、長い系列でも重要な情報を残しやすくしています。

```python
import torch.nn as nn

class LSTMForecaster(nn.Module):
    def __init__(self, input_size, hidden_size, num_layers, output_size):
        super().__init__()
        self.lstm = nn.LSTM(
            input_size=input_size,
            hidden_size=hidden_size,
            num_layers=num_layers,
            batch_first=True,          # (batch, seq, feature)
            dropout=0.2,               # 層間のドロップアウト
            bidirectional=False,
        )
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        # x: (batch, seq_len, input_size)
        out, (h_n, c_n) = self.lstm(x)
        # out: (batch, seq_len, hidden_size)
        last = out[:, -1, :]   # 最後のタイムステップのみ使う
        return self.fc(last)   # (batch, output_size)

model = LSTMForecaster(input_size=1, hidden_size=64, num_layers=2, output_size=7)
```

`input_size=1` は、各時点の特徴量が 1 つであることを表します。`hidden_size=64` は、LSTM が内部で持つ表現の次元数です。大きくすると表現力は上がりますが、過学習や計算量も増えます。

`output_size=7` は、次の 7 ステップをまとめて予測するという意味です。`forward()` では LSTM の全時点の出力 `out` から最後の時点 `out[:, -1, :]` だけを取り出し、全結合層で 7 日分の予測に変換しています。

### 学習

学習ループは、通常の PyTorch モデルと同じ構造です。入力 `X_b` から予測 `pred` を作り、正解 `y_b` との誤差を `MSELoss` で計算し、`loss.backward()` で勾配を求めて、`optimizer.step()` で重みを更新します。

```python
from torch.utils.data import TensorDataset, DataLoader

X_t = torch.tensor(X, dtype=torch.float32)
y_t = torch.tensor(y, dtype=torch.float32)

dataset = TensorDataset(X_t, y_t)
loader = DataLoader(dataset, batch_size=32, shuffle=True)

criterion = nn.MSELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)

for epoch in range(50):
    model.train()
    for X_b, y_b in loader:
        optimizer.zero_grad()
        pred = model(X_b)
        loss = criterion(pred, y_b)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)  # 勾配クリッピング
        optimizer.step()
```

`clip_grad_norm_` は勾配が大きくなりすぎるのを防ぐ処理です。RNN 系モデルでは、系列を通して勾配を伝えるため、勾配消失だけでなく勾配爆発が起きることもあります。勾配クリッピングは、学習を安定させるための定番テクニックです。

実際の評価では、訓練データの loss だけでなく、未来側に残しておいた検証データの loss を見ます。訓練 loss だけが下がって検証 loss が悪化する場合は、過学習の可能性があります。

> **情報工学メモ：LSTM の 4 つのゲート**  
> LSTM は内部に「セル状態（長期記憶）」と「隠れ状態（短期記憶）」を持ちます。① **忘却ゲート**：前の記憶をどれだけ忘れるか。② **入力ゲート**：新しい情報をどれだけ記憶するか。③ **出力ゲート**：セル状態のどの部分を出力に使うか。④ **GRU** は LSTM を簡略化して 2 ゲートにしたもので、パラメータが少なく速いです。

---

## GRU（Gated Recurrent Unit）

LSTM より軽量で、多くのタスクで同等の性能を発揮します。

GRU は LSTM を少し簡略化したモデルです。セル状態を分けず、隠れ状態だけで情報を保持します。そのためパラメータ数が少なく、学習が速いことがあります。

データ量が少ない、モデルを軽くしたい、まずベースラインを作りたい、という場合は GRU から試すのもよい選択です。LSTM と GRU のどちらが良いかはデータによるため、検証データで比較します。

次のコードは、LSTM と同じように最後の時点の出力を取り出し、全結合層で予測値に変換する GRU モデルです。構造は LSTM より少し単純ですが、入力テンソルの形は同じです。

```python
class GRUForecaster(nn.Module):
    def __init__(self, input_size, hidden_size, output_size):
        super().__init__()
        self.gru = nn.GRU(input_size, hidden_size, num_layers=2,
                           batch_first=True, dropout=0.2)
        self.fc = nn.Linear(hidden_size, output_size)

    def forward(self, x):
        out, _ = self.gru(x)
        return self.fc(out[:, -1, :])
```

`out[:, -1, :]` は、各サンプルについて最後の時点の隠れ表現を取り出しています。時系列全体を読んだあとの要約表現として使い、そこから未来の値を予測します。

---

## Transformer を使った時系列予測

近年は LSTM より Transformer ベースのモデルが高精度です。

Transformer は、系列の各時点同士の関係を Attention で直接見ます。RNN のように左から順に状態を更新するのではなく、系列全体の中で「どの時点を重視するか」を学習します。

ただし、Transformer はデータ量や計算資源を多く必要とすることがあります。小さな時系列データでは、LSTM や GRU、あるいはラグ特徴量を使った機械学習モデルのほうが扱いやすい場合もあります。

次の簡易実装では、各時点の特徴量を `d_model` 次元へ写し、Transformer Encoder に渡しています。RNN と違い、系列を順番に 1 ステップずつ処理するのではなく、Attention で時点同士の関係を見ます。

```python
class TransformerForecaster(nn.Module):
    def __init__(self, input_size, d_model, nhead, num_layers, output_size):
        super().__init__()
        self.input_proj = nn.Linear(input_size, d_model)
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=d_model, nhead=nhead, dim_feedforward=d_model*4,
            dropout=0.1, batch_first=True
        )
        self.transformer = nn.TransformerEncoder(encoder_layer, num_layers=num_layers)
        self.fc = nn.Linear(d_model, output_size)

    def forward(self, x):
        x = self.input_proj(x)          # (batch, seq, d_model)
        x = self.transformer(x)
        return self.fc(x[:, -1, :])     # 最終ステップ

model = TransformerForecaster(input_size=1, d_model=64, nhead=4, num_layers=2, output_size=7)
```

この簡易例では、入力特徴量を `input_proj` で `d_model` 次元に変換し、Transformer Encoder に渡しています。実務では、時点の順序を表す位置情報、祝日や曜日などのカレンダー特徴量、複数系列を扱う工夫も重要になります。

---

## sklearn での古典的な手法（参考）

深層学習を使わなくても、時系列予測はできます。代表的なのが、過去の値を特徴量として表に加える方法です。この過去の値を **ラグ特徴量** と呼びます。

たとえば `lag_7` は「7 日前の値」です。今日の気温を予測するとき、昨日、1 週間前、2 週間前の値を特徴量として使う、という考え方です。

次のコードでは、1 日前、7 日前、14 日前、30 日前の値を列として追加しています。これにより、時系列データを通常の表形式データとして扱えるようになります。

```python
from sklearn.preprocessing import MinMaxScaler

# 多変量時系列の特徴量として「ラグ特徴量」を作る
df = pd.DataFrame({"temp": temps})
for lag in [1, 7, 14, 30]:
    df[f"lag_{lag}"] = df["temp"].shift(lag)

df.dropna(inplace=True)
X = df.drop("temp", axis=1)
y = df["temp"]
```

この方法の利点は、表データとして扱えることです。ランダムフォレスト、勾配ブースティング、線形回帰など、これまで学んだ sklearn のモデルをそのまま使えます。データ量が少ないときや説明しやすさを重視するときは、まずラグ特徴量のベースラインを作るのがおすすめです。

`shift(lag)` によって過去の値を現在行にずらして持ってきます。先頭の数行は過去データが存在しないため欠損になり、`dropna` で削除しています。

---

## モデル選びの目安

| 状況 | 最初に試す方法 |
|------|----------------|
| データが少ない | ラグ特徴量 + sklearn |
| 季節性や周期性が強い | ラグ特徴量、移動平均、曜日・月特徴量 |
| 系列が長く、過去の文脈が重要 | LSTM / GRU |
| データ量が多く、複雑な関係を扱いたい | Transformer 系モデル |
| 説明しやすさが重要 | ラグ特徴量 + 線形モデル / 決定木系 |

深層学習モデルは強力ですが、常に最初の選択肢とは限りません。時系列予測では、単純なベースラインを作ってから複雑なモデルに進むと、改善が本物かどうか判断しやすくなります。

---

## よくある失敗

### 時間順を無視してランダム分割する

時系列では、未来のデータを訓練に混ぜると評価が甘くなります。原則として、古い期間で学習し、新しい期間で評価します。

### スケーリングを全期間で fit する

標準化や正規化も、訓練期間だけで `fit` します。全期間で平均や最大値を計算すると、未来の情報を使ったことになります。

### window と horizon を意識しない

過去 7 日で明日を予測する問題と、過去 90 日で次の 30 日を予測する問題は難しさが違います。問題設定を明確にしないと、モデルの良し悪しを比較できません。

### 深層学習だけで解こうとする

データ数が少ない場合、LSTM や Transformer は過学習しやすくなります。まずは「前日と同じ値を予測する」「7 日前と同じ値を予測する」などの単純なベースラインと比較してください。

---


## 確認問題

1. 時系列データでランダムに train/test split するのが危険な理由を説明してください。
2. `window=30`、`horizon=7` は何を意味しますか。
3. LSTM が普通の RNN より長い系列を扱いやすい理由を説明してください。
4. ラグ特徴量を使う方法と LSTM を使う方法の違いを説明してください。

## ミニ演習

1. 100 個の数値からなる 1 次元時系列を作り、`window=10`、`horizon=1` の `X` と `y` を作ってください。
2. 作った `X` の形が `(N, 10, 1)` になることを確認してください。
3. `lag_1`、`lag_7`、`lag_14` を持つ DataFrame を作り、欠損行を削除してください。
4. LSTM とラグ特徴量モデルのどちらを先に試すべきか、データ量・説明しやすさ・精度の観点で考えてください。

---

## 関連ページ

- [PyTorch 入門](PyTorch入門) — テンソル・学習ループ
- [深層学習入門](深層学習入門) — バックプロパゲーション・BPTT
- [NLP 基礎](NLP基礎) — 系列モデルのテキスト応用
- [データ可視化](データ可視化) — 予測結果のプロット

---

[← ホームへ](Home)
