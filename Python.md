# Python 基礎

> プログラミングが初めての方でも読めるように書かれた入門ページです。データサイエンス・Web API・自動化など幅広い用途で使われる Python を、動かしながら学びます。

---

## はじめて読む人へ

Python は、データサイエンスでもWeb開発でも使われる読みやすいプログラミング言語です。このページでは、まず文法を暗記するより「値を作る」「条件で分ける」「繰り返す」「処理を関数にまとめる」という流れを意識してください。


### 読む前に押さえること

- 変数は、値に名前を付けて再利用するためのものです。
- リストや辞書は、複数の値を整理するための入れ物です。
- 関数は、同じ処理を名前付きで再利用するための仕組みです。

### 読み終えたら説明できること

- 基本的なデータ型と制御構文を説明できる。
- リスト・辞書・関数を使った短い処理を書ける。
- エラーが出たときに、どの行で何が起きたか読める。

---

## Python とは

**汎用プログラミング言語** です。読みやすい文法と豊富なライブラリが特徴で、データサイエンス・Web 開発・自動化・機械学習など幅広い分野で使われています。

最初のコードでは、プログラムが「命令を上から順に実行する」ことを確認します。`print` は、値を画面に表示するための最も基本的な関数です。

```python
# これが Python のコードです
print("Hello, World!")
```

この 1 行は、文字列 `"Hello, World!"` を `print` 関数に渡しています。Python では、文字列をダブルクォートまたはシングルクォートで囲みます。

---

## まず動かしてみる

**Python インタープリタ（REPL）をターミナルで起動：**

REPL は、1 行入力するとすぐに結果が返ってくる対話環境です。小さな計算や文法の確認をするときに便利です。

```bash
python3
```

起動すると `>>>` というプロンプトが表示されます。これは「Python の命令を入力できます」という合図です。

!!! info ""
    >>> print("Hello, World!")
    Hello, World!
    >>> 1 + 2
    3
    >>> "Python" + " is fun"
    'Python is fun'
    >>> exit()   # 終了

REPL では、式だけを入力しても結果が表示されます。`1 + 2` は数値の足し算、`"Python" + " is fun"` は文字列の連結です。プログラムでは、同じ考え方をファイルに書いてまとめて実行します。

**スクリプトファイルを実行：**

まとまった処理は `.py` ファイルに保存して実行します。REPL は実験、スクリプトは再利用するプログラム、と考えると分かりやすいです。

```bash
# hello.py を作って
python3 hello.py
```

> コメントは `#` から行末まで。プログラムの実行には影響しません。

---

## 変数とデータ型

**変数** は値に名前をつけて保存する箱です。Python では型宣言が不要で、代入するだけで使えます。

変数名は、後で値を取り出すためのラベルです。Python では `変数名 = 値` の形で代入します。右側の値が先に作られ、その値に左側の名前が付く、と読むと自然です。

```python
name = "田中太郎"     # 文字列（str）
age = 21              # 整数（int）
height = 170.5        # 小数（float）
is_student = True     # 真偽値（bool）
nothing = None        # 値なし（None）
```

この例では、`name` には文字列、`age` には整数、`is_student` には真偽値が入っています。Python は値を見て型を判断するため、C のように `int age` と書く必要はありません。

**型を確認する：**

`type()` を使うと、値がどの型として扱われているかを確認できます。エラーの原因を探すときにも役立ちます。

```python
print(type(name))     # <class 'str'>
print(type(age))      # <class 'int'>
print(type(is_student))  # <class 'bool'>
```

**型を変換する：**

入力や CSV から読み込んだ値は、数字に見えても文字列として扱われることがあります。計算したいときは、必要に応じて型を変換します。

```python
int("42")       # "42" → 42
float("3.14")   # "3.14" → 3.14
str(100)        # 100 → "100"
bool(0)         # 0 → False（0・空文字・None は False、それ以外は True）
```

型変換に失敗するとエラーになります。たとえば `int("abc")` は整数にできないため `ValueError` になります。

