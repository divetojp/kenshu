# Hugging Face 入門

**Hugging Face** は機械学習モデルを公開・共有・利用するためのプラットフォームです。`transformers` ライブラリを使うと BERT・GPT・Stable Diffusion などの最先端モデルを数行のコードで利用できます。

Hugging Face を学ぶときは、「モデル置き場」と「Python ライブラリ」を分けて考えると理解しやすくなります。Web 上の Hugging Face Hub にはモデルやデータセットが公開されており、手元の Python では `transformers` や `datasets` を使ってそれらを読み込みます。

最初に、モデル利用に必要なライブラリをインストールします。`transformers` はモデル本体、`datasets` は公開データセット、`accelerate` は GPU などの実行環境を扱いやすくするための補助ライブラリです。

```bash
pip install transformers datasets accelerate
```

このコマンドは、Hugging Face を使うための道具を Python 環境に追加します。実際のモデルはこの時点ですべて入るわけではなく、後で `from_pretrained()` や `pipeline()` を呼んだときに必要なモデルファイルがダウンロードされます。

---

## はじめて読む人へ

Hugging Face は、機械学習モデルやデータセットを共有・利用するためのプラットフォームです。特に自然言語処理や画像認識のモデルを簡単に試せます。


### 読む前に押さえること

- Pipeline API は、モデルをすぐ試すための高レベルな入口です。
- Tokenizer は、文章をモデルが扱えるID列に変換します。
- ファインチューニングは、既存モデルを自分のデータに合わせて再学習することです。

### 読み終えたら説明できること

- pipeline、tokenizer、model の関係を説明できる。
- 公開モデルを使って推論する流れを理解できる。
- Hub にモデルを公開する意味を説明できる。

---

## Pipeline API（最速で試す）

Pipeline API は、モデル選び、トークナイズ、推論、出力整形をまとめて行う高レベルな入口です。初めてモデルを試すときは、細かい設定を知らなくても動かせるため便利です。

ただし、Pipeline の裏側では tokenizer と model が動いています。実務で入力形式を調整したり、バッチ処理を高速化したり、ファインチューニングしたりする段階では、裏側の流れも理解しておく必要があります。

Pipeline を使うと、入力文を渡すだけで結果が返ってきます。しかし内部では、文章をトークン ID に変換し、ニューラルネットワークへ入力し、出力をラベルや文章に戻す、という複数の処理が順番に行われています。まずは「便利な入口」として使い、慣れてきたら tokenizer と model の役割に分解して理解します。

```python
from transformers import pipeline

# 感情分析
classifier = pipeline("sentiment-analysis",
                       model="distilbert-base-uncased-finetuned-sst-2-english")
result = classifier("I love this product! It works perfectly.")
# → [{'label': 'POSITIVE', 'score': 0.9998}]

# 日本語感情分析
ja_classifier = pipeline("sentiment-analysis",
                          model="daigo/bert-base-japanese-sentiment")
result = ja_classifier("この映画は最高でした！")

# テキスト生成
generator = pipeline("text-generation", model="gpt2")
text = generator("The future of AI is", max_length=50, num_return_sequences=3)

# 固有表現認識（NER）
ner = pipeline("ner", model="dslim/bert-base-NER")
result = ner("Barack Obama was born in Honolulu, Hawaii.")

# 翻訳（日本語→英語）
translator = pipeline("translation", model="Helsinki-NLP/opus-mt-ja-en")
result = translator("データサイエンスは面白い")

# 要約
summarizer = pipeline("summarization", model="facebook/bart-large-cnn")
text = "長い文章..." * 10
result = summarizer(text, max_length=130, min_length=30)
```

どの例でも、`pipeline("タスク名", model="モデル名")` で処理器を作り、その処理器に入力データを渡しています。`model` に指定している文字列は、Hugging Face Hub 上のモデル ID です。初回実行時はモデルがダウンロードされるため、少し時間がかかることがあります。

