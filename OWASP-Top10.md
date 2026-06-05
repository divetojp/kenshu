# OWASP Top 10

Web アプリケーションでよく起きるセキュリティの失敗を10項目にまとめたリストです。OWASP（Open Web Application Security Project）が数年おきに更新しています。「セキュリティ診断でここを必ず見る」という共通言語として使われます。

---

## はじめて読む人へ

このリストは「プロのエンジニアでもやりがちなミス」を集めたものです。全項目を覚える必要はありません。「自分が今書いているコードに当てはまるものはないか」という視点で使います。

### 読む前に押さえること

- [セキュリティ基礎](セキュリティ.md) — HTTPS・.env・XSS/CSRF の基本
- [セキュリティ詳解](セキュリティ詳解.md) — SQLi・bcrypt・認証/認可のコード

### 読み終えたら説明できること

- Top 10 の各項目が「何をされる失敗か」を1文で説明できる
- 自分のコードで該当するリスクを指摘できる

---

## 2021 年版 Top 10 早見表

| # | リスク名 | ひとことで言うと |
|---|---------|----------------|
| A01 | アクセス制御の不備 | 「他人のデータが見える・操作できる」 |
| A02 | 暗号化の失敗 | 「パスワードが平文で流れる・保存される」 |
| A03 | インジェクション | 「入力が命令として実行される」 |
| A04 | 安全でない設計 | 「そもそも設計の段階で穴がある」 |
| A05 | セキュリティの設定ミス | 「デフォルトのまま・デバッグモードON」 |
| A06 | 脆弱なコンポーネント | 「古いライブラリの既知の穴を突かれる」 |
| A07 | 認証・識別の失敗 | 「ログイン機能に穴がある」 |
| A08 | ソフトウェア整合性の失敗 | 「CI/CD やパッケージに不正が混入」 |
| A09 | ログ・監視の不足 | 「攻撃されても気づかない」 |
| A10 | SSRF | 「サーバーを踏み台に内部に侵入される」 |

---

## A01：アクセス制御の不備

**何が起きるか：** ログインしているだけで、他の人のデータにもアクセスできてしまう。

**シナリオ**

自分のプロフィールページ → https://example.com/users/42/profile
URLの42を43に変えたら → 他人のプロフィールが見えた 😱
```python
# ❌ 認証だけして認可を忘れている
@app.get("/users/{user_id}/data")
def get_data(user_id: int, current_user = Depends(get_current_user)):
    return db.get_user(user_id)  # 誰のIDでも取れる！

# ✅ 「自分のリソースか」を必ず確認する
@app.get("/users/{user_id}/data")
def get_data(user_id: int, current_user = Depends(get_current_user)):
    if current_user["id"] != user_id:
        raise HTTPException(403, "アクセスできません")
    return db.get_user(user_id)
```

**チェック：** URL の ID・番号を変えたとき、他人のデータが見えないか？

---

## A02：暗号化の失敗

**何が起きるか：** パスワードや個人情報が平文（そのままの文字）で保存・送信されており、DB が漏れたり通信が傍受されたりすると即座に読まれる。

**典型的な失敗**

DB に保存: password = "abc123"  ← 平文
DB に保存: password = "900150983cd24fb0d6963f7d28e17f72"  ← MD5（簡単に解読可）

HTTP で送信: POST /login password=abc123  ← 暗号化されていない
```python
# ✅ bcrypt で保存
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"])
hashed = pwd_context.hash("abc123")
# → "$2b$12$..." ← これだけ保存する
```

**チェック：** パスワードは bcrypt/Argon2 か？ HTTPS が強制されているか？

---

## A03：インジェクション

**何が起きるか：** ユーザーの入力が「データ」ではなく「命令」として実行される。SQL インジェクション・OS コマンドインジェクションが代表例。

```python
# SQL インジェクション
query = f"SELECT * FROM users WHERE id = {user_id}"
# → user_id に "1; DROP TABLE users; --" を入力されると users テーブルが消える

# OS コマンドインジェクション
filename = request.args["file"]
os.system(f"cat logs/{filename}")
# → filename に "../etc/passwd; rm -rf /" を入れられると被害甚大

# ✅ どちらも「命令とデータを分ける」で解決
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
subprocess.run(["cat", f"logs/{filename}"], shell=False)
```