---

## 文字列

文字列は、名前、文章、ファイルパス、URL など、テキストデータを扱うための型です。Python では文字列を足して結合したり、長さを調べたり、一部だけを取り出したりできます。

```python
greeting = "こんにちは"
name = "太郎"

# 結合
message = greeting + "、" + name + "さん！"
print(message)   # こんにちは、太郎さん！

# 繰り返し
print("=" * 30)  # ============================

# 長さ
print(len(greeting))  # 5

# スライス
s = "Hello, World!"
print(s[0])      # H（先頭）
print(s[-1])     # !（末尾）
print(s[0:5])    # Hello（0番目〜4番目）
print(s[7:])     # World!（7番目以降）
```

文字列の添字も 0 から始まります。`s[0:5]` は 0 番目から 5 番目の手前までを取り出すので、結果は `"Hello"` になります。この「終わりは含まない」というルールは、リストのスライスでも同じです。

**f-string（文字列に変数を埋め込む）：**

f-string は、変数の値を文章の中に自然に埋め込む書き方です。レポート文、ログ、画面表示を作るときによく使います。

```python
name = "田中"
age = 21
print(f"名前: {name}、年齢: {age}歳")
# → 名前: 田中、年齢: 21歳

pi = 3.14159
print(f"π ≈ {pi:.2f}")  # 小数点2桁まで
# → π ≈ 3.14
```

`{name}` の部分が変数の値に置き換わります。`{pi:.2f}` は、小数点以下 2 桁で表示する指定です。

**よく使う文字列メソッド：**

メソッドは、値に対して呼び出す操作です。文字列の前後の空白を削る、置換する、分割する、といった処理はデータ前処理でも頻繁に使います。

```python
s = "  Hello, World!  "

s.strip()         # "Hello, World!"（前後の空白を除去）
s.lower()         # "  hello, world!  "（小文字に）
s.upper()         # "  HELLO, WORLD!  "（大文字に）
s.replace("Hello", "Hi")  # "  Hi, World!  "（置換）
s.split(", ")     # ["  Hello", "World!  "]（分割）
"Python".startswith("Py")  # True
"Python".endswith("on")    # True
```

`strip()` は入力データの余計な空白を取り除くとき、`split()` は CSV 風の文字列やログを分割するときに使います。文字列処理は、データ分析の前処理の入口です。

---

## 演算子

演算子は、値同士を計算したり比較したりする記号です。計算結果だけでなく、比較演算子が `True` / `False` を返すことを押さえると、条件分岐につながります。

```python
# 算術演算子
print(10 + 3)   # 13
print(10 - 3)   # 7
print(10 * 3)   # 30
print(10 / 3)   # 3.333...（小数）
print(10 // 3)  # 3     （整数除算・切り捨て）
print(10 % 3)   # 1     （余り）
print(2 ** 10)  # 1024  （べき乗）

# 比較演算子（結果は True か False）
print(5 > 3)   # True
print(5 == 5)  # True （等しい）
print(5 != 3)  # True （等しくない）

# 論理演算子
print(True and False)   # False
print(True or False)    # True
print(not True)         # False
```

`/` は小数の割り算、`//` は切り捨ての整数除算、`%` は余りです。偶数判定では `n % 2 == 0` のように、余りを使うことがよくあります。

---

## 条件分岐（if / elif / else）

条件分岐は、「ある条件を満たすときだけ処理する」ための仕組みです。プログラムに判断をさせる最初の構文として重要です。

```python
score = 75

if score >= 80:
    print("優")
elif score >= 60:
    print("良")   # ← これが実行される
elif score >= 40:
    print("可")
else:
    print("不可")
```

この例では、条件を上から順番に調べ、最初に真になったブロックだけを実行します。`score` が 75 なので、`score >= 60` が真になり、「良」が表示されます。

**インデント（字下げ）が重要：**

Python はブロックをインデント（スペース4つ）で区別します。C 言語や JavaScript の `{}` に相当します。

