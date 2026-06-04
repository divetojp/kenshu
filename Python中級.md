# Python 中級

[Python基礎](Python.md) で変数・関数・クラスを学んだ次のステップです。**「動くコードを書く」から「読みやすく・安全で・再利用しやすいコードを書く」**への移行を目指します。

---

## はじめて読む人へ

「Python の基礎は学んだけど、他人のコードを読むと知らない書き方が多い」「型ヒントって何？デコレータって何？」——このページで Python の中級機能を体系的に習得します。

### 読む前に押さえること

- [Python基礎](Python.md) — 変数・関数・クラス・例外処理

### 読み終えたら説明できること

- 型ヒントを使ってバグを事前に防げる
- デコレータでコードの共通処理を切り出せる
- ジェネレータで大きなデータを効率よく処理できる
- poetry で依存関係を管理できる

---

## 型ヒント（Type Hints）

Python 3.5 以降で使える「変数や引数の型を明示する機能」です。**実行時にエラーにはなりません**が、エディタ補完・静的解析ツール・チームメンバーへの情報共有に役立ちます。

```python
# 型ヒントなし（何を渡せばいいかわからない）
def greet(name):
    return "Hello, " + name

# 型ヒントあり（str を渡して str が返ることが明確）
def greet(name: str) -> str:
    return "Hello, " + name
```

### よく使う型

```python
from typing import Optional, Union, Any
from collections.abc import Callable

# 基本型
x: int = 1
y: float = 3.14
s: str = "hello"
flag: bool = True

# コレクション
names: list[str] = ["Alice", "Bob"]
scores: dict[str, int] = {"Alice": 90, "Bob": 85}
pair: tuple[int, str] = (1, "one")
unique: set[int] = {1, 2, 3}

# Optional（None になりうる）
user_id: Optional[int] = None   # int | None と同じ

# Union（複数の型）
value: Union[int, str] = 42   # int | str と同じ（Python 3.10+）

# 関数型
callback: Callable[[int, str], bool]  # (int, str) -> bool
```

### dataclass で構造体を作る

```python
from dataclasses import dataclass, field

@dataclass
class Student:
    name: str
    score: int
    courses: list[str] = field(default_factory=list)  # ミュータブルなデフォルト値

    def is_passing(self) -> bool:
        return self.score >= 60

s = Student("田中", 75, ["数学", "英語"])
print(s)          # Student(name='田中', score=75, courses=['数学', '英語'])
print(s.is_passing())  # True
```

### mypy で静的型チェック

```bash
pip install mypy
mypy script.py  # 型エラーを事前に検出
```

```python
def add(a: int, b: int) -> int:
    return a + b

result = add("hello", 3)   # mypy が "Argument 1 has incompatible type str; expected int" と報告
```

---

## デコレータ

関数に**機能を後付けする**仕組みです。ログ・キャッシュ・認証チェックなど「どの関数にも共通して必要な処理」を重複なく追加できます。

### デコレータの基本

```python
import time

def timer(func):
    """実行時間を計測するデコレータ"""
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} の実行時間: {end - start:.4f}秒")
        return result
    return wrapper

@timer                     # timer(slow_function) と同じ
def slow_function(n: int) -> int:
    total = 0
    for i in range(n):
        total += i
    return total

slow_function(1_000_000)
# slow_function の実行時間: 0.0523秒
```

### functools.wraps で関数情報を保持する

```python
from functools import wraps

def logger(func):
    @wraps(func)   # func の __name__, __doc__ などを保持
    def wrapper(*args, **kwargs):
        print(f"[LOG] {func.__name__} 呼び出し: args={args}")
        result = func(*args, **kwargs)
        print(f"[LOG] {func.__name__} 完了: 戻り値={result}")
        return result
    return wrapper

@logger
def add(a: int, b: int) -> int:
    """2つの数を足す"""
    return a + b

add(3, 4)
print(add.__name__)  # "add"（wraps がなければ "wrapper" になる）
```

### 引数付きデコレータ（デコレータファクトリ）

```python
def retry(times: int = 3):
    """失敗したら times 回リトライするデコレータ"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"リトライ {attempt + 1}/{times}: {e}")
        return wrapper
    return decorator

@retry(times=3)
def unstable_request(url: str):
    import random
    if random.random() < 0.7:
        raise ConnectionError("接続失敗")
    return "成功"
```

---

## ジェネレータ

**必要なときだけ値を生成する**仕組みです。100万件のデータを一度にメモリに載せるのではなく、1件ずつ処理できます。

