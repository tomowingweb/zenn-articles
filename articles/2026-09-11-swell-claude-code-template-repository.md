---
title: "SWELL子テーマ開発テンプレートリポジトリでプロジェクト立ち上げを5分で終わらせる"
emoji: "🚀"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "github", "githubactions"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-09-04-github-actions-claude-code-review-swell)では、GitHub ActionsでSWELL子テーマのPR自動レビューを構築しました。これで副業SWELL案件の品質管理フローが一通り整いました。

ただし、新しい案件ごとに毎回同じ設定をするのは手間です。具体的には：

- `swell-hooks-mcp` のインストールと `claude_desktop_config.json` への登録
- `.claude/commands/review-swell.md` の作成
- `.claude/hooks/` 配下のスクリプト群
- `.github/workflows/swell-review.yml` の作成
- Anthropic API キーの GitHub Secrets 登録案内

これらを毎回ゼロからやると30分以上かかります。

今回は、これまで構築してきたすべての設定を **GitHubテンプレートリポジトリにまとめ**、`Use this template` ボタン一発で Claude Code + MCP + GitHub Actions が整った状態から開発を始められる仕組みを作ります。目標は **新規案件の環境構築を5分以内**に収めることです。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- Node.js 20.x
- `swell-hooks-mcp` v1.1.0（自作MCPサーバー）
- GitHub（テンプレートリポジトリ機能）

## テンプレートリポジトリの設計

### 含めるもの

これまでの連載で作ってきた設定・ツールをすべて同梱します。

```
swell-child-theme-template/           # テンプレートリポジトリのルート
├── .claude/
│   ├── commands/
│   │   └── review-swell.md           # /review-swell カスタムコマンド
│   ├── hooks/
│   │   ├── php-review.sh             # Stopフック用スクリプト
│   │   └── check-pending-review.sh   # StartUpフック用スクリプト
│   └── settings.json                 # Stop/StartUpフック設定
├── .github/
│   └── workflows/
│       └── swell-review.yml          # PR自動レビューワークフロー
├── .mcp/
│   └── swell-hooks-mcp/              # MCPサーバー（ソース同梱）
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts
│           ├── hooks-data.ts
│           └── customizer-data.ts
├── functions.php                     # 子テーマのエントリポイント（最小構成）
├── style.css                         # 子テーマ定義（Template: swell）
├── .gitignore
├── CLAUDE.md                         # プロジェクト説明（Claude Code向け）
└── README.md                         # セットアップ手順
```

### 含めないもの

- Anthropic API キー（`.env` や設定ファイルに含めない。README に登録手順を記載）
- ローカル環境のパス（`MAMP` のパスなどは `.env.example` に例示のみ）
- `node_modules/`（`.gitignore` で除外し `npm install` で取得する）

## 実装内容

### 1. CLAUDE.md の整備

Claude Code はプロジェクトルートの `CLAUDE.md` を自動で読み込みます。テンプレートリポジトリには、SWELL案件特有のコンテキストを記載した `CLAUDE.md` を用意しておきます。

```markdown
# SWELL子テーマ開発プロジェクト

## プロジェクト概要
このリポジトリはSWELL子テーマの開発用リポジトリです。

## 技術スタック
- WordPress 6.x + SWELLテーマ
- PHP 8.x
- ローカル環境: MAMP Pro

## 利用可能なMCPツール
- `swell-hooks-mcp`: SWELLのフック名・カスタマイザーキーを検索
  - `search_hooks(query)`: フック名を検索
  - `search_customizer(query)`: カスタマイザーキーを検索

## 利用可能なカスタムコマンド
- `/review-swell`: SWELL子テーマのPHPファイルをコードレビュー

## コーディング規約
- PHPのサニタイズ: sanitize_text_field, intval, absint
- 出力エスケープ: esc_html, esc_attr, wp_kses_post
- フックの優先度: SWELLのデフォルト優先度は10
- get_theme_mod のキー名は search_customizer ツールで確認すること

## フロントエンドのみ必要な処理
is_admin() で分岐し、管理画面では不要なスクリプト・スタイルをエンキューしない
```

このファイルがあるだけで、Claude Code は「このプロジェクトがSWELL案件であること」「どのMCPツールを使えるか」を把握した状態でセッションを開始します。毎回説明しなくて済みます。

### 2. functions.php の最小構成テンプレート

空のリポジトリより、SWELL子テーマとして最低限機能する `functions.php` を用意しておきます。

