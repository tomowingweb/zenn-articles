---
title: "GitHub ActionsでSWELL子テーマのPR自動レビューを構築する"
emoji: "🤖"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "githubactions", "php"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-08-28-claude-code-auto-review-swell-child-theme)では、Claude CodeのStop/StartUpフックを組み合わせてSWELL子テーマのPHPコードを自動レビューする仕組みを作りました。ローカルで `review-report.md` に結果を書き出す運用は快適でしたが、副業案件でクライアントや共同作業者がいる場合、**レビュー結果がGitHub上で見えないと共有しづらい**という課題が残っていました。

今回はローカルのClaude Codeレビューに加えて、**PRを作成したときにGitHub Actionsが自動でSWELL特化のコードレビューを実行し、結果をPRコメントとして投稿する**構成を実装します。

これにより副業SWELL案件の品質管理フローが以下のようになります。

```
ローカル実装
  └── Claude Code（Stop/StartUpフック）でローカルレビュー
        └── PR作成
              └── GitHub Actionsが自動起動
                    └── claude CLI でSWELLレビューを実行
                          └── 結果をPRコメントに投稿
```

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- GitHub Actions（Ubuntu latest）
- Node.js 20.x
- `swell-hooks-mcp` v1.1.0（自作MCPサーバー）
- Anthropic API キー

## 設計方針

### GitHub Actionsでclaude CLIを動かす

Claude Code（`claude` CLI）はAPI経由で非対話的に実行できます。`--print` フラグを使うと、通常の対話セッションを開かずに1回のプロンプトに応答した出力を返します。

```bash
claude --print "レビューしてください：\n$(cat functions.php)"
```

GitHub Actionsでこれを使い、差分ファイルをプロンプトに含めてレビューを実行します。

### swell-hooks-mcpをActions上で動かす

MCPサーバーをGitHub Actionsで動かすには、npmパッケージとしてインストールできる状態にしておく必要があります。前回までの `swell-hooks-mcp` はローカルのみでしたが、今回 `npm pack` でパッケージ化してリポジトリに含めるか、GitHub Packages経由で配布する方法を取ります。

今回はシンプルに **リポジトリにMCPサーバーのソースコードを同梱し、Actions上でローカルインストールする**方法を採用しました。

## 実装内容

### 1. Anthropic APIキーをGitHub Secretsに登録

リポジトリの Settings → Secrets and variables → Actions から `ANTHROPIC_API_KEY` を登録します。

### 2. GitHub Actionsワークフローファイルの作成

`.github/workflows/swell-review.yml` を作成します。

