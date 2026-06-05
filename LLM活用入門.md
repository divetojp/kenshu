# LLM 活用入門

**大規模言語モデル（Large Language Model, LLM）** は、膨大なテキストデータから事前学習した巨大なニューラルネットワークです。翻訳・要約・コード生成・質問応答など多様なタスクを、追加学習なしにプロンプト（指示文）だけでこなせます。

LLM をアプリに組み込む主な方法は 2 つです。**API 経由の利用**（Claude・GPT-4 などのクラウドモデル）と、**ローカル実行**（Llama など OSS モデル）。本ページでは API 利用を中心に解説します。OSS モデルのローカル実行は [Hugging Face 入門](HuggingFace入門) を参照してください。

---

## はじめて読む人へ

LLM は、大量のテキストから言語のパターンを学習したモデルです。文章生成だけでなく、要約、分類、検索補助、ツール呼び出しなど、さまざまなタスクに使えます。


### 読む前に押さえること

- プロンプトは、LLMに出す指示文です。
- RAG は、外部知識を検索してから回答に使う仕組みです。
- Tool Use は、モデルが必要に応じて関数やAPIを呼び出す設計です。

### 読み終えたら説明できること

- ゼロショット、Few-shot、RAG の違いを説明できる。
- LLMの出力を検証する必要性を理解できる。
- Tool Use の基本的な処理ループを読める。

---

## API の基本的な使い方

### Anthropic API（Claude）

Claude は Anthropic が提供するモデルです。API キーを環境変数 `ANTHROPIC_API_KEY` に設定してから使います。

まず、Python から Anthropic API を呼び出すための公式ライブラリをインストールします。API キーはコードに直接書かず、環境変数として設定しておきます。

```bash
pip install anthropic
```

次のコードでは、クライアントを作成し、ユーザーのメッセージをモデルに送っています。API 呼び出しは「モデル名」「出力の最大長」「会話メッセージ」を指定する、と読むと全体像がつかみやすいです。

```python
import anthropic

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY 環境変数を参照

message = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Pythonで素数を列挙する関数を書いてください"}
    ]
)
print(message.content[0].text)
```

`messages` は会話履歴を表します。ここではユーザーの 1 メッセージだけですが、実際のチャットアプリでは過去のやり取りをリストとして渡します。

`max_tokens` は出力の長さの上限です。長くしすぎるとコストや応答時間が増え、短すぎると回答が途中で切れることがあります。

### OpenAI API（ChatGPT / GPT-4）

OpenAI API も、基本構造は同じです。クライアントを作り、モデル名とメッセージを指定して、返ってきたレスポンスから本文を取り出します。

```python
from openai import OpenAI

client = OpenAI()  # OPENAI_API_KEY 環境変数を参照

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "あなたはデータサイエンスの専門家です"},
        {"role": "user",   "content": "特徴量エンジニアリングのベストプラクティスを教えてください"}
    ],
    temperature=0.7,   # 0: 確定的、1: 多様な出力
    max_tokens=500,
)
print(response.choices[0].message.content)
```

`system` メッセージは、モデルの役割や回答方針を指定するために使います。`temperature` は出力のランダムさを調整する値で、分類や抽出のように安定性がほしい場合は低め、アイデア出しのように多様性がほしい場合は高めにします。

---

## プロンプトエンジニアリング

LLM の性能は「どう指示するか」に大きく左右されます。同じモデルでも、プロンプトの書き方で回答の質が大きく変わります。

### 基本テクニック

**ゼロショット**は例を見せずにそのままタスクを指示する方法です。シンプルなタスクには十分機能します。**Few-shot** は「例を見せてから質問する」方法で、出力フォーマットや判断基準を伝えたいときに有効です。**Chain-of-Thought（CoT）** は「まずステップごとに考えてから答えを出してください」と指示することで、モデルに推論ステップを踏ませる手法です。算数や論理推論のような問題で特に効果があります。

次の 3 つのプロンプトは、同じモデルに対してどの程度の文脈を与えるかが違います。プロンプトは「お願い文」ではなく、モデルへ渡す入力仕様だと考えると設計しやすくなります。

```python
# ゼロショット
prompt_zero = "以下の文章のネガポジを判定してください：「この映画は最高でした」"

# Few-shot：例を見せてから質問
prompt_few = """
以下のルールでネガポジ判定してください：

例：
「最悪でした」→ Negative
「素晴らしい体験」→ Positive
「普通だった」→ Neutral

判定してください：「思ったより良かった」
"""

# Chain-of-Thought：推論ステップを要求
prompt_cot = """
問題を解く前に、まずステップごとに考えてから答えを出してください。

問題：1から100の整数で、3の倍数か5の倍数の合計を求めてください。
"""
```

