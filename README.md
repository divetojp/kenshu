# OTA テックガイド

滋賀大学データサイエンス学部・近江テック・アカデミーの学生を対象にした、ソフトウェアエンジニアリング・機械学習・インフラの実践リファレンスです。

## 公開サイト

**[https://ohmitechacademy.github.io/tech-guide/](https://ohmitechacademy.github.io/tech-guide/)**

## 扱うテーマ

| セクション | 主要トピック |
|-----------|-------------|
| 基礎・ツール | プログラミング入門・ターミナル・Git/GitHub・Linux・Markdown |
| 数学・統計 | 確率統計・線形代数・微分最適化・情報理論・ベイズ理論・フーリエ解析・離散数学 |
| 機械学習・AI | 教師あり/なし/強化学習・深層学習・NLP・LLM・MLOps |
| データサイエンス | pandas・データ可視化・特徴量エンジニアリング・時系列・パイプライン |
| 次元削減・クラスタリング | PCA・因子分析・LDA・CCA・MDS・クラスター分析 |
| Web・バックエンド | Python・JavaScript・TypeScript・React・Next.js・FastAPI・データベース |
| インフラ・クラウド | Docker・Kubernetes・CI/CD・Terraform・Cloudflare・システム設計 |
| セキュリティ | OWASP Top 10・認証認可・セキュリティ詳解 |

## リポジトリ構成

```
tech-guide/
├── *.md              # コンテンツ（ルートに配置、CI でビルド時に docs/ にコピー）
├── Home.md           # トップページ（index.md にマッピング）
├── mkdocs.yml        # MkDocs Material テーマの設定
├── requirements.txt  # ビルド依存関係
├── docs/
│   ├── javascripts/  # KaTeX（数式描画）
│   └── stylesheets/  # カスタム CSS
└── .github/workflows/deploy.yml  # GitHub Pages への自動デプロイ
```

## コードサンプル

[examples/](examples/) フォルダに、解説するコードサンプルを格納しています（現在は準備中）。

## 関連リポジトリ

- [ota_hp](https://github.com/ohmitechacademy/ota_hp) — コーポレートサイト本体
- [tech-guide](https://github.com/ohmitechacademy/tech-guide) — このリポジトリ