```yaml
name: SWELL子テーマ コードレビュー

on:
  pull_request:
    paths:
      - '**.php'
      - '**.css'
      - '**.js'

jobs:
  swell-review:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Node.js セットアップ
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Claude Code CLIをインストール
        run: npm install -g @anthropic-ai/claude-code

      - name: swell-hooks-mcpをインストール
        run: |
          cd .mcp/swell-hooks-mcp
          npm install
          npm run build

      - name: 差分ファイルを取得
        id: diff
        run: |
          BASE=${{ github.event.pull_request.base.sha }}
          HEAD=${{ github.event.pull_request.head.sha }}
          CHANGED=$(git diff --name-only "$BASE" "$HEAD" | grep -E '\.(php|css|js)$' || true)
          echo "files<<EOF" >> $GITHUB_OUTPUT
          echo "$CHANGED" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT

      - name: SWELL子テーマ コードレビューを実行
        id: review
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          # MCP設定ファイルを作成（Actions環境向け）
          cat > /tmp/mcp-config.json << 'EOF'
          {
            "mcpServers": {
              "swell-hooks-mcp": {
                "command": "node",
                "args": [".mcp/swell-hooks-mcp/dist/index.js"]
              }
            }
          }
          EOF

          # 変更ファイルの内容を収集
          REVIEW_TARGET="${{ steps.diff.outputs.files }}"
          FILE_CONTENTS=""
          for f in $REVIEW_TARGET; do
            if [ -f "$f" ]; then
              FILE_CONTENTS="$FILE_CONTENTS\n### $f\n\`\`\`\n$(cat "$f")\n\`\`\`\n"
            fi
          done

          # Claude Codeでレビュー実行
          REVIEW_RESULT=$(claude \
            --print \
            --mcp-config /tmp/mcp-config.json \
            "以下のSWELL子テーマのコードをレビューしてください。

## チェック項目
1. フック互換性：search_hooks ツールで使用フック名がSWELLで有効か照合
2. カスタマイザーキー：search_customizer ツールで get_theme_mod のキー名を確認
3. WordPressセキュリティ：サニタイズ・エスケープ・nonce検証・権限チェック
4. パフォーマンス：不要なクエリ、is_admin()分岐漏れ

## 出力形式
Markdownで以下のセクションで出力してください：
- 🔴 要修正（致命的・セキュリティリスク）
- 🟡 推奨修正（品質改善・ベストプラクティス）
- 🟢 問題なし
- 💡 改善提案

## レビュー対象ファイル
$FILE_CONTENTS")

          # 出力をGITHUB_OUTPUTに書き込む
          {
            echo "result<<REVIEW_EOF"
            echo "$REVIEW_RESULT"
            echo "REVIEW_EOF"
          } >> $GITHUB_OUTPUT

      - name: レビュー結果をPRコメントに投稿
        uses: actions/github-script@v7
        with:
          script: |
            const reviewResult = `${{ steps.review.outputs.result }}`;
            const changedFiles = `${{ steps.diff.outputs.files }}`;

            const body = `## 🤖 SWELL子テーマ 自動コードレビュー

**対象ファイル:**
${changedFiles.split('\n').map(f => `- \`${f}\``).join('\n')}

---

${reviewResult}

---
*このコメントはClaude Code（Anthropic API）+ swell-hooks-mcp によって自動生成されました。*`;

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: body
            });
```

### 3. MCPサーバーをリポジトリに同梱

`.mcp/swell-hooks-mcp/` ディレクトリにMCPサーバーのソースコードを配置します。

```
.mcp/
  swell-hooks-mcp/
    package.json
    tsconfig.json
    src/
      index.ts       # MCPサーバー本体
      hooks-data.ts  # SWELLフックのデータ定義
      customizer-data.ts  # カスタマイザーキーのデータ定義
```

`package.json` に `build` スクリプトを追加しておきます。

```json
{
  "name": "swell-hooks-mcp",
  "version": "1.1.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  }
}
```

### 4. MCP設定ファイルの切り替え

ローカルでは `claude_desktop_config.json` でグローバルにMCPを管理していますが、GitHub Actions用には `/tmp/mcp-config.json` を一時生成して `--mcp-config` フラグで渡す方式にしました。

これにより、ローカルのMCP設定と衝突せず、CI/CDパイプライン専用の設定として独立させられます。

## 実際のPRコメント例

PRを作成すると、約2〜3分後に以下のようなコメントが自動投稿されます。

```markdown
## 🤖 SWELL子テーマ 自動コードレビュー

**対象ファイル:**
- `functions.php`
- `includes/custom-post-type.php`
- `assets/css/style.css`

---

## 🔴 要修正

**[functions.php:112]** `add_action('swell_before_footerr', ...)` — タイポあり
- `search_hooks` で照合：`swell_before_footerr` は存在しないフック名
- 正しくは `swell_before_footer`

## 🟡 推奨修正

**[custom-post-type.php:67]** `$_GET['tab']` をサニタイズせずに使用
- `sanitize_text_field($_GET['tab'])` を推奨

## 🟢 問題なし

- `esc_html()` / `esc_attr()` による出力エスケープ：適切 ✓
- `wp_verify_nonce()` によるnonce検証：実装済み ✓
- `get_theme_mod('header_height', 80)` のキー名：`search_customizer` で確認済み ✓

## 💡 改善提案

`custom-post-type.php` の `register_post_type` 呼び出しを `init` フック（優先度5）に移動すると
他プラグインとの競合リスクを下げられます。

---
*このコメントはClaude Code（Anthropic API）+ swell-hooks-mcp によって自動生成されました。*
```

