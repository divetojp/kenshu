# 日本語 NLP 実践

[NLP基礎](NLP基礎.md) で学んだ概念（トークナイズ・TF-IDF・Word2Vec）を**日本語に適用する**ための実践ガイドです。日本語は英語と違い「単語の区切りがない」という根本的な違いがあり、形態素解析という前処理が必須です。

---

## はじめて読む人へ

英語の NLP ライブラリをそのまま日本語に使おうとするとうまくいきません。「東京都知事」は「東京 / 都 / 知事」なのか「東京都 / 知事」なのかが自明ではないからです。このページでは日本語固有の処理をゼロから学びます。

### 読む前に押さえること

- [NLP基礎](NLP基礎.md) — トークン化・TF-IDF・Word2Vec の概念

### 読み終えたら説明できること

- 形態素解析（MeCab・SudachiPy）で日本語をトークン化できる
- 日本語テキストの前処理パイプラインを実装できる
- 日本語の文書分類・感情分析を実装できる

---

## 英語 vs 日本語の NLP の違い

| 観点 | 英語 | 日本語 |
|------|------|--------|
| 単語の区切り | スペースで自明 | **形態素解析が必要** |
| 文字体系 | アルファベットのみ | 漢字・ひらがな・カタカナ・英数字が混在 |
| 活用形 | 比較的シンプル | 複雑（行く/行った/行って/行けば...） |
| ストップワード | a, the, is... | は・が・の・を・に... |
| 固有表現 | 大文字が手がかり | 手がかりなし |

---

## 形態素解析：MeCab

**形態素解析**は文を最小の意味単位（形態素）に分割し、品詞などの情報を付与します。

```bash
# インストール（Mac）
brew install mecab mecab-ipadic
pip install mecab-python3 unidic-lite
```

```python
import MeCab

tagger = MeCab.Tagger()

text = "東京都知事が新しい政策を発表しました。"
result = tagger.parse(text)
print(result)
# 東京    名詞,固有名詞,地域,一般,*,*,東京,トウキョウ,トウキョウ
# 都      名詞,接尾,地域,*,*,*,都,ト,ト
# 知事    名詞,一般,*,*,*,*,知事,チジ,チジ
# が      助詞,格助詞,一般,*,*,*,が,ガ,ガ
# ...

# 名詞だけ抽出（キーワード抽出）
def extract_nouns(text: str) -> list[str]:
    node = tagger.parseToNode(text)
    nouns = []
    while node:
        pos = node.feature.split(",")[0]
        if pos == "名詞" and node.surface:
            nouns.append(node.surface)
        node = node.next
    return nouns

print(extract_nouns(text))
# ['東京', '都', '知事', '政策']
```

---

## SudachiPy（より現代的な形態素解析器）

MeCab より辞書の更新が頻繁で、新語・固有名詞に強いです。

```bash
pip install sudachipy sudachidict-core
```

```python
from sudachipy import tokenize, SplitMode

# SplitMode A: 最小単位、B: 中間、C: 最大単位
text = "東京都知事が新しい政策を発表しました。"

for mode, label in [(SplitMode.A, "最小"), (SplitMode.C, "最大")]:
    tokens = tokenize(text, mode)
    print(f"[{label}] {[t.surface() for t in tokens]}")
# [最小] ['東京', '都', '知事', 'が', '新しい', '政策', 'を', '発表', 'し', 'まし', 'た', '。']
# [最大] ['東京都', '知事', 'が', '新しい', '政策', 'を', '発表', 'しました', '。']

# 品詞フィルタリング
def get_base_forms(text, pos_filter=("名詞", "動詞", "形容詞")):
    tokens = tokenize(text, SplitMode.C)
    return [
        t.dictionary_form()          # 原形（活用形を正規化）
        for t in tokens
        if t.part_of_speech()[0] in pos_filter
        and t.surface() not in {"する", "ある", "いる", "なる"}  # 軽動詞除去
    ]

print(get_base_forms("東京都知事が新しい政策を発表しました"))
# ['東京都', '知事', '新しい', '政策', '発表']
```

---

## 前処理パイプライン

```python
import re
import unicodedata

def normalize_text(text: str) -> str:
    """日本語テキストの正規化"""
    # Unicode 正規化（全角英数 → 半角、半角カナ → 全角）
    text = unicodedata.normalize("NFKC", text)
    # URL・メールアドレスを除去
    text = re.sub(r'https?://\S+', '', text)
    text = re.sub(r'\S+@\S+', '', text)
    # HTML タグを除去
    text = re.sub(r'<[^>]+>', '', text)
    # 連続する空白・改行を1つに
    text = re.sub(r'\s+', ' ', text).strip()
    return text

# ストップワード（日本語）
STOPWORDS_JA = set([
    "の", "に", "は", "を", "た", "が", "で", "て", "と", "し",
    "れ", "さ", "ある", "いる", "も", "する", "から", "な", "こと",
    "として", "い", "や", "れる", "など", "なっ", "ない", "この",
    "ため", "その", "あっ", "よう", "また", "もの", "という",
])

def tokenize_ja(text: str, remove_stopwords: bool = True) -> list[str]:
    """日本語テキストをトークン化して返す"""
    text = normalize_text(text)
    tokens = get_base_forms(text)
    if remove_stopwords:
        tokens = [t for t in tokens if t not in STOPWORDS_JA and len(t) > 1]
    return tokens

# 使用例
sample = "東京都の新しい政策について、知事が記者会見で発表しました。"
print(tokenize_ja(sample))
# ['東京都', '政策', '知事', '記者会見', '発表']
```

---

## TF-IDF による文書分類（日本語版）

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report