```php
<?php
/**
 * SWELL子テーマ functions.php
 *
 * @package swell-child
 */

defined( 'ABSPATH' ) || exit;

/**
 * 子テーマのスタイルシートを読み込む
 */
add_action( 'wp_enqueue_scripts', 'swell_child_enqueue_styles' );
function swell_child_enqueue_styles() {
    wp_enqueue_style(
        'swell-parent-style',
        get_template_directory_uri() . '/style.css',
        array(),
        wp_get_theme( get_template() )->get( 'Version' )
    );

    wp_enqueue_style(
        'swell-child-style',
        get_stylesheet_uri(),
        array( 'swell-parent-style' ),
        wp_get_theme()->get( 'Version' )
    );
}
```

```css
/*
 Theme Name: SWELL Child
 Template: swell
 Version: 1.0.0
 Text Domain: swell-child
*/
```

よく使うフックのコメントアウト例や `add_filter` / `add_action` のサンプルを含めておくと、案件開始直後からClaude Codeに「このパターンを参考に実装して」と指示できます。

### 3. README.md にセットアップ手順を明記

テンプレートを使う人（未来の自分も含む）が迷わないよう、README に手順を書きます。

```markdown
# SWELL子テーマ開発テンプレート

Claude Code + swell-hooks-mcp + GitHub Actions を使ったSWELL子テーマ開発テンプレートです。

## セットアップ手順（5分）

### 1. テンプレートからリポジトリを作成
「Use this template」→「Create a new repository」でリポジトリを作成します。

### 2. リポジトリをクローン
\`\`\`bash
git clone https://github.com/<your-org>/<your-repo>.git
cd <your-repo>
\`\`\`

### 3. MCPサーバーをビルド
\`\`\`bash
cd .mcp/swell-hooks-mcp
npm install
npm run build
cd ../..
\`\`\`

### 4. Claude Desktop の MCP 設定に追加
\`~/.config/claude/claude_desktop_config.json\` に以下を追加：

\`\`\`json
{
  "mcpServers": {
    "swell-hooks-mcp": {
      "command": "node",
      "args": ["<リポジトリのフルパス>/.mcp/swell-hooks-mcp/dist/index.js"]
    }
  }
}
\`\`\`

Claude Code を再起動すると MCP が有効になります。

### 5. GitHub Secrets に Anthropic API キーを登録
リポジトリの Settings → Secrets and variables → Actions から：
- Name: `ANTHROPIC_API_KEY`
- Value: Anthropicダッシュボードで取得したAPIキー

### 6. MAMPでWordPress + SWELLテーマを起動
MAMP Pro でローカルサーバーを立ち上げ、WordPress に SWELL テーマを適用し、
このリポジトリの内容をSWELL子テーマとして有効化します。

### 完了 🎉
\`cd <your-repo> && claude\` でClaude Code を起動します。
CLAUDE.md が自動で読み込まれ、SWELL案件用のコンテキストが設定されます。
```

### 4. GitHubテンプレートリポジトリとして設定

リポジトリの Settings → General → Template repository にチェックを入れます。これだけで「Use this template」ボタンがリポジトリのトップページに表示されるようになります。

プライベートリポジトリでもテンプレートとして使えるため、自分だけが使う副業ツールとして管理できます。

### 5. setup.sh で手順をさらに自動化

README の手順をそのままスクリプト化した `setup.sh` を用意しておくと、ステップ3以降をワンコマンドで実行できます。

```bash
#!/bin/bash
# SWELL子テーマ開発環境セットアップスクリプト

set -euo pipefail

echo "=== SWELL子テーマ開発テンプレート セットアップ ==="
echo ""

# MCPサーバーをビルド
echo "📦 swell-hooks-mcp をビルドします..."
cd .mcp/swell-hooks-mcp
npm install --silent
npm run build --silent
cd ../..
echo "✅ swell-hooks-mcp のビルド完了"
echo ""

# Claude Desktop 設定ファイルのパスを確認
CLAUDE_CONFIG="$HOME/.config/claude/claude_desktop_config.json"
REPO_DIR="$(pwd)"
MCP_PATH="$REPO_DIR/.mcp/swell-hooks-mcp/dist/index.js"

echo "📋 Claude Desktop の MCP 設定を確認してください："
echo ""
echo "  $CLAUDE_CONFIG に以下を追加："
echo ""
cat << EOF
{
  "mcpServers": {
    "swell-hooks-mcp": {
      "command": "node",
      "args": ["$MCP_PATH"]
    }
  }
}
EOF
echo ""
echo "⚠️  Claude Code を再起動して MCP を有効にしてください。"
echo ""

# GitHub Secretsの案内
echo "🔑 GitHub Secrets の設定："
echo "  リポジトリ Settings → Secrets → Actions"
echo "  ANTHROPIC_API_KEY を登録してください"
echo ""

echo "✨ セットアップ完了！"
echo "  'claude' コマンドでClaude Codeを起動してください。"
```

