# Streamlit

## Streamlit とは

**Python スクリプトだけでインタラクティブな Web アプリを作れるライブラリ** です。

HTML・CSS・JavaScript を書かずに、pandas のデータや scikit-learn の予測結果をそのままブラウザで可視化できます。データサイエンティストが分析結果を素早くデモとして共有するのに最適です。

まず、Streamlit を Python 環境にインストールします。Streamlit は「Python スクリプトを上から実行し、その結果を Web 画面として描画する」仕組みです。

```bash
pip install streamlit
```

インストール後は、`streamlit run app.py` のように Python ファイルを指定して起動します。Notebook と違い、他の人がブラウザから操作できる画面として共有しやすいのが特徴です。

---

## はじめて読む人へ

Streamlit は、Python だけで簡単なWebアプリを作るためのライブラリです。Notebook で作った分析やグラフを、他の人がブラウザで触れる形にできます。

コードやコマンドが出てきたら、最初から全部を覚えようとしなくて大丈夫です。まずは「何を入力し、何が処理され、何が出力されるのか」を文章で説明できるように読むと、手を動かす前の理解が安定します。

### 読む前に押さえること

- Streamlit は、上から順にPythonコードを実行して画面を作ります。
- 入力UIを使うと、ユーザーが条件を変えながら結果を確認できます。
- 重い処理にはキャッシュを使うと、表示が速くなります。

### 読み終えたら説明できること

- `st.write`、`st.dataframe`、`st.plotly_chart` の役割を説明できる。
- CSVを読み込んで画面に表示できる。
- 簡単な予測アプリの流れを理解できる。

---

## 最初のアプリ

Streamlit アプリは、普通の Python ファイルとして書きます。`st.title` や `st.write` のような関数を呼ぶと、その位置に対応する UI がブラウザ上に表示されます。

```python
# app.py
import streamlit as st

st.title("はじめての Streamlit アプリ")
st.write("Hello, World!")

name = st.text_input("名前を入力してください")
if name:
    st.success(f"こんにちは、{name}さん！")
```

このコードでは、タイトル、本文、テキスト入力欄、成功メッセージを順番に表示しています。Streamlit はユーザーが入力を変えるたびにスクリプトを上から再実行し、その時点の状態に合わせて画面を作り直します。

```bash
streamlit run app.py
# → http://localhost:8501 でブラウザが自動的に開く
```

`streamlit run` は、Python ファイルを Streamlit アプリとして起動するコマンドです。通常は `http://localhost:8501` で確認できます。

コードを保存するたびにブラウザが自動更新されます。

---

## よく使う UI 部品

Streamlit の UI 部品は、ユーザーから入力を受け取るために使います。スライダー、セレクトボックス、テキスト入力などを使うと、コードを書き換えなくても条件を変えられるアプリになります。

UI 部品の戻り値は普通の Python の変数として扱えます。たとえばスライダーで選んだ数値を使って DataFrame を絞り込んだり、セレクトボックスで選んだ列をグラフの軸にしたりできます。

### テキスト表示

テキスト表示系の関数は、画面に情報を出すための基本部品です。見出し、説明文、Markdown、コード例などを用途に応じて使い分けます。

```python
st.title("大見出し")
st.header("中見出し")
st.subheader("小見出し")
st.text("等幅テキスト")
st.markdown("**Markdown** も使えます")
st.write("なんでも表示できる汎用コマンド")

# コードブロック
st.code("""
def hello():
    return "world"
""", language="python")
```

`st.write` は汎用的な表示関数で、文字列、数値、DataFrame など多くの型を表示できます。教材やデモでは便利ですが、画面設計を明確にしたい場合は `st.title`、`st.markdown`、`st.dataframe` のように専用関数を使い分けます。

### 入力ウィジェット

入力ウィジェットは、ユーザーが値を選んだり入力したりするための部品です。戻り値は Python の変数に入るため、そのまま条件分岐やフィルタに使えます。

```python
name    = st.text_input("名前", placeholder="田中 太郎")
age     = st.number_input("年齢", min_value=0, max_value=120, value=20)
level   = st.slider("レベル", 1, 10, 5)
agree   = st.checkbox("利用規約に同意する")
genre   = st.selectbox("ジャンル", ["SF", "ミステリー", "恋愛"])
genres  = st.multiselect("好きなジャンル（複数可）", ["SF", "ミステリー", "恋愛"])
clicked = st.button("送信")

if clicked:
    st.write(f"{name}（{age}歳）が選択: {genre}")
```

`st.button` はクリックされた瞬間だけ `True` になります。スライダーやセレクトボックスは、現在選ばれている値を毎回返します。この違いを意識すると、ボタンで実行する処理と常に反映するフィルタを分けやすくなります。