次の例では、インデントされている 2 行だけが `if` の中です。最後の行はインデントされていないので、条件に関係なく実行されます。

```python
if True:
    print("この行はブロックの中")
    print("この行も")
print("この行はブロックの外")
```

**1行の条件式（三項演算子）：**

短い条件分岐は 1 行で書けます。読みやすい場合だけ使い、複雑な条件では通常の `if` 文に戻すとよいです。

```python
status = "成人" if age >= 18 else "未成年"
```

---

## 繰り返し

### for ループ

`for` ループは、リストや範囲の要素を 1 つずつ取り出して処理します。「データの各行に同じ処理をする」というデータサイエンスの基本にもつながります。

```python
fruits = ["りんご", "バナナ", "みかん"]

for fruit in fruits:
    print(fruit)

# range() で数値のループ
for i in range(5):
    print(i)   # 0, 1, 2, 3, 4

for i in range(1, 11):
    print(i)   # 1, 2, 3, ..., 10

for i in range(0, 10, 2):
    print(i)   # 0, 2, 4, 6, 8（ステップ2）

# enumerate() でインデックスと値を同時に取れる
for i, fruit in enumerate(fruits):
    print(f"{i}: {fruit}")
# 0: りんご
# 1: バナナ
# 2: みかん
```

`range(5)` は 0 から 4 までの整数を作ります。`enumerate()` を使うと、要素そのものと添字を同時に取り出せます。

### while ループ

`while` ループは、条件が真である間だけ繰り返します。回数がはっきりしていない処理に使えますが、条件がいつか偽になるように書かないと無限ループになります。

```python
count = 0
while count < 5:
    print(count)
    count += 1   # count = count + 1 と同じ

# break と continue
for i in range(10):
    if i == 3:
        continue   # i=3 をスキップ
    if i == 7:
        break      # i=7 でループを終了
    print(i)
```

`break` はループを終了し、`continue` はその回の残りを飛ばして次の繰り返しへ進みます。データを順番に見て、条件に合うものだけ処理するときによく使います。

---

## リスト（List）

順番を持つデータの集まりです。

リストは、複数の値を 1 つの変数で扱うための基本的な入れ物です。点数一覧、ファイル名一覧、ユーザー一覧のように、順番を持つデータに使います。

```python
scores = [85, 72, 91, 68, 79]

# アクセス
print(scores[0])   # 85（先頭）
print(scores[-1])  # 79（末尾）

# 変更
scores[0] = 90

# 追加・削除
scores.append(88)         # 末尾に追加
scores.insert(0, 100)     # 先頭に挿入
scores.remove(72)         # 72 を削除（最初の1つ）
popped = scores.pop()     # 末尾を取り出して削除

# 長さ・集計
len(scores)
sum(scores)
min(scores)
max(scores)

# 並べ替え
sorted(scores)              # 元のリストは変えない（新しいリストを返す）
sorted(scores, reverse=True)
scores.sort()               # 元のリストを変える

# スライス
scores[1:3]    # インデックス1〜2の要素
scores[:3]     # 先頭から3つ
scores[2:]     # 3番目以降

# 要素の確認
print(85 in scores)   # True
```

リストも添字は 0 から始まります。`append` は末尾に追加、`pop` は要素を取り出して削除します。`sorted(scores)` は新しいリストを返し、`scores.sort()` は元のリスト自体を変更する、という違いも重要です。

**リスト内包表記（短く書けて、関数型スタイル）：**

リスト内包表記は、「各要素に同じ変換をする」「条件に合う要素だけを取り出す」処理を短く書く構文です。データ前処理で非常によく使います。

```python
numbers = [1, 2, 3, 4, 5]

doubled = [n * 2 for n in numbers]         # [2, 4, 6, 8, 10]
evens   = [n for n in numbers if n % 2 == 0]  # [2, 4]
squared_evens = [n**2 for n in numbers if n % 2 == 0]  # [4, 16]
```

