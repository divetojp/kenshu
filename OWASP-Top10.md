# OWASP Top 10

Web アプリケーションで最も重大なセキュリティリスク 10 項目です。OWASP（Open Web Application Security Project）が定期的に更新し、セキュリティ診断・設計・レビューの共通言語として使われます。

---

## はじめて読む人へ

セキュリティの問題は「実装の細かいミス」だけでなく、「設計の考え方」から生まれることが多いです。このページでは各リスクの仕組みと防御策をまとめて把握し、開発中に「これは A03 のリスクに該当するか？」と考える習慣をつけることを目標にします。

### 読む前に押さえること

- [セキュリティ基礎](セキュリティ.md) の HTTPS・.env・XSS/CSRF の基本
- [セキュリティ詳解](セキュリティ詳解.md) の SQLi・認証・パスワード管理

### 読み終えたら説明できること

- OWASP Top 10 の 10 項目の名前と概要を説明できる
- 自分のコードで該当するリスクを指摘できる

---

## 2021 年版 Top 10

| 順位 | リスク名 | 一言説明 |
|------|--------|---------|
| A01 | アクセス制御の不備 | 認可チェックの漏れ・他人のデータへのアクセス |
| A02 | 暗号化の失敗 | 平文保存・弱いアルゴリズム |
| A03 | インジェクション | SQL・コマンド・テンプレートへの悪意ある入力 |
| A04 | 安全でない設計 | 脅威モデリングや安全性を考慮しないアーキテクチャ |
| A05 | セキュリティの設定ミス | デフォルトパスワード・不要機能の公開 |
| A06 | 脆弱なコンポーネント | 既知の脆弱性を持つライブラリの使用 |
| A07 | 認証・識別の失敗 | ブルートフォース・セッション固定・弱いパスワード |
| A08 | ソフトウェア・データの整合性の失敗 | CI/CDへの不正介入・署名なし更新 |
| A09 | ログ・監視の不足 | 攻撃を検知・記録できていない |
| A10 | SSRF（サーバーサイドリクエストフォージェリ） | サーバーを踏み台に内部リソースへアクセス |

---

## A01：アクセス制御の不備

最も多発するリスクです。「ログインしているかどうかのチェック」だけでなく、「このユーザーはこのリソースにアクセスしてよいか」の確認が漏れています。

**典型例：**

```python
# 脆弱：自分以外のユーザー情報が取得できる
@app.get("/users/{user_id}/profile")
def get_profile(user_id: int):
    return db.get_user(user_id)  # 誰のIDでも取れる

# 安全：自分のリソースだけに制限
@app.get("/users/{user_id}/profile")
def get_profile(user_id: int, current_user = Depends(get_current_user)):
    if current_user["id"] != user_id:
        raise HTTPException(status_code=403, detail="アクセスできません")
    return db.get_user(user_id)
```

**確認チェックリスト：**
- URL の ID を変えると他人のデータが見えないか
- 管理者専用エンドポイントにロールチェックがあるか

---

## A02：暗号化の失敗

データが平文で保存・送信されているリスクです。

**やってはいけないこと：**

```python
# NG: パスワードを平文で保存
user.password = request.password

# NG: MD5でハッシュ（高速すぎて総当たりに弱い）
import hashlib
user.password = hashlib.md5(request.password.encode()).hexdigest()

# OK: bcrypt / Argon2 を使う
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"])
user.password = pwd_context.hash(request.password)
```

**確認チェックリスト：**
- DB に保存するパスワードは bcrypt/Argon2 か
- HTTPS が強制されているか
- 機密データ（クレカ番号等）の保存方法は適切か

---

## A03：インジェクション

外部入力が「コード」として実行されるリスクです。SQL インジェクションが代表例ですが、OS コマンド・テンプレートエンジンにも同じ問題が起きます。

```python
# OS コマンドインジェクション（NG）
import subprocess
filename = request.query_params["file"]
subprocess.run(f"cat /var/log/{filename}", shell=True)  # ; rm -rf / を渡せる

# 安全（引数をリストで渡す）
subprocess.run(["cat", f"/var/log/{filename}"], shell=False)
```

**原則：入力を「コード」ではなく「データ」として扱う。**

- SQL: プレースホルダを使う
- OS コマンド: `shell=False` でリストを渡す
- テンプレート: 自動エスケープを有効にする

---

## A04：安全でない設計

コードの実装問題ではなく、設計段階でセキュリティを考慮していないリスクです。

**例：**
- パスワードリセット機能を「秘密の質問」だけで認証している
- 決済フローでサーバー側の金額検証がなく、クライアントから価格を送れる
- レート制限がなくブルートフォースし放題

**対策：**

```python
# レート制限の例（slowapi を使用）
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/login")
@limiter.limit("5/minute")  # 1分間に5回まで
def login(request: Request, data: LoginRequest):
    ...
```

---

## A05：セキュリティの設定ミス

デフォルト設定のまま本番に出すリスクです。

**よくある設定ミス：**