出力はタスクによって異なります。感情分析ならラベルと確率、NER なら単語ごとの固有表現、翻訳や要約なら生成された文章が返ります。実務では、返ってきた値をそのまま信じるのではなく、スコアの意味、入力文の長さ、モデルが学習した言語やドメインを確認します。

---

## Tokenizer の使い方

Tokenizer は、人間の文章をモデルが扱える数値列に変換する部品です。深層学習モデルは文字列を直接読めないため、文章を単語やサブワードに分割し、それぞれを ID に置き換えます。

この「文章を数値にする」処理は、使うモデルごとに決まっています。たとえば日本語 BERT 用の tokenizer と英語 GPT 用の tokenizer では、分割の仕方も語彙表も異なります。そのため、基本的にはモデルと同じ名前の tokenizer を読み込みます。

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("cl-tohoku/bert-base-japanese-whole-word-masking")

# テキスト → トークン ID
text = "データサイエンスを学んでいます"
encoded = tokenizer(text, return_tensors="pt", truncation=True, max_length=128)

print(encoded["input_ids"])       # トークン ID
print(encoded["attention_mask"])  # 実際のトークン=1、パディング=0

# ID → テキスト（デコード）
ids = encoded["input_ids"][0]
print(tokenizer.decode(ids))

# バッチ処理（複数文章を同時に）
texts = ["文章1", "文章2", "文章3"]
batch = tokenizer(texts, padding=True, truncation=True,
                  max_length=128, return_tensors="pt")
```

`input_ids` は、トークンを語彙表の番号に変換したものです。`attention_mask` は、どこまでが実際の文章で、どこからが長さをそろえるための埋め草かを表します。

`padding=True` は複数の文章の長さをそろえる指定、`truncation=True` は長すぎる文章を切り詰める指定です。モデルには最大入力長があるため、長文を扱うときは「どこで切るか」「切ってよい情報か」を考える必要があります。

---

## ファインチューニング

事前学習済みモデルを自分のデータでさらに学習させます。少ないデータでも高精度を実現できます。

事前学習済みモデルは、大量の一般的な文章から言語のパターンを学んでいます。しかし、自分の課題に必要な分類基準までは知らないことがあります。たとえば「映画レビューが肯定的か否定的か」「問い合わせ文が解約希望かどうか」といった具体的な判断基準は、手元のラベル付きデータで追加学習させます。

次のコードは、短い文章とラベルから 2 クラス分類モデルを作る最小例です。実際の授業や実務では、データ数を増やし、訓練データと検証データを分け、過学習していないかを確認します。

```python
from transformers import (
    AutoTokenizer, AutoModelForSequenceClassification,
    TrainingArguments, Trainer
)
from datasets import Dataset
import numpy as np
from sklearn.metrics import accuracy_score

# データ準備
train_data = {
    "text": ["面白い映画", "つまらない映画", "最高の作品", "見る価値なし"],
    "label": [1, 0, 1, 0]
}
train_dataset = Dataset.from_dict(train_data)

# トークナイズ
model_name = "cl-tohoku/bert-base-japanese"
tokenizer = AutoTokenizer.from_pretrained(model_name)

def tokenize(examples):
    return tokenizer(examples["text"], truncation=True, max_length=128)

train_dataset = train_dataset.map(tokenize, batched=True)

# モデル（出力層を 2 クラス分類に変更）
model = AutoModelForSequenceClassification.from_pretrained(model_name, num_labels=2)

# 学習設定
training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=8,
    learning_rate=2e-5,
    warmup_steps=100,
    weight_decay=0.01,
    logging_steps=10,
    save_strategy="epoch",
)

def compute_metrics(pred):
    labels = pred.label_ids
    preds = pred.predictions.argmax(-1)
    return {"accuracy": accuracy_score(labels, preds)}

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    compute_metrics=compute_metrics,
)