`[n * 2 for n in numbers]` は、「numbers の各 n について、n * 2 を新しいリストに入れる」と読みます。後ろに `if` を付けると、条件を満たす要素だけを残せます。

---

## タプル（Tuple）

変更できないリストです。座標・RGB 値など「変えてはいけないデータ」に使います。

タプルは、作った後に中身を変えられないデータ構造です。関数から複数の値を返すときや、座標のようにひとまとまりで扱いたい値に使います。

```python
point = (3, 4)
rgb = (255, 128, 0)

# アクセスはリストと同じ
print(point[0])  # 3

# 変更しようとするとエラー
# point[0] = 5  # TypeError

# アンパック（一括代入）
x, y = point
print(x, y)  # 3 4
```

アンパックは、タプルの各要素を複数の変数に一度に代入する書き方です。`x, y = point` によって、`x` に 3、`y` に 4 が入ります。

---

## 辞書（Dictionary）

キーと値のペアでデータを管理します。

辞書は、名前付きの情報をまとめるためのデータ構造です。リストが「何番目か」で値を取り出すのに対して、辞書は「どのキーか」で値を取り出します。

```python
person = {
    "name": "田中太郎",
    "age": 21,
    "scores": [85, 72, 91],
}

# 値の取得
print(person["name"])              # 田中太郎
print(person.get("email", "未設定"))  # "未設定"（キーがなければデフォルト値）

# 値の変更・追加
person["age"] = 22
person["email"] = "tanaka@example.com"

# キーの確認
print("name" in person)  # True

# 全要素を繰り返す
for key, value in person.items():
    print(f"{key}: {value}")

# キー一覧・値一覧
list(person.keys())    # ["name", "age", "scores", "email"]
list(person.values())  # ["田中太郎", 22, [...], "tanaka@example.com"]
```

`person["name"]` はキーが存在しないとエラーになります。一方、`person.get("email", "未設定")` はキーがなければデフォルト値を返すため、データに欠けがある場面で便利です。

**辞書内包表記：**

辞書内包表記は、リストなどからキーと値の対応表を作る書き方です。ID からデータを引けるようにする、名前からスコアを引けるようにする、といった用途で使います。

```python
students = [
    {"name": "田中", "score": 85},
    {"name": "山田", "score": 72},
]

# 名前 → スコア のマッピングを作る
score_map = {s["name"]: s["score"] for s in students}
# {"田中": 85, "山田": 72}
```

---

## セット（Set）

重複なし・順序なしの集まりです。

セットは「含まれているかどうか」を高速に調べたいときや、重複を取り除きたいときに使います。順番は保証されないため、並び順が大事なデータには向きません。

```python
tags = {"Python", "Web", "AI", "Python"}  # 重複は自動で消える
print(tags)   # {'Python', 'Web', 'AI'}

tags.add("Data")
tags.remove("Web")

# 集合演算
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

print(a | b)  # 和集合 {1,2,3,4,5,6}
print(a & b)  # 積集合 {3,4}
print(a - b)  # 差集合 {1,2}

# よく使う用途：重複を除去する
nums = [1, 2, 2, 3, 3, 3]
unique = list(set(nums))  # [1, 2, 3]
```

和集合 `|`、積集合 `&`、差集合 `-` は、数学の集合と同じ考え方です。たとえば「両方のリストに含まれるユーザー」を探すときは積集合が使えます。

---

## 関数

関数は、処理のまとまりに名前を付ける仕組みです。同じ処理を何度も書く代わりに、関数として定義しておくと、必要な場所で呼び出せます。

良い関数は、入力と出力がはっきりしています。引数で何を受け取り、戻り値で何を返すのかを意識すると、あとからテストしやすく、他の人にも使いやすいコードになります。

**関数** とは、まとまった処理に名前をつけて繰り返し使えるようにしたものです。

次の例では、名前を受け取って挨拶文を返す関数を定義しています。`def` で関数を作り、`return` で結果を返します。