**チェック：** ユーザー入力を文字列結合で SQL・コマンドに混ぜていないか？

---

## A04：安全でない設計

**何が起きるか：** コードの実装以前に、設計の段階でセキュリティを考慮していない。

**典型的な「安全でない設計」**

パスワードリセット:
「秘密の質問：お母さんの旧姓は？」
→ SNS で調べれば分かる情報 → なりすましできる

決済:
フロントエンドから「商品金額: 10円」を送れる
→ 攻撃者がリクエストを改ざんして1円で購入

ログイン:
失敗しても何度でも試せる
→ ブルートフォース（総当たり）でパスワードを特定される
```python
# ✅ レート制限（ブルートフォース対策）
from slowapi import Limiter

@app.post("/login")
@limiter.limit("5/minute")   # 1分間に5回まで
def login(request: Request, data: LoginRequest):
    ...
```

**チェック：** 金額・権限に関わるデータはサーバー側で検証しているか？

---

## A05：セキュリティの設定ミス

**何が起きるか：** デフォルトのパスワード・設定、デバッグモードの本番有効化、不要な機能の公開。

```python
# よくある設定ミス

# ❌ 本番でデバッグモード ON → スタックトレースがユーザーに見える
DEBUG = True  # 本番は必ず False

# ❌ エラーの詳細をそのまま返す
@app.exception_handler(Exception)
async def handler(req, exc):
    return JSONResponse(500, {"error": str(exc)})  # スタックトレース公開

# ✅ 本番用エラーハンドリング
@app.exception_handler(Exception)
async def handler(req, exc):
    logger.error(f"Error: {exc}")  # ログには残す
    return JSONResponse(500, {"error": "サーバーエラーが発生しました"})  # 詳細は隠す
```

| 設定ミス | 危険 | 対処 |
|---------|------|------|
| `DEBUG=True` で本番公開 | 内部コードが見える | `DEBUG=False` |
| デフォルトパスワード（admin/admin） | 即乗っ取り | 初回起動時に変更必須 |
| CORS で `*` を許可 | どのサイトからでも API 呼び出し可 | 許可するオリジンを限定 |
| `/admin` が認証なし | 誰でも管理画面にアクセス | IP 制限・認証を追加 |

---

## A06：脆弱なコンポーネント

**何が起きるか：** 使用しているライブラリに既知の脆弱性がある。古いバージョンのまま放置すると、その脆弱性を突く攻撃ツールが公開されているため、誰でも攻撃できる状態になる。

```bash
# Python の脆弱性チェック
pip install pip-audit
pip-audit

# JavaScript の脆弱性チェック
npm audit

# 自動チェック（GitHub Dependabot）
# .github/dependabot.yml を作るだけで週次チェック有効化
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
```

**チェック：** 定期的に `npm audit` / `pip-audit` を実行しているか？

---

## A07：認証・識別の失敗

**何が起きるか：** ログイン機能の実装に穴があり、アカウントを乗っ取られる。

```python
# よくある認証の穴

# ❌ レート制限なし → パスワードを無限に試せる（ブルートフォース）
@app.post("/login")
def login(data: LoginData):
    if db.check_password(data.email, data.password):
        return {"token": create_token(data.email)}

# ❌ JWT のアルゴリズム未指定
jwt.decode(token, SECRET_KEY)  # "alg: none" を受け入れる実装がある

# ✅ 安全な実装
@app.post("/login")
@limiter.limit("5/minute")     # レート制限
def login(data: LoginData):
    ...

jwt.decode(token, SECRET_KEY, algorithms=["HS256"])  # アルゴリズム明示
```

**チェック：** ログインにレート制限はあるか？ JWT のアルゴリズムは明示しているか？

---

## A08：ソフトウェア・データの整合性の失敗

**何が起きるか：** CI/CD パイプラインや npm パッケージに悪意あるコードが混入する。

**典型例**