trainer.train()
trainer.save_model("./my_model")  # モデルを保存
```

`AutoModelForSequenceClassification` は、文章分類用の出力層を持つモデルを読み込みます。`num_labels=2` は、出力クラスが 2 種類であることを表します。

`TrainingArguments` には、学習回数、バッチサイズ、学習率、保存方法などを指定します。`Trainer` は、モデル、データ、評価関数、学習設定をまとめて、学習ループを実行するための高レベル API です。初学者はまずこの形で全体像をつかみ、慣れてきたら PyTorch の学習ループを自分で書くと理解が深まります。

ファインチューニングで大切なのは、コードを動かすことよりも「何をモデルに学ばせているか」を説明できることです。入力は文章、正解はラベル、損失関数は予測と正解のズレを測るもの、という対応を意識してください。

---

## datasets ライブラリ

Hugging Face のデータセットハブから公開データセットを取得できます。

機械学習では、モデルだけでなくデータセットの扱いも重要です。`datasets` ライブラリを使うと、公開データセットを同じ形式で読み込み、訓練データ・テストデータに分かれた状態で扱えます。

```python
from datasets import load_dataset

# 有名なデータセット
imdb = load_dataset("imdb")           # 映画レビュー（英語）
squad = load_dataset("squad")         # 質問応答データ
mnist = load_dataset("mnist")         # 手書き数字

# pandas に変換
df = imdb["train"].to_pandas()
print(df.head())

# フィルタリング
positive_reviews = imdb["train"].filter(lambda x: x["label"] == 1)
```

`load_dataset("imdb")` のように指定すると、Hub 上のデータセットを取得します。返ってくるオブジェクトは、`train` や `test` などの分割を持っていることが多く、`imdb["train"]` のようにアクセスします。

pandas に変換すると、表として確認したり可視化したりしやすくなります。一方で、巨大なデータセットを全部 pandas に変換するとメモリを使いすぎることがあるため、学習では `datasets` の形式のまま扱うことも多いです。

---

## モデルを Hugging Face Hub に公開

Hugging Face Hub は、学習済みモデルを共有する場所でもあります。自分でファインチューニングしたモデルを Hub に置くと、別の PC、チームメンバー、デプロイ環境から同じモデル ID で読み込めます。

公開するときは、モデル本体だけでなく tokenizer も一緒にアップロードします。tokenizer が違うと、同じ文章でも違う ID 列に変換されてしまい、モデルが正しく動かないからです。

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer

model = AutoModelForSequenceClassification.from_pretrained("./my_model")
tokenizer = AutoTokenizer.from_pretrained("cl-tohoku/bert-base-japanese")

# Hub にアップロード（要: huggingface-cli login）
model.push_to_hub("your-username/my-sentiment-model")
tokenizer.push_to_hub("your-username/my-sentiment-model")

# 公開したモデルを使う
from transformers import pipeline
clf = pipeline("sentiment-analysis", model="your-username/my-sentiment-model")
```

`push_to_hub()` を使うには、事前に `huggingface-cli login` でログインしておく必要があります。アップロード後は、`your-username/my-sentiment-model` のようなモデル ID で読み込めます。

モデルを公開するときは、ライセンス、学習データの説明、想定用途、苦手な入力、評価結果を Model Card に書くのが望ましいです。特に人に影響する判断へ使うモデルでは、精度だけでなく、どのようなデータで学習したかを説明できることが重要です。

---

## よく使うモデルの選び方

| タスク | 推奨モデル（日本語） |
|--------|---------------------|
| テキスト分類 | `cl-tohoku/bert-base-japanese` |
| 固有表現認識 | `jureki/bert-base-nlu-japanese` |
| 質問応答 | `cl-tohoku/bert-base-japanese` + SQuAD-JA |
| テキスト生成 | `rinna/japanese-gpt-1b` |
| 文の埋め込み | `sonoisa/sentence-bert-base-ja-mean-tokens` |

---


## 確認問題

1. Hugging Face 入門 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [NLP 基礎](NLP基礎) — テキスト処理の基礎知識
- [LLM 活用入門](LLM活用入門) — API 経由での LLM 活用
- [実験管理](実験管理) — ファインチューニングの実験記録
- [機械学習基盤](機械学習基盤) — 本番環境でのモデル提供

---

[← ホームへ](Home)