### ファイルアップロード

ファイルアップロードは、ユーザーが手元のデータをアプリに渡すための入口です。CSV を読み込んでその場で可視化する分析ツールを作るときによく使います。

```python
uploaded = st.file_uploader("CSV ファイルをアップロード", type="csv")
if uploaded:
    df = pd.read_csv(uploaded)
    st.dataframe(df)
```

`uploaded` には、アップロードされたファイルのように扱えるオブジェクトが入ります。何もアップロードされていないときは `None` なので、`if uploaded:` で確認してから読み込みます。

---

## pandas との組み合わせ

Streamlit と pandas を組み合わせると、CSV の読み込み、条件による絞り込み、統計量の表示を短いコードでアプリ化できます。Notebook で作った分析を、他の人が触れるダッシュボードに変える第一歩です。

```python
import streamlit as st
import pandas as pd

st.title("成績分析ダッシュボード")

# CSV 読み込み（キャッシュで高速化）
@st.cache_data
def load_data(path: str) -> pd.DataFrame:
    return pd.read_csv(path)

df = load_data("scores.csv")

# フィルタ UI
min_score = st.slider("最低スコア", 0, 100, 60)
filtered  = df[df["score"] >= min_score]

# テーブル表示
st.subheader(f"該当者: {len(filtered)} 名")
st.dataframe(filtered.sort_values("score", ascending=False))

# 統計情報
col1, col2, col3 = st.columns(3)
col1.metric("平均スコア", f"{filtered['score'].mean():.1f}")
col2.metric("最高スコア", filtered["score"].max())
col3.metric("合格率", f"{(filtered['score'] >= 60).mean():.1%}")
```

`@st.cache_data` は、同じ入力に対する読み込み結果を保存し、再実行のたびに CSV を読み直さないようにします。Streamlit は操作のたびにスクリプトを再実行するため、重い読み込み処理にはキャッシュが重要です。

`st.columns(3)` は画面を 3 つの列に分けます。`metric` は数値の要約を目立たせる表示で、平均、最大値、合格率などの KPI を見せるのに向いています。

---

## グラフの表示

### matplotlib / seaborn

matplotlib や seaborn の図は、まず `fig` を作ってから `st.pyplot(fig)` で Streamlit に渡します。Notebook で作っていた図を、そのまま Web アプリに載せる感覚です。

```python
import matplotlib.pyplot as plt
import seaborn as sns

fig, ax = plt.subplots(figsize=(8, 4))
ax.hist(df["score"], bins=20, edgecolor="black")
ax.set_xlabel("スコア")
ax.set_ylabel("人数")
st.pyplot(fig)
```

この例では、スコアの分布をヒストグラムで表示しています。`fig, ax = plt.subplots()` の形にすると、Streamlit に渡す図が明確になります。

### Plotly（インタラクティブ）

Plotly は、マウスで拡大したりホバーで詳細を見たりできるグラフを作れます。Streamlit では `st.plotly_chart` でそのまま表示できます。

```python
import plotly.express as px

fig = px.scatter(df, x="study_hours", y="score",
                 color="passed", hover_data=["name"],
                 title="学習時間とスコアの関係")
st.plotly_chart(fig, use_container_width=True)
```

`use_container_width=True` を指定すると、グラフが画面幅に合わせて表示されます。ダッシュボードでは、ユーザーの画面サイズが異なるため、横幅に自然に追従する設定が便利です。

---

## scikit-learn モデルを組み込む

Streamlit は、学習済みモデルを読み込んで予測デモを作る用途にも向いています。ユーザーが特徴量を入力し、その場で予測結果を見ることで、モデルの挙動を直感的に確認できます。

```python
import streamlit as st
import pandas as pd
import joblib

st.title("予測デモ")

# モデルを読み込む（起動時に1回だけ）
@st.cache_resource
def load_model():
    return joblib.load("pipeline.pkl")

pipeline = load_model()

# 入力フォーム
st.header("特徴量を入力してください")
col1, col2 = st.columns(2)
with col1:
    feature1 = st.number_input("特徴量1", value=5.1)
    feature2 = st.number_input("特徴量2", value=3.5)
with col2:
    feature3 = st.number_input("特徴量3", value=1.4)
    feature4 = st.number_input("特徴量4", value=0.2)

if st.button("予測する"):
    X = [[feature1, feature2, feature3, feature4]]
    pred = pipeline.predict(X)[0]
    proba = pipeline.predict_proba(X)[0].max()

    st.success(f"予測結果: **クラス {pred}**（確信度: {proba:.1%}）")
    st.progress(proba)
```