## ポイント

### `--print` フラグで非対話的に使う

`claude --print` は対話セッションを開かずに1回のプロンプト応答を返すフラグです。CI/CDパイプラインでは対話操作ができないため、このフラグが必須になります。

出力はそのまま文字列として扱えるため、GitHub ActionsのステップアウトプットやPRコメントに流し込みやすいです。

### `--mcp-config` で環境ごとにMCP設定を切り替え

Claude Codeはデフォルトで `~/.config/claude/claude_desktop_config.json` のMCP設定を使いますが、`--mcp-config` フラグで別の設定ファイルを指定できます。

CI環境では `~/.config` に書き込んでグローバル設定するより、ワークフロー内で `/tmp` に一時ファイルを生成して渡す方がクリーンです。ジョブが終わればファイルは消えます。

### コスト管理

`claude --print` によるAPI呼び出しはトークン消費が発生します。PR 1件あたりの費用は差分の量によりますが、通常の副業案件（数ファイル・数百行の変更）で概算 **1〜5円程度**です。

コストが気になる場合は `on: pull_request` の `paths` フィルターを絞り、PHPファイルのみをトリガーにするなど調整できます。

```yaml
on:
  pull_request:
    paths:
      - '**.php'  # PHPのみ（CSSやJSは除外）
```

### ローカルレビューとの使い分け

| タイミング | ツール | 目的 |
|---|---|---|
| 実装中（セッション終了時） | Claude Code Stopフック | 即時フィードバック・修正 |
| PR作成時 | GitHub Actions | チーム共有・記録 |

ローカルのStopフック→StartUpフックレビューで **修正してからPRを作る**流れを維持することで、GitHub Actionsのレビューはあくまで最終確認・記録として機能します。先に修正してからPRを出すので、Actionsのコメントが「問題なし」になる確率が上がり、コスト効率も良くなります。

### レビュー結果をGit管理する必要がなくなった

ローカル運用では `review-report.md` をGitにコミットしていましたが、GitHub Actionsに移行することでPRコメントとして自動記録されます。リポジトリに専用ファイルを置く必要がなくなり、`.gitignore` に `review-report.md` を追加してすっきりしました。

## まとめ

今回実装した内容をまとめます。

| コンポーネント | 役割 |
|---|---|
| `.github/workflows/swell-review.yml` | PR時に自動レビューを実行 |
| `claude --print --mcp-config` | 非対話的API実行 + MCP連携 |
| `.mcp/swell-hooks-mcp/` | CI環境向けにMCPを同梱 |
| `actions/github-script` | レビュー結果をPRコメントに投稿 |

前回のローカルレビュー（Stop/StartUpフック）と今回のGitHub Actions連携で、副業SWELL案件の開発フロー全体をカバーできるようになりました。

- **ローカル**：実装直後に問題を発見・修正
- **GitHub**：PR上でレビュー結果を記録・共有

これまでの連載（SWELLフックMCP → カスタマイザーMCP → ローカル自動レビュー → 今回のCI連携）で、Claude Code + MCPによるSWELL子テーマ開発の自動化ツールチェーンが一通り形になりました。

次回は、今回のセットアップを **テンプレートリポジトリ化して副業案件のプロジェクト立ち上げを5分で終わらせる** 仕組みを作ろうと考えています。毎回ゼロから設定するのではなく、`Use this template` ボタン一発でClaude Code + MCPの環境が整った状態から開発を始められる構成です。