ゼロショットは短く書けますが、出力形式がぶれやすいことがあります。Few-shot は例によって判断基準と出力形式を示せます。CoT は推論を促すための考え方ですが、実務では最終回答だけを出させる、根拠を別に検証するなど、用途に合わせた設計が必要です。

### システムプロンプトで役割を定義

システムプロンプトはモデルの「基本方針」を定義します。ユーザーの指示より前に読み込まれ、モデルの振る舞いを制御します。

次の例では、LLM を「データサイエンティスト」として振る舞わせ、DataFrame の概要と質問を渡して分析方針を出させています。

```python
import anthropic
client = anthropic.Anthropic()

def analyze_data_with_llm(df_info: str, question: str) -> str:
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="""あなたはデータサイエンティストです。
ユーザーがデータの概要を説明し、分析の質問をします。
Python の pandas/sklearn を使った具体的な分析手法を提案してください。
コードを含む回答を心がけてください。""",
        messages=[{
            "role": "user",
            "content": f"データの概要：\n{df_info}\n\n質問：{question}"
        }]
    )
    return message.content[0].text

import pandas as pd
df = pd.read_csv("data.csv")
answer = analyze_data_with_llm(df.describe().to_string(), "欠損値のパターンを分析して前処理の方針を提案してください")
print(answer)
```

`df.describe().to_string()` は、データの統計的な概要を文字列にしたものです。LLM は実際の DataFrame を直接操作しているわけではなく、渡された文字情報をもとに提案している点に注意してください。

LLM の提案は、分析者の判断を置き換えるものではありません。欠損値の処理や特徴量設計は、データの意味、業務上の制約、評価方法と合わせて検証します。

---

## Tool Use（関数呼び出し）

Tool Use（OpenAI では Function Calling と呼ぶ）は、**LLM が自分でツールを選択し、必要な引数を生成して関数を呼び出せる**仕組みです。

通常の API 呼び出しでは、LLM はテキストを返すだけです。Tool Use を使うと、LLM は「このタスクにはこの関数が必要だ」と判断し、その関数の引数を JSON で返します。アプリ側はその引数で実際の関数を実行し、結果を LLM に戻すことで、LLM は最終的な回答を生成します。

Tool Use の本質は、LLM に実行権限を直接渡すのではなく、「どの関数をどんな引数で呼びたいか」を提案させることです。実際に関数を実行するのはアプリ側であり、ここで安全性の確認や権限制御を行います。

!!! info ""
    ユーザーの質問
    ↓
    LLM が「ツールが必要」と判断
    ↓
    ツール名と引数 JSON を返す
    ↓
    アプリが実際の関数を実行
    ↓
    実行結果を LLM に渡す
    ↓
    LLM が結果をもとに最終回答を生成
この流れにより、LLM は天気 API、計算関数、データベース検索、社内ツールなどを組み合わせた回答ができるようになります。ただし、ツールの設計が曖昧だと、誤った引数や危険な操作につながるため、入力スキーマを明確に書くことが重要です。

### Anthropic API での Tool Use

次のコードは、天気取得と計算という 2 つのツールを LLM に提示する例です。`tools` には、ツール名、説明、入力の JSON Schema を定義します。

```python
import anthropic
import json

client = anthropic.Anthropic()

# 使えるツールを定義（JSON Schema でパラメータを指定）
tools = [
    {
        "name": "get_weather",
        "description": "指定した都市の現在の天気を取得します",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "都市名（例：Tokyo, Osaka）"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度の単位"
                }
            },
            "required": ["city"]
        }
    },
    {
        "name": "calculate",
        "description": "数式を計算します",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "計算する数式（例：2 + 2 * 3）"
                }
            },
            "required": ["expression"]
        }
    }
]

# 実際の関数
def get_weather(city: str, unit: str = "celsius") -> dict:
    # 実際のアプリでは外部 API を呼ぶ（ここはダミー）
    return {"city": city, "temperature": 22, "unit": unit, "condition": "晴れ"}

def calculate(expression: str) -> dict:
    result = eval(expression)  # 本番では安全な評価器を使うこと
    return {"expression": expression, "result": result}

# ツールの実行マップ
tool_functions = {"get_weather": get_weather, "calculate": calculate}

def chat_with_tools(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]

    # 1回目：LLM がツールを選択
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        tools=tools,
        messages=messages
    )

    # ツール呼び出しが必要な場合
    while response.stop_reason == "tool_use":
        tool_results = []

        for block in response.content:
            if block.type == "tool_use":
                # LLM が選んだツールを実行
                func = tool_functions[block.name]
                result = func(**block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": json.dumps(result, ensure_ascii=False)
                })

        # ツール結果を LLM に渡して続きを生成
        messages.append({"role": "assistant", "content": response.content})
        messages.append({"role": "user", "content": tool_results})

        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

    # テキストブロックを結合して返す
    return "".join(block.text for block in response.content if hasattr(block, "text"))

print(chat_with_tools("東京の天気と、15 × 28 の計算結果を教えてください"))
```