`bash setup.sh` を実行するだけで、MCPのビルドとパスの表示が自動化されます。Claude Desktop の設定ファイル編集は手動が安全なため、スクリプトでは編集内容を表示するに留めています。

## 実際に使ってみた

新しいSWELL案件（企業サイトのカスタマイズ依頼）が来たタイミングでテンプレートを試しました。

```
所要時間の記録：
- テンプレートからリポジトリ作成: 1分
- git clone + bash setup.sh: 2分
- Claude Desktop 設定の編集 + 再起動: 1分
- GitHub Secrets 登録: 1分
- 合計: 約5分
```

前回まで30分かかっていた環境構築が5分に短縮されました。

`claude` を起動した直後のセッションで、Claude Code が `CLAUDE.md` を読んで「SWELL子テーマの開発プロジェクトですね。フック名の確認は `search_hooks` ツールで行えます」と自発的に案内してくれました。毎回の説明が不要になったのが地味に快適です。

## ポイント

### テンプレートリポジトリの使いどころ

GitHubのテンプレートリポジトリは「fork」と似ていますが、fork と異なり：

- 元リポジトリとのGit履歴が切り離される（案件ごとにクリーンな履歴）
- Organization 内の他のメンバーとも共有できる
- プライベートリポジトリのままテンプレートとして使える

副業ツールとして自分だけが使う場合でも、別のPCや新しいMacに移行したとき、または1年ぶりに案件を受けたときに「前の設定どこだっけ」とならないメリットがあります。

### CLAUDE.md がコンテキスト管理の核心

テンプレートを使う中で最も効果を感じたのは `CLAUDE.md` です。

Claude Code は `CLAUDE.md` の内容を毎回のセッション開始時に自動で読み込みます。「このプロジェクトはSWELL案件だ」「使えるMCPツールはこれだ」「コーディング規約はこれだ」という情報が最初から入っているため、Claude Code の最初の返答の質が上がります。

毎回「このプロジェクトはSWELLテーマのカスタマイズで、フック名はswell-hooks-mcpで確認してください」と説明しなくて済むようになりました。案件数が増えると、このコンテキスト節約の積み重ねが効いてきます。

### MCPサーバーをリポジトリに同梱するメリット

`swell-hooks-mcp` をリポジトリの `.mcp/` 配下に同梱することで：

- バージョン管理ができる（MCPのデータを更新したらコミットするだけ）
- GitHub Actions 上でも同じMCPが使える（前回記事のCI連携と整合）
- チームでの開発時に全員が同じMCPを使える

npmパッケージとして公開するより、同梱の方がシンプルで副業ツールとしての管理コストが低くなります。

### テンプレートの更新と既存リポジトリへの反映

テンプレートリポジトリを更新しても、すでに作成済みのリポジトリには自動反映されません。この点はforkと異なります。

対処として、更新差分を `CHANGELOG.md` に記録しておき、既存リポジトリには必要に応じて手動でコピーする運用にしています。Claude Code に「CHANGELOG.md の最新変更を参考に .claude/commands/review-swell.md を更新して」と依頼すると、自分でファイルを見比べて差分を適用してくれます。

## まとめ

これまでの連載で作ってきた以下の要素を一つのテンプレートリポジトリにまとめました。

| コンポーネント | 役割 |
|---|---|
| `CLAUDE.md` | SWELL案件のコンテキストをClaude Codeに自動提供 |
| `.claude/commands/review-swell.md` | SWELLに特化したコードレビューコマンド |
| `.claude/hooks/` | Stop/StartUpフックによる自動レビュー促進 |
| `.mcp/swell-hooks-mcp/` | フック・カスタマイザー検索MCPサーバー |
| `.github/workflows/swell-review.yml` | PR時の自動コードレビュー |
| `setup.sh` | 環境構築の半自動化 |
| `README.md` | 5分で環境構築できる手順書 |

副業SWELL案件の連載（SWELLフックMCP → カスタマイザーMCP → ローカル自動レビュー → GitHub Actions連携 → 今回のテンプレート化）が一区切りしました。

Claude Code と MCPを活用したSWELL子テーマ開発の自動化ツールチェーンが、再現可能なテンプレートとして形になりました。次の案件から「Use this template」→ `bash setup.sh` → `claude` で、すぐに本題の実装に入れます。

次回は少し視点を変えて、**Claude Code の scheduled tasks（スケジュール実行）機能を使ってZenn記事の下書きを自動生成する**仕組みを試してみる予定です。毎週の作業ログを元に記事ドラフトを自動作成することで、アウトプットの継続を仕組み化できるか実験します。