「lodash」と見せかけた「1odash」（Lをイチにした偽物）をインストール
→ インストール時にマルウェアが実行される（タイポスクワッティング）

CI/CD の Secrets が GitHub Actions のサードパーティ Action に漏れる
```bash
# package-lock.json / poetry.lock をコミットしてバージョンを固定
git add package-lock.json
```

```yaml
# GitHub Actions: サードパーティ Action はハッシュで固定
- uses: actions/checkout@v4         # ❌ タグは変更可能
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # ✅ ハッシュで固定
```

---

## A09：ログ・監視の不足

**何が起きるか：** 攻撃が起きても検知できず、被害が拡大してから発覚する。

```python
# ✅ 重要な操作は必ずログに残す
import logging
logger = logging.getLogger(__name__)

@app.post("/login")
def login(req: LoginRequest, request: Request):
    user = authenticate(req.email, req.password)
    if not user:
        # 失敗ログ（何度も失敗していたら攻撃の可能性）
        logger.warning(f"Login failed | email={req.email} | ip={request.client.host}")
        raise HTTPException(401)

    logger.info(f"Login success | user_id={user.id}")
    return create_token(user)
```

**ログに残すべきもの：**

□ ログイン成功・失敗（IP アドレスも）
□ 権限エラー（403）が出た箇所
□ 重要データの変更・削除
□ 管理者操作
□ 短時間に大量のリクエスト
---

## A10：SSRF（サーバーサイドリクエストフォージェリ）

**何が起きるか：** 「この URL の内容を取ってきて」という機能を悪用し、サーバーの内側（プライベートネットワーク・クラウドのメタデータ）にアクセスさせる。

**シナリオ（画像取得機能）**

正常: https://example.com/fetch?url=https://images.cdn.com/photo.jpg
攻撃: https://example.com/fetch?url=http://169.254.169.254/latest/meta-data/
↑ AWS のインスタンスメタデータ。IAM の認証情報が丸ごと取れる
```python
from urllib.parse import urlparse

# ✅ 許可リストに含まれたホストだけにリクエストする
ALLOWED_HOSTS = {"images.example.com", "cdn.trusted.com"}

def fetch_external(url: str) -> bytes:
    parsed = urlparse(url)
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError(f"このホストへのアクセスは許可されていません: {parsed.hostname}")
    return requests.get(url, timeout=5).content
```

---

## 開発時のチェックリスト

コードレビューやリリース前に確認してください。

| 確認項目 | 対応する A # |
|---------|------------|
| URL の ID を変えると他人のデータが見えないか | A01 |
| パスワードは bcrypt/Argon2 で保存しているか | A02 |
| ユーザー入力をSQL/コマンドに直接混ぜていないか | A03 |
| ログイン試行にレート制限があるか | A04・A07 |
| 本番で DEBUG=False になっているか | A05 |
| `npm audit` / `pip-audit` をパスしているか | A06 |
| JWT のアルゴリズムを明示しているか | A07 |
| ロックファイル（package-lock.json 等）をコミットしているか | A08 |
| 認証失敗・権限エラーをログに残しているか | A09 |
| 外部 URL を取得する機能は許可リストを使っているか | A10 |

---

## 確認問題

1. A01（アクセス制御の不備）と A07（認証の失敗）の違いを「ログインの前後」という観点で説明してください。
2. A09「ログ・監視の不足」がセキュリティリスクになる理由を、「攻撃が成功した後」の視点から説明してください。
3. 自分が今作っているまたは使っているアプリケーションで、最も当てはまりそうな項目を選んで理由を説明してください。

---

## 関連ページ

- [セキュリティ基礎](セキュリティ.md) — HTTPS・.env・XSS/CSRF の入門
- [セキュリティ詳解](セキュリティ詳解.md) — 各攻撃の実装レベルの解説
- [認証・認可](認証・認可.md) — JWT・OAuth2・ロール管理の実装
- [マルウェア・サイバー攻撃](マルウェア・サイバー攻撃.md) — Web 以外の攻撃手法
- [情報セキュリティ法規と制度](情報セキュリティ法規.md) — 被害時の法的対応・相談窓口