`response.stop_reason == "tool_use"` の間は、モデルがツール呼び出しを求めています。アプリ側は `block.name` でツール名を見て、`block.input` を引数として実際の Python 関数を実行します。

この例の `eval(expression)` は学習用の簡略化です。本番でユーザー入力をそのまま `eval` すると任意コード実行の危険があります。数式評価には、安全なパーサーや限定された計算関数を使います。

### データ分析での Tool Use 活用例

LLM に pandas の関数を「ツール」として渡すことで、自然言語で DataFrame の分析を指示できます。

この設計では、LLM が DataFrame を直接読むのではなく、「どの列をどのように集計したいか」をツール引数として指定します。実際の集計は Python 側の関数が行います。

```python
import pandas as pd
import anthropic
import json

client = anthropic.Anthropic()

df = pd.DataFrame({
    "name": ["Alice", "Bob", "Charlie", "Dave"],
    "score": [85, 92, 78, 95],
    "department": ["DS", "CS", "DS", "CS"]
})

tools = [
    {
        "name": "get_stats",
        "description": "指定した列の統計情報を取得します",
        "input_schema": {
            "type": "object",
            "properties": {
                "column": {"type": "string", "description": "列名"},
                "group_by": {"type": "string", "description": "グループ化する列名（省略可）"}
            },
            "required": ["column"]
        }
    }
]

def get_stats(column: str, group_by: str = None) -> dict:
    if group_by:
        result = df.groupby(group_by)[column].describe().to_dict()
    else:
        result = df[column].describe().to_dict()
    return result

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    tools=tools,
    messages=[{"role": "user", "content": "部門別のスコアの平均と標準偏差を教えてください"}]
)

if response.stop_reason == "tool_use":
    for block in response.content:
        if block.type == "tool_use":
            result = get_stats(**block.input)
            print(f"ツール呼び出し: {block.name}({block.input})")
            print(f"結果: {result}")
```

自然言語の質問を、検証可能なデータ処理に変換できるのが Tool Use の強みです。ただし、列名の存在確認、アクセス権限、集計対象の制限などはアプリ側で実装する必要があります。

---

## RAG（検索拡張生成）

RAG は、LLM にすべてを記憶させるのではなく、必要な情報を外部から検索してプロンプトに渡す方法です。社内文書、教材、FAQ のように更新される知識を扱うときに役立ちます。

重要なのは、検索結果をそのまま答えにするのではなく、LLM が回答を作るための根拠として渡すことです。回答の品質は、モデルだけでなく、検索される文書の質や分割方法にも左右されます。

LLM は学習データの期限（Knowledge Cutoff）以降の情報を知りません。**RAG（Retrieval-Augmented Generation）** は「ユーザーの質問に関連するドキュメントを検索し、そのドキュメントを LLM に一緒に渡す」ことで、最新情報や社内文書に回答できるようにします。

RAG のパイプラインは以下の 3 ステップです：
1. **インデックス作成**：ドキュメントをベクトル化して検索可能な状態にする
2. **検索（Retrieval）**：質問と類似度の高いドキュメントを取得する
3. **生成（Generation）**：取得したドキュメントを文脈として LLM に渡し、回答を生成する

次のコードは、RAG の考え方を最小構成で示す例です。実用システムではベクトルデータベースを使うことが多いですが、ここでは TF-IDF とコサイン類似度で「質問に近い文書を探す」流れを見ます。

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity
import anthropic

documents = [
    "環境構築にはDockerを使います。docker compose upで起動してください。",
    "デプロイはCloudflare Pagesを使い、main ブランチにマージすると自動実行されます。",
    "コードレビューはGitHubのPull Requestで行います。",
]

def retrieve(query: str, docs: list[str], top_k: int = 2) -> list[str]:
    vectorizer = TfidfVectorizer()
    tfidf_matrix = vectorizer.fit_transform(docs + [query])
    sims = cosine_similarity(tfidf_matrix[-1], tfidf_matrix[:-1])[0]
    top_indices = sims.argsort()[::-1][:top_k]
    return [docs[i] for i in top_indices]