# サンプルデータ（スポーツ / 政治ニュース）
docs = [
    "サッカー日本代表がワールドカップで勝利した",
    "野球のペナントレースが佳境を迎えている",
    "バスケットボールのNBAで日本人選手が活躍",
    "オリンピックで日本選手がメダルを獲得した",
    "国会で新しい予算案が可決された",
    "首相が外交政策について記者会見を開いた",
    "参議院選挙で与党が過半数を確保した",
    "政府が経済対策を発表した",
]
labels = [0, 0, 0, 0, 1, 1, 1, 1]  # 0: スポーツ、1: 政治

# 前処理してトークン化
tokenized = [" ".join(tokenize_ja(doc)) for doc in docs]
print("トークン化後:", tokenized[0])

# TF-IDF + ロジスティック回帰
X_train, X_test, y_train, y_test = train_test_split(
    tokenized, labels, test_size=0.25, random_state=42
)

vectorizer = TfidfVectorizer()
X_train_vec = vectorizer.fit_transform(X_train)
X_test_vec  = vectorizer.transform(X_test)

clf = LogisticRegression()
clf.fit(X_train_vec, y_train)
y_pred = clf.predict(X_test_vec)
print(classification_report(y_test, y_pred, target_names=["スポーツ", "政治"]))
```

---

## 日本語の感情分析

### oseti（日本語感情分析ライブラリ）

```python
# pip install oseti
import oseti

analyzer = oseti.Analyzer()

texts = [
    "この映画は本当に感動的で素晴らしかった",
    "最悪の体験だった。二度と行きたくない",
    "普通の映画だと思います",
]

for text in texts:
    score = analyzer.analyze(text)
    print(f"テキスト: {text}")
    print(f"  感情スコア: {score}（正: ポジティブ、負: ネガティブ）\n")
```

### 日本語 BERT（事前学習済みモデル）

```python
from transformers import pipeline, AutoTokenizer, AutoModelForSequenceClassification

# 東北大学の日本語 BERT モデル
model_name = "cl-tohoku/bert-base-japanese-whole-word-masking"
tokenizer = AutoTokenizer.from_pretrained(model_name)

# 感情分析（ファインチューニング済みモデル）
sentiment_pipeline = pipeline(
    "sentiment-analysis",
    model="daigo/bert-base-japanese-sentiment",
    tokenizer="daigo/bert-base-japanese-sentiment",
)

texts = [
    "この商品は品質が良くて満足しています",
    "配送が遅く、品質も期待以下でした",
]

for text in texts:
    result = sentiment_pipeline(text)[0]
    print(f"テキスト: {text}")
    print(f"  感情: {result['label']}, スコア: {result['score']:.3f}\n")
```

---

## 固有表現認識（NER）

```python
# GiNZA: 日本語 NLP ライブラリ（spaCy ベース）
# pip install ginza ja-ginza
import spacy

nlp = spacy.load("ja_ginza")

text = "トヨタ自動車の社長が東京で記者会見を開き、来年の新型車を発表した。"
doc = nlp(text)

print("固有表現:")
for ent in doc.ents:
    print(f"  {ent.text} → {ent.label_}")
# トヨタ自動車 → ORG（組織）
# 東京 → GPE（地政学的エンティティ）

print("\n依存関係:")
for token in doc:
    if token.dep_ != "punct":
        print(f"  {token.text} --[{token.dep_}]--> {token.head.text}")
```

---

## 日本語テキストの埋め込み（Embeddings）

```python
from transformers import AutoTokenizer, AutoModel
import torch

def get_sentence_embedding(text: str, model_name: str = "cl-tohoku/bert-base-japanese") -> torch.Tensor:
    """日本語テキストをベクトルに変換"""
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    model = AutoModel.from_pretrained(model_name)

    inputs = tokenizer(text, return_tensors="pt", truncation=True, max_length=128)
    with torch.no_grad():
        outputs = model(**inputs)

    # CLS トークンの埋め込みを文ベクトルとして使用
    return outputs.last_hidden_state[:, 0, :].squeeze()

# 類似度計算
from torch.nn.functional import cosine_similarity

text1 = "東京は日本の首都です"
text2 = "日本の政治の中心は東京にあります"
text3 = "サッカーは世界で人気のスポーツです"

emb1 = get_sentence_embedding(text1)
emb2 = get_sentence_embedding(text2)
emb3 = get_sentence_embedding(text3)

sim12 = cosine_similarity(emb1.unsqueeze(0), emb2.unsqueeze(0)).item()
sim13 = cosine_similarity(emb1.unsqueeze(0), emb3.unsqueeze(0)).item()
print(f"テキスト1 vs テキスト2（意味的に近い）: {sim12:.3f}")
print(f"テキスト1 vs テキスト3（意味的に遠い）: {sim13:.3f}")
```

---

## 確認問題

1. 英語の NLP ライブラリがそのまま日本語に使えない理由を「形態素解析」の観点から説明してください。
2. MeCab と SudachiPy の分割モード（A/B/C）の違いを、「東京都知事」を例に説明してください。
3. 日本語の感情分析で「oseti（辞書ベース）」と「BERT（ニューラル）」の使い分けを、速度・精度・データ量の観点で説明してください。

---

## 関連ページ

- [NLP基礎](NLP基礎.md) — トークン化・TF-IDF・Word2Vec の英語版
- [Transformer・Attention](Transformer-Attention.md) — BERT の仕組み
- [Hugging Face 入門](HuggingFace入門.md) — 日本語 BERT モデルの活用
- [LLM 活用入門](LLM活用入門.md) — ChatGPT API で日本語テキスト処理