`@st.cache_resource` は、モデルのように大きくて何度も読み込みたくないオブジェクトをキャッシュするために使います。スライダーや入力欄を操作するたびにモデルを読み直すと遅くなるため、起動時に 1 回だけ読み込む設計にします。

ここで読み込んでいる `pipeline.pkl` は、前処理とモデルをまとめて保存したものを想定しています。入力フォームの値を 2 次元リスト `X` にし、`predict` と `predict_proba` に渡す流れは、機械学習アプリの基本です。

---

## レイアウトの整理

アプリが大きくなると、すべてを上から並べるだけでは見づらくなります。サイドバー、タブ、展開ボックスを使うと、設定・グラフ・生データを整理して配置できます。

```python
# サイドバー
with st.sidebar:
    st.header("設定")
    theme = st.selectbox("テーマ", ["明るい", "暗い"])
    show_raw = st.checkbox("生データを表示")

# タブ
tab1, tab2, tab3 = st.tabs(["📊 グラフ", "📋 データ", "⚙️ 設定"])
with tab1:
    st.write("グラフをここに表示")
with tab2:
    if show_raw:
        st.dataframe(df)

# 展開ボックス（折りたたみ）
with st.expander("詳細情報を見る"):
    st.write("ここに詳細を書く")
```

`with st.sidebar:` の中に書いた部品はサイドバーに表示されます。`st.tabs` は画面を複数のタブに分け、`st.expander` は詳しい情報を必要なときだけ開けるようにします。

---

## デプロイ（Streamlit Community Cloud）

Streamlit が提供する **無料のホスティングサービス** です。GitHub と連携するだけでデプロイできます。

### 手順

1. アプリのコードを GitHub リポジトリに push
2. `requirements.txt` をリポジトリのルートに置く
3. [share.streamlit.io](https://share.streamlit.io) にログイン（GitHub アカウントで OK）
4. **「New app」** → リポジトリ・ブランチ・ファイル名（例：`app.py`）を指定
5. **「Deploy!」** → 数分でデプロイ完了

`https://your-app.streamlit.app` のような URL でアクセスできます。

### Docker でローカル本番確認

Docker を使うと、Streamlit アプリの実行環境をコンテナにまとめられます。ローカルで本番に近い形を確認したいときや、サーバーへ配置するときに便利です。

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8501
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

この Dockerfile は、依存パッケージを入れたあと、コンテナ起動時に Streamlit を 8501 番ポートで立ち上げます。`--server.address=0.0.0.0` は、コンテナ外からアクセスできるようにする指定です。

```bash
docker build -t streamlit-app .
docker run -p 8501:8501 streamlit-app
```

`docker run -p 8501:8501` によって、手元のブラウザからコンテナ内の Streamlit アプリへアクセスできます。

---

## FastAPI との使い分け

| | Streamlit | FastAPI |
|-|-----------|---------|
| 用途 | データ可視化・デモ・社内ツール | 本番 API・フロントエンドとの連携 |
| 難易度 | 低い（Python だけ） | 中程度（API 設計の知識が必要） |
| フロントエンドの自由度 | 限定的（Streamlit の UI に依存） | 高い（何でも作れる） |
| 向いている人 | DS エンジニア・分析者 | バックエンドエンジニア |

**判断基準：** まずは Streamlit で素早くデモを作り、本格公開が必要になったら FastAPI に移行する、という流れが効率的です。

---

## よくある疑問

**Q. Streamlit アプリはデータを保持できる？**  
A. デフォルトでは保持できません（再実行のたびにリセット）。`st.session_state` を使うと、セッション中のデータを保持できます。

Streamlit は操作のたびにスクリプトを再実行するため、普通の変数だけでは前回の状態が残りません。カウンター、ログイン状態、途中入力などを保持したいときに `st.session_state` を使います。

```python
if "count" not in st.session_state:
    st.session_state.count = 0

if st.button("カウントアップ"):
    st.session_state.count += 1

st.write(f"カウント: {st.session_state.count}")
```

`st.session_state` は、ユーザーのセッションごとに値を保存する辞書のようなものです。初回だけ `count` を 0 にし、ボタンが押されるたびに 1 増やしています。

**Q. 認証（ログイン機能）は付けられる？**  
A. Streamlit Community Cloud では GitHub アカウントによるアクセス制限ができます。カスタム認証は `streamlit-authenticator` ライブラリが使えます。

---


## 確認問題

1. Streamlit は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [pandas / scikit-learn / matplotlib](pandas-sklearn) — データの準備
- [Python × Web API（FastAPI）](FastAPI) — より本格的な API 開発
- [Docker](Docker) — コンテナ化してデプロイ

---

[← ホームへ](Home)