def rag_query(question: str) -> str:
    relevant_docs = retrieve(question, documents)
    context = "\n\n".join(relevant_docs)

    client = anthropic.Anthropic()
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=[{
            "role": "user",
            "content": f"""以下の参考資料をもとに質問に答えてください。

参考資料：
{context}

質問：{question}"""
        }]
    )
    return message.content[0].text

print(rag_query("デプロイの方法を教えてください"))
```

`retrieve` は、質問と文書を同じベクトル空間に変換し、類似度の高い文書を返します。`rag_query` は、その検索結果を `context` としてプロンプトに埋め込み、LLM に回答を作らせます。

実用的な RAG では TF-IDF の代わりにベクトルデータベース（Chroma・Pinecone・pgvector など）と埋め込みモデルを使うのが一般的です。精度が大幅に向上します。

RAG で重要なのは、「検索できる文書を整えること」です。文書の分割が大きすぎると関係ない情報が混ざり、小さすぎると文脈が失われます。検索結果の質が低ければ、LLM の回答品質も下がります。

---

## 構造化出力（JSON モード）

LLM にテキストではなく JSON を出力させることで、プログラムから扱いやすくなります。出力形式を明示的に指示し、`json.loads` でパースします。

アプリに組み込む場合、自然文の回答よりも JSON のような構造化データのほうが扱いやすいことがあります。名前、年齢、カテゴリ、スコアなどを後続処理で使うなら、出力形式を固定します。

```python
import json

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    messages=[{
        "role": "user",
        "content": """以下の文章から人物情報をJSONで抽出してください。

文章：「田中太郎（28歳、東京在住）はデータサイエンティストとして働いています。」

出力形式：{"name": string, "age": int, "location": string, "job": string}
JSONのみ出力してください。"""
    }]
)

data = json.loads(message.content[0].text)
print(data)  # {'name': '田中太郎', 'age': 28, 'location': '東京', 'job': 'データサイエンティスト'}
```

`json.loads` は、JSON 文字列を Python の辞書に変換します。ただし、モデルが余計な説明文を混ぜるとパースに失敗します。実務では JSON モードやスキーマ指定、再試行、バリデーションを組み合わせます。

---

## データ分析への応用：バッチ処理

大量のテキストデータを分類・分析する場合は、安価で高速なモデルを使います。

アンケート自由記述や問い合わせ文の分類では、1 件ずつ API を呼ぶと時間とコストが増えます。複数件をまとめて処理するバッチ化によって効率化できます。

```python
import pandas as pd

df = pd.read_csv("survey_results.csv")  # 自由記述アンケート

def batch_classify(texts: list[str], categories: list[str]) -> list[str]:
    prompt = f"""以下のカテゴリに分類してください：{categories}

テキストを1行ずつ出力し、各行を「テキスト::カテゴリ」の形式にしてください：

{chr(10).join(f"{i+1}. {t}" for i, t in enumerate(texts))}
"""
    message = client.messages.create(
        model="claude-haiku-4-5-20251001",  # 高速・安価なモデルで処理
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )
    return [line.split("::")[-1].strip() for line in message.content[0].text.strip().split("\n")]

categories = ["バグ報告", "機能要望", "質問", "その他"]
df["category"] = batch_classify(df["comment"].tolist()[:100], categories)
```

この例では、コメントを番号付きでまとめてプロンプトに入れ、各行をカテゴリへ分類させています。出力を `::` で分割してカテゴリだけを取り出していますが、実務では JSON 配列で返させるほうが壊れにくいです。

大量処理では、コスト、レート制限、失敗時の再実行、個人情報の扱いを設計する必要があります。LLM に送る前に、不要な個人情報をマスクすることも検討します。

---

## モデルの使い分け

| モデル | 用途 | 特徴 |
|--------|------|------|
| claude-haiku-4-5 | 大量処理・単純なタスク | 最速・最安 |
| claude-sonnet-4-6 | 通常の開発・分析・Tool Use | バランス型 |
| claude-opus-4-8 | 複雑な推論・高精度が必要 | 最高品質 |
| gpt-4o | OpenAI エコシステム | 幅広い対応 |

Tool Use や複雑な推論には `sonnet` 以上を推奨します。`haiku` は単純な分類・抽出タスクに向いています。

---


## 確認問題

1. LLM 活用入門 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [NLP 基礎](NLP基礎) — テキスト処理の基礎・Transformer の仕組み
- [Hugging Face 入門](HuggingFace入門) — OSS モデルのローカル実行・ファインチューニング
- [FastAPI](FastAPI) — LLM 機能を API として公開

---

[← ホームへ](Home)