```python
# 通常のリスト（すべてメモリに格納）
def even_numbers_list(n: int) -> list[int]:
    return [i for i in range(n) if i % 2 == 0]

# ジェネレータ（1件ずつ生成）
def even_numbers_gen(n: int):
    for i in range(n):
        if i % 2 == 0:
            yield i   # ← yield で値を返す（関数は停止して待機）

import sys
n = 1_000_000
print(f"リスト: {sys.getsizeof(even_numbers_list(n)):,} bytes")
print(f"ジェネレータ: {sys.getsizeof(even_numbers_gen(n)):,} bytes")
# リスト:      4,000,056 bytes
# ジェネレータ:       112 bytes

# 使い方はイテラブルと同じ
for x in even_numbers_gen(10):
    print(x, end=' ')  # 0 2 4 6 8
```

### ジェネレータ式

```python
# リスト内包表記
squares_list = [x**2 for x in range(1_000_000)]   # 全件メモリに格納

# ジェネレータ式（() に変えるだけ）
squares_gen = (x**2 for x in range(1_000_000))    # 必要な時だけ生成

# sum, max, min などはジェネレータを直接受け取れる
total = sum(x**2 for x in range(1_000_000))
```

---

## コンテキストマネージャ（with 文）

「使い終わったら必ず後片付けする」処理を保証します。ファイル・DB 接続・ロックなどの管理に使います。

```python
# ファイルは with を使えば確実にクローズされる
with open("data.txt", "w") as f:
    f.write("hello")
# ブロックを抜けた時点で f.close() が自動で呼ばれる

# 自前のコンテキストマネージャ
from contextlib import contextmanager

@contextmanager
def timer_context(label: str):
    import time
    start = time.time()
    try:
        yield   # ← with ブロック内の処理がここで実行される
    finally:
        elapsed = time.time() - start
        print(f"{label}: {elapsed:.4f}秒")

with timer_context("データ処理"):
    data = list(range(1_000_000))
    total = sum(data)
# データ処理: 0.0312秒
```

---

## 内包表記の発展

```python
# 条件付きリスト内包
evens = [x for x in range(20) if x % 2 == 0]

# ネスト（行列の転置）
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
transposed = [[row[i] for row in matrix] for i in range(3)]

# 辞書内包
word_lengths = {word: len(word) for word in ["hello", "world", "python"]}
# {'hello': 5, 'world': 5, 'python': 6}

# 集合内包
unique_lengths = {len(word) for word in ["hello", "world", "python"]}
# {5, 6}
```

---

## パッケージ管理（poetry）

`pip` より依存関係を確実に管理できます。

```bash
# インストール
pip install poetry

# 新規プロジェクト作成
poetry new my-project
cd my-project

# パッケージを追加（pyproject.toml + poetry.lock に記録される）
poetry add requests pandas numpy

# 開発用パッケージ（本番環境不要）
poetry add --group dev pytest mypy

# 依存関係をインストール
poetry install

# スクリプト実行（仮想環境内で）
poetry run python main.py
```

### pyproject.toml の構造

```toml
[tool.poetry]
name = "my-project"
version = "0.1.0"
description = ""

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.31.0"
pandas = "^2.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4.0"
mypy = "^1.5.0"
```

`poetry.lock` ファイルをコミットすることで、チーム全員が**まったく同じバージョン**の依存関係を使えます。

---

## 例外処理の応用

```python
# カスタム例外クラス
class ValidationError(ValueError):
    def __init__(self, field: str, message: str):
        self.field = field
        super().__init__(f"{field}: {message}")

def validate_age(age: int) -> None:
    if age < 0:
        raise ValidationError("age", "0以上の値を入力してください")
    if age > 150:
        raise ValidationError("age", "150以下の値を入力してください")

try:
    validate_age(-1)
except ValidationError as e:
    print(f"バリデーションエラー - フィールド: {e.field}, 内容: {e}")
```

---

## 実践：型ヒント + デコレータを組み合わせる

```python
from functools import wraps
from typing import TypeVar, Callable
import time

F = TypeVar('F', bound=Callable)

def timed(func: F) -> F:
    """型ヒント付きのタイマーデコレータ"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__}: {elapsed*1000:.1f}ms")
        return result
    return wrapper  # type: ignore[return-value]

@timed
def process_data(data: list[int]) -> dict[str, float]:
    return {
        "sum": sum(data),
        "mean": sum(data) / len(data),
        "max": max(data),
    }

result = process_data(list(range(1_000_000)))
```

---

## 確認問題

1. `Optional[str]` と `str | None`（Python 3.10+）は同じ意味ですか？デコレータで `@wraps(func)` を付ける必要がある理由も説明してください。
2. 100万行の CSV ファイルを 1 行ずつ処理したいとき、リストと比べてジェネレータを使うメリットを説明してください。
3. `poetry add pandas` と `pip install pandas` の最大の違いを「再現性」の観点から説明してください。

---

## 関連ページ

- [Python基礎](Python.md) — 変数・型・関数・クラスの基礎
- [テスト方法論](テスト方法論.md) — pytest を使った自動テスト
- [FastAPI](FastAPI.md) — 型ヒント・デコレータを活用した API 開発
- [データパイプライン](データパイプライン.md) — ジェネレータを使った大規模データ処理