```python
def greet(name):
    return f"こんにちは、{name}さん！"

print(greet("田中"))  # こんにちは、田中さん！
print(greet("山田"))  # こんにちは、山田さん！
```

`greet("田中")` の `"田中"` は引数です。関数の中では、その値が `name` という変数として使われます。

**デフォルト引数：**

引数に既定値を設定しておくと、呼び出し時に省略できます。よく使う値をデフォルトにしておくと、関数を短く呼び出せます。

```python
def power(base, exp=2):
    return base ** exp

print(power(3))     # 9（exp=2 がデフォルト）
print(power(3, 3))  # 27
```

**複数の戻り値：**

Python では、複数の値をタプルとしてまとめて返せます。受け取る側ではアンパックして別々の変数に入れられます。

```python
def min_max(numbers):
    return min(numbers), max(numbers)   # タプルで返す

low, high = min_max([3, 1, 4, 1, 5, 9])
print(low, high)   # 1 9
```

**型ヒント（推奨）：**

型ヒントは、引数や戻り値がどの型を想定しているかを示す注釈です。Python の実行自体を厳密に制限するものではありませんが、読み手やエディタがコードを理解しやすくなります。

```python
def calculate_bmi(weight: float, height: float) -> float:
    return weight / (height / 100) ** 2

bmi = calculate_bmi(70, 175)
print(f"BMI: {bmi:.1f}")   # BMI: 22.9
```

**lambda（無名関数・簡単な関数を1行で）：**

`lambda` は、短い関数をその場で作る書き方です。特に `sorted` の並べ替え基準や、`map` / `filter` のような処理に渡す関数として登場します。

```python
square = lambda x: x ** 2
print(square(5))  # 25

# sorted() の key によく使う
students = [{"name": "田中", "score": 85}, {"name": "山田", "score": 72}]
sorted_students = sorted(students, key=lambda s: s["score"], reverse=True)
```

ただし、複雑な処理を `lambda` で無理に書くと読みにくくなります。処理が長い場合は、普通に `def` で名前付き関数にします。

---

## エラーハンドリング（try / except）

エラーが起きても、プログラムが止まらないように処理します。

エラーハンドリングは、「失敗する可能性がある処理」を安全に扱うための仕組みです。ユーザー入力、ファイル読み込み、API 通信など、外部に依存する処理ではエラーが起きる前提で設計します。

```python
def divide(a, b):
    try:
        result = a / b
        return result
    except ZeroDivisionError:
        print("ゼロで割ることはできません")
        return None
    except TypeError as e:
        print(f"型エラー: {e}")
        return None
    finally:
        print("処理完了")  # 成功・失敗に関わらず実行

print(divide(10, 2))    # 5.0
print(divide(10, 0))    # ゼロで割ることはできません → None
```

`try` の中に通常の処理を書き、エラーが起きたら対応する `except` に移ります。`finally` は成功しても失敗しても実行されるため、後片付けに使われます。

**よくある例外：**

| 例外クラス | 発生場面 |
|-----------|---------|
| `ValueError` | 不正な値（`int("abc")` など） |
| `TypeError` | 不正な型（`"3" + 3` など） |
| `KeyError` | 辞書に存在しないキー |
| `IndexError` | リストの範囲外アクセス |
| `FileNotFoundError` | ファイルが見つからない |
| `ZeroDivisionError` | ゼロで割り算 |

---

## クラス

**クラス** は、データ（属性）と操作（メソッド）をまとめた設計図です。

クラスを使うと、関連するデータと処理を 1 つのまとまりとして扱えます。学生の名前、ID、点数リストと、その平均点を計算する処理をまとめると、プログラムの構造が現実の対象に近くなります。

