# AGENTS Guideline

This repository is pre-alpha and under active development. The API is not stable and may change without a major version bump, so backwards compatibility is not guaranteed at this stage.
So developers of this repository DO NOT need to worry about breaking changes or maintaining backwards compatibility. We prefer to iterate quickly and make breaking changes as needed, rather than trying to maintain backwards compatibility.

## Policy

Follow the YANGI, SOLID, DRY, and KISS principles in all code and documentation. Prioritize simplicity, readability, and maintainability over cleverness or optimization. Avoid premature optimization and over-engineering. Strive for clear and concise code that is easy to understand and modify.

## Development Process

Run `mise install` first to install the toolchain and project tools.

At the end of a session, run `mise run ci` and make sure it passes. Use the narrower tasks while iterating:

```bash
mise run fmt      # Format
mise run lint     # Lint and policy checks
mise run test     # Tests
mise run ci       # Full required verification
```

## Commands

Run `mise install` first to install all tools.

```bash
mise run ci    # Run all ci:* tasks
mise run fmt   # Run all fmt:* tasks
mise run lint  # Run all lint:* tasks
mise run test  # Run all test:* tasks
```

## Tools

All tools are managed by mise. Run `mise install` to install them.

| Tool          | Purpose                             |
| ------------- | ----------------------------------- |
| uv            | Python package manager              |
| dprint        | Code formatter                      |
| prek          | Pre-commit hook runner              |
| shfmt         | Shell script formatter              |
| actionlint    | GitHub Actions linter               |
| zizmor        | GitHub Actions security linter      |
| shellcheck    | Shell script linter                 |
| ghalint       | GitHub Actions linter               |
| pinact        | Pin GitHub Actions versions to SHAs |
| go            | Go toolchain                        |
| golangci-lint | Go linter suite and formatter       |
| govulncheck   | Go vulnerability scanner            |

## Purpose

SEO 作業用の Claude Code プラグイン（agents / skills / MCP）。Google Analytics 4・
Search Console・Google Ads のデータを 1 つのプラグインから扱えるようにする。
公式 MCP があるものは公式を使い、ないものだけを Go の single binary MCP サーバとして自作する。
リポジトリ自体を marketplace として公開し、`/plugin marketplace add illumination-k/seo-agents` で入れられるようにする。

### 非目標

- 書き込み系の操作（サイトマップ送信・広告の入稿や入札変更など）はしない。すべて read-only
- 公式 MCP がある機能を自作サーバで再実装しない
- Semrush / Ahrefs などサードパーティ SEO ツールの連携は初期スコープ外

## Architecture

### MCP サーバの分担

| API                        | サーバ                                      | 備考                                                              |
| -------------------------- | ------------------------------------------- | ----------------------------------------------------------------- |
| GA4 (Admin / Data API)     | 公式 `googleanalytics/google-analytics-mcp` | `pipx run analytics-mcp` で起動                                   |
| Google Ads (GAQL)          | 公式 `googleads/google-ads-mcp`             | `search` / `list_accessible_customers` など                       |
| Search Console             | 自作 `cmd/seo-mcp`                          | 公式なし                                                          |
| Google Ads Keyword Planner | 自作 `cmd/seo-mcp`                          | 公式 Ads MCP は GAQL のみで `GenerateKeywordIdeas` を扱えないため |

### ディレクトリ構成（目標）

```
seo-agents/
├─ .claude-plugin/
│  ├─ plugin.json          # プラグインマニフェスト
│  └─ marketplace.json     # このリポジトリを marketplace として公開
├─ .mcp.json               # 公式 GA / Ads MCP と自作 seo-mcp の起動定義
├─ agents/                 # seo-analyst, keyword-researcher, technical-seo-auditor など
├─ skills/                 # レポート手順・GAQL / GSC クエリ例などの知識
├─ bin/seo-mcp             # 初回起動時に GitHub Releases から OS/arch 別バイナリを取得して exec するラッパー
├─ cmd/seo-mcp/            # 自作 MCP サーバ（stdio）
└─ internal/
   ├─ gsc/                 # Search Console クライアント
   ├─ keywordplanner/      # Ads REST API で GenerateKeywordIdeas を叩く
   └─ tools/               # MCP tool 定義
```

### 自作 MCP サーバ（seo-mcp）

- `github.com/modelcontextprotocol/go-sdk` を使い stdio で動かす
- Search Console は `google.golang.org/api/searchconsole/v1` を使う
  - tools: `list_sites`, `query_search_analytics`（dimensions / filters / dataState 対応、rowLimit のページングはサーバ側で吸収）,
    `inspect_url`, `list_sitemaps`
- Keyword Planner は公式 Go クライアントがないので REST（`customers/{id}:generateKeywordIdeas`）を直接叩く
  - tools: `generate_keyword_ideas`（seed keywords / URL、地域・言語指定）, `get_keyword_historical_metrics`
  - Ads API のバージョンは 1 箇所の定数で固定し、Dependabot 対象外なので定期的に手動で上げる
- 結果は LLM が読みやすいよう集約・整形して返す。巨大な生レスポンスはそのまま返さない

### 認証

- すべて ADC（Application Default Credentials）に揃え、公式 MCP と同じ手順で動くようにする:
  `gcloud auth application-default login --scopes=https://www.googleapis.com/auth/analytics.readonly,https://www.googleapis.com/auth/webmasters.readonly,https://www.googleapis.com/auth/adwords,https://www.googleapis.com/auth/cloud-platform`
- gcloud CLI と ADC の認証情報は `CLOUDSDK_CONFIG`（`.mise.toml` の `[env]`）でリポジトリ直下の `.gcloud/` に固定する。`.gcloud/` は gitignore し、絶対に commit しない
- Ads は追加で `GOOGLE_ADS_DEVELOPER_TOKEN` と（MCC 経由なら）`GOOGLE_ADS_LOGIN_CUSTOMER_ID` を環境変数で受け取る
- Search Console のスコープは `webmasters.readonly` のみ。write スコープは要求しない
- トークンや認証情報をログ・tool の出力に含めない

### 配布

- goreleaser で darwin/linux/windows × amd64/arm64 のバイナリを GitHub Releases に出す
- `.mcp.json` からは `${CLAUDE_PLUGIN_ROOT}/bin/seo-mcp` を起動する。ラッパーはプラグインのバージョンに対応するリリースを取得し、
  チェックサムを検証してキャッシュする
- 開発時は `go run ./cmd/seo-mcp` で直接起動できるようにする

## Notes

- golang テンプレート由来の初期配置は目標構成に合わせて移す（`cmd/seo-mcp/` と `internal/` に整理）
- Keyword Planner は Ads の developer token が Basic access 以上でないと実データが返らない点を README に明記する
- 未決事項: agents の具体的な分担と数、GA4 / GSC の突き合わせ（ランディングページ単位の結合）を skill で持つか tool で持つか、
  PageSpeed Insights / CrUX を tool に追加するか