| 問題 | 悪い例 | 良い例 |
|------|--------|--------|
| デバッグモードON | `DEBUG=True` | 本番は `DEBUG=False` |
| デフォルトパスワード | admin/admin | ランダムな強いパスワード |
| 不要なエンドポイント公開 | `/admin` が誰でもアクセス可 | IP制限・VPN必須 |
| エラーの詳細露出 | スタックトレースをそのまま返す | 汎用エラーメッセージを返す |

```python
# FastAPI: 本番でのエラー詳細非表示
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

@app.exception_handler(Exception)
async def general_exception_handler(request, exc):
    # 詳細はログに出す、ユーザーには汎用メッセージ
    logger.error(f"Unhandled error: {exc}")
    return JSONResponse(status_code=500, content={"detail": "サーバーエラーが発生しました"})
```

---

## A06：脆弱なコンポーネント

使用しているライブラリに既知の脆弱性がある状態です。

```bash
# Python: 脆弱性チェック
pip install safety
safety check

# JavaScript
npm audit
npm audit fix

# GitHub Dependabot を有効にする（.github/dependabot.yml）
```

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "pip"
    directory: "/"
    schedule:
      interval: "weekly"
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## A07：認証・識別の失敗

ログイン機能の実装ミスです。

**よくある問題：**
- ブルートフォース攻撃（レート制限なし）
- 弱いパスワードを許容
- JWT の `alg: none` を受け入れる

```python
# JWT 検証時のアルゴリズム明示（必須）
# NG: algorithms を指定しないと "none" を受け入れる実装がある
payload = jwt.decode(token, SECRET_KEY)  # 危険

# OK: 使用するアルゴリズムを明示
payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
```

---

## A08：ソフトウェア・データの整合性の失敗

CI/CD パイプラインや依存パッケージの更新に不正が混入するリスクです。

**具体例：**
- `npm install` で悪意あるパッケージをインストール（typosquatting）
- CI/CD の secrets が漏洩し、ビルドに悪意あるコードが注入される

**対策：**
- `package-lock.json` / `poetry.lock` をコミットし、バージョンを固定する
- GitHub Actions の `GITHUB_TOKEN` の権限を最小化する
- サードパーティ Action のハッシュを固定する（`@v3` でなく `@sha256:...`）

---

## A09：ログ・監視の不足

攻撃が起きても検知・追跡できない状態です。

```python
import logging

logger = logging.getLogger(__name__)

@app.post("/login")
def login(req: LoginRequest):
    user = authenticate(req.email, req.password)
    if not user:
        # 失敗ログを必ず残す
        logger.warning(f"Login failed: email={req.email} ip={get_client_ip()}")
        raise HTTPException(status_code=401)

    logger.info(f"Login success: user_id={user.id}")
    return issue_token(user)
```

**ログに残すべき事象：**
- ログイン成功・失敗
- 権限エラー（403）
- 重要なデータ変更（削除・ロール変更）
- 異常に多いリクエスト

---

## A10：SSRF（サーバーサイドリクエストフォージェリ）

サーバーが外部の URL を取得する機能を悪用し、内部ネットワークへアクセスさせる攻撃です。

```
攻撃者 → 「この URL の内容を取得して」
          → http://169.254.169.254/latest/meta-data/  ← AWS メタデータ
サーバーが取得 → IAM 認証情報が漏洩
```

```python
from urllib.parse import urlparse

ALLOWED_HOSTS = {"api.trusted.com", "cdn.example.com"}

def fetch_url(url: str) -> bytes:
    parsed = urlparse(url)
    # 許可リストに含まれていないホストへのリクエストを拒否
    if parsed.hostname not in ALLOWED_HOSTS:
        raise ValueError(f"許可されていないホストです: {parsed.hostname}")
    # ... requests.get(url) ...
```

---

## 開発チェックリスト

プルリクエストのレビュー時に確認する観点です。

| 項目 | 確認内容 |
|------|---------|
| A01 | 認可チェックが全エンドポイントにあるか |
| A02 | パスワードは bcrypt か、機密情報は平文保存していないか |
| A03 | 入力を SQL/コマンドに直接埋め込んでいないか |
| A05 | DEBUG=False、不要なエンドポイントを閉じているか |
| A06 | `npm audit` / `safety check` をパスしているか |
| A07 | レート制限があるか、JWT アルゴリズムを明示しているか |
| A09 | 認証失敗・権限エラーをログに記録しているか |

---

## 確認問題

1. A01（アクセス制御の不備）と A07（認証の失敗）の違いを具体例を使って説明してください。
2. `http://169.254.169.254/` へのリクエストが危険な理由を説明してください。
3. 自分が開発しているアプリケーションで最もリスクが高い Top 10 項目はどれですか？理由も述べてください。

---

## 関連ページ

- [セキュリティ基礎](セキュリティ.md) — HTTPS・秘密情報管理
- [セキュリティ詳解](セキュリティ詳解.md) — SQLi・XSS・CSRF・bcrypt の実装
- [認証・認可](認証・認可.md) — JWT・OAuth2 の実装
- [ネットワーク詳解](ネットワーク詳解.md) — TLS・HTTP ヘッダーの仕組み