```python
class Student:
    # コンストラクタ（インスタンス生成時に呼ばれる）
    def __init__(self, name: str, student_id: str):
        self.name = name
        self.student_id = student_id
        self.scores = []

    def add_score(self, score: float):
        self.scores.append(score)

    def average(self) -> float:
        if not self.scores:
            return 0.0
        return sum(self.scores) / len(self.scores)

    def __str__(self):
        return f"Student({self.name}, ID={self.student_id})"

# インスタンスを作る
s = Student("田中太郎", "DS2024001")
s.add_score(85)
s.add_score(92)

print(s)              # Student(田中太郎, ID=DS2024001)
print(s.average())    # 88.5
print(s.name)         # 田中太郎
```

`Student("田中太郎", "DS2024001")` によって、`Student` クラスから具体的な学生オブジェクトが作られます。`self` は「そのオブジェクト自身」を表し、`self.scores` のように書くことで各学生ごとのデータを持てます。

> クラスの詳細は [オブジェクト指向](オブジェクト指向) ページで学びます。

---

## ファイル操作

ファイル操作は、プログラムの外にあるデータを読み書きするための基本です。分析では CSV や JSON、ログファイルを読むことが多いため、ファイルを安全に開いて閉じる書き方を覚えておきます。

```python
# テキストファイルを書く
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("こんにちは\n")
    f.write("Python\n")

# テキストファイルを読む
with open("output.txt", "r", encoding="utf-8") as f:
    content = f.read()           # 全体を文字列として読む
    # lines = f.readlines()     # 1行ずつのリストとして読む

# with ブロックを抜けると自動でファイルが閉じられる

# pathlib を使う（より直感的）
from pathlib import Path

Path("output.txt").write_text("Hello\n", encoding="utf-8")
text = Path("output.txt").read_text(encoding="utf-8")
```

`with open(...) as f:` を使うと、ブロックを抜けたときに自動でファイルが閉じられます。`encoding="utf-8"` は文字コードの指定で、日本語を含むファイルを扱うときに重要です。

---

## モジュールとパッケージ

モジュールは、他のファイルに用意された関数やクラスを読み込む仕組みです。Python は標準ライブラリが豊富なので、日付、乱数、ファイルパス、数学関数などを自分で一から作る必要はありません。

```python
# 標準ライブラリ
import os
import sys
import math
import random
from datetime import datetime, timedelta
from collections import Counter, defaultdict

# 使い方
print(math.sqrt(16))       # 4.0
print(math.pi)              # 3.14159...
print(random.randint(1, 6)) # サイコロ（1〜6）

today = datetime.now()
print(today.strftime("%Y-%m-%d"))  # 2024-01-15

# 外部ライブラリ（pip でインストールが必要）
# import pandas as pd
# import numpy as np
```

`import math` のように読み込むと、`math.sqrt()` のようにモジュール名を付けて使います。`from datetime import datetime` のように書くと、必要な名前だけを直接使えます。

---

## 仮想環境（venv）

プロジェクトごとにパッケージのバージョンを分離します。

仮想環境は、プロジェクトごとの Python 環境を分けるための仕組みです。あるプロジェクトでは pandas 2 系、別のプロジェクトでは古いバージョンが必要、という状況でも衝突を避けられます。

```bash
# 仮想環境を作成
python3 -m venv .venv

# 有効化
source .venv/bin/activate      # Mac / Linux
.venv\Scripts\activate         # Windows

# プロンプトに (.venv) が表示される
(.venv) $ pip install pandas fastapi

# 現在の環境を記録
pip freeze > requirements.txt

# 別環境で同じ環境を再現
pip install -r requirements.txt

# 終了
deactivate
```

`.venv` を有効化している間に入れたパッケージは、そのプロジェクト用の環境に入ります。`requirements.txt` は、別の人や別の PC で同じ環境を再現するためのパッケージ一覧です。

> `.venv/` フォルダは `.gitignore` に追加し、`requirements.txt` だけを Git で管理します。

---

## よく使うパターン集

### CSV / JSON

CSV と JSON は、外部データを扱うときによく出てくる形式です。CSV は表形式、JSON は辞書やリストを組み合わせた階層構造のデータに向いています。

```python
import csv, json
from pathlib import Path

# CSV 読み込み
with open("data.csv", encoding="utf-8") as f:
    records = list(csv.DictReader(f))
# → [{"name": "田中", "score": "85"}, ...]

# JSON 読み書き
data = json.loads(Path("data.json").read_text(encoding="utf-8"))
Path("out.json").write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")
```

`csv.DictReader` は、CSV の各行を辞書として読み込みます。`json.loads` は JSON 文字列を Python の辞書やリストへ変換し、`json.dumps` は Python のデータを JSON 文字列へ戻します。

### 環境変数（秘密情報の管理）

API キーやデータベース接続文字列のような秘密情報は、コードに直接書かず、環境変数から読み込みます。コードに書いて GitHub に公開すると、第三者に悪用される危険があります。

```python
from dotenv import load_dotenv
import os

load_dotenv()  # .env ファイルを読み込む

API_KEY = os.getenv("API_KEY", "default-key")
DATABASE_URL = os.environ["DATABASE_URL"]   # なければ KeyError
```

`os.getenv()` は、環境変数がなければデフォルト値を返せます。`os.environ["DATABASE_URL"]` は必須の設定として扱い、存在しなければエラーにします。

### ロギング

ロギングは、プログラムの実行状況を記録する仕組みです。`print` でも確認はできますが、ログレベルや時刻を付けることで、あとから問題を追跡しやすくなります。

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)

logger.info("処理を開始します")
logger.warning("データが空です")
logger.error("接続に失敗しました: %s", err)
```

`info` は通常の進行状況、`warning` は注意が必要な状態、`error` は処理に失敗した状態を表します。Web API やバッチ処理では、ログがデバッグの重要な手がかりになります。

### 型定義（dataclass）

`dataclass` は、データをまとめるだけのクラスを短く書くための仕組みです。辞書でも表せますが、フィールド名や型が明示されるため、読みやすく保守しやすくなります。

```python
from dataclasses import dataclass, field

@dataclass
class NewsItem:
    title: str
    date: str
    category: str
    tags: list[str] = field(default_factory=list)

item = NewsItem(title="入試説明会", date="2024-02-01", category="イベント")
print(item.title)  # 入試説明会
```

`field(default_factory=list)` は、空のリストを安全に初期値として使うための書き方です。複数のインスタンスで同じリストを共有してしまう事故を防げます。

---

## よくある疑問

**Q. `python` と `python3` どちらを使う？**  
A. Mac/Linux では `python3` を使ってください。`python` は古い Python 2 を指すことがあります。

**Q. インデントはスペース何個？**  
A. 4 つが標準（PEP 8）。タブとスペースを混在させるとエラーになります。

**Q. `None` と `False` と `0` はどう違う？**  
A. 型が違います（`NoneType`, `bool`, `int`）。意味的には「値なし」「偽」「ゼロ」です。`if` の条件式では 3 つとも偽として扱われます。

**Q. リストとタプルはどう使い分ける？**  
A. 変更する可能性があるならリスト、変更しないならタプル。関数の戻り値など「複数値を返すだけ」のときにタプルがよく使われます。

---


## 確認問題

1. Python 基礎 は、何の問題を解決するための考え方・道具ですか。
2. このページで出てきた重要語を 3 つ選び、それぞれ 1 文で説明してください。
3. コード例やコマンド例がある場合、入力・処理・出力を分けて説明してください。
4. このページの内容が、前後の STEP や自分の作りたいものにどうつながるか説明してください。

---

## 関連ページ

- [pandas / scikit-learn / matplotlib](pandas-sklearn) — データサイエンスライブラリ
- [FastAPI](FastAPI) — Python で Web API を作る
- [Streamlit](Streamlit) — Python だけで Web アプリを作る
- [オブジェクト指向](オブジェクト指向) — クラスの詳細
- [関数型プログラミング](関数型プログラミング) — lambda・map・filter の詳細
- [テスト方法論](テスト方法論) — Python のテストの書き方（pytest）
- [正規表現](正規表現) — re モジュールの使い方

---

[← ホームへ](Home)
