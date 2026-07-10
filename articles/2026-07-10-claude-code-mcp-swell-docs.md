---
title: "Claude CodeのMCPサーバーでSWELL公式ドキュメントを自動参照する"
emoji: "📡"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "mcp", "php"]
published: false
---

## はじめに

前回の記事（[Claude CodeでSWELL子テーマ開発を効率化した話](https://zenn.dev/tomowingweb/articles/2026-07-03-claude-code-swell-child-theme)）で「Claude CodeのMCP（Model Context Protocol）を使ってSWELL公式ドキュメントを直接参照させる仕組みを試したい」と書きました。

今回はそれを実際に試したので、具体的なセットアップ方法と使い勝手をまとめます。

SWELLのフック名やカスタマイザー設定をClaude Codeが自力で調べてくれるようになると、「フック名が間違っていた」「そのオプションは存在しない」というAI特有のハルシネーションを大幅に減らせます。結論から言うと、**fetch MCPサーバーを活用したドキュメント参照**が現時点でいちばん手軽で効果的でした。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- Node.js 20.x

## MCPとは

MCP（Model Context Protocol）はAnthropicが策定したオープンな仕様で、AI（Claude）と外部ツール・データソースをつなぐためのプロトコルです。Claude Codeでは `.claude/settings.json`（またはグローバルの `~/.claude/settings.json`）にMCPサーバーの設定を書くことで、Claudeがファイルシステムやブラウザ、独自APIなどを「ツール」として使えるようになります。

今回利用するのは **fetch MCPサーバー**です。URLを渡すとページのHTMLを取得してくれるシンプルなサーバーで、Claude Codeが公式ドキュメントを直接読みに行けるようになります。

## セットアップ

### 1. fetch MCPサーバーのインストール

```bash
npm install -g @anthropic-ai/mcp-server-fetch
```

### 2. Claude Codeの設定ファイルに追記

プロジェクトルートの `.claude/settings.json`（なければ作成）に以下を追加します。

```json
{
  "mcpServers": {
    "fetch": {
      "command": "mcp-server-fetch",
      "args": []
    }
  }
}
```

グローバルに使いたい場合は `~/.claude/settings.json` に同様の設定を追記します。

### 3. 動作確認

Claude Codeを再起動して `/mcp` コマンドを実行し、`fetch` サーバーが接続されているか確認します。

```
/mcp
```

`fetch: connected` と表示されれば準備完了です。

## 実際の使い方

### SWELL公式フック一覧を読み込む

Claude Codeとの会話でこう指示します。

```
SWELL公式のアクションフック一覧を以下のURLから取得して、
ヘッダー周辺で使えるフックをリストアップして。
https://swell-theme.com/docs/php-hook/
```

fetchサーバーがページを取得し、Claudeがフック一覧を解析して回答してくれます。以前は「フック名を間違えるリスク」があったのですが、ドキュメントをリアルタイムで参照するため、精度が格段に上がりました。

### カスタマイザー設定の確認

```
SWELLのカスタマイザーで「TOPページのヘッダースライダー」を
制御している設定キーを、以下のドキュメントから調べて。
https://swell-theme.com/docs/customizer-top/
```

こういった質問もfetchで公式ドキュメントを読んで返してくれます。`get_theme_mod()` に渡すキー名を調べる手間がなくなりました。

## CLAUDE.mdを使った恒久的なコンテキスト設定

毎回URLを伝えるのは面倒なので、プロジェクトルートに `CLAUDE.md` を置いて、SWELLに関する基本情報をまとめておく方法が効果的です。

```markdown
# SWELL子テーマ開発プロジェクト

## このプロジェクトについて
- WordPressサイト（SWELL親テーマ）の子テーマ開発
- MAMPローカル環境: /Applications/MAMP/htdocs/mysite/
- PHP 8.1

## よく参照するドキュメント
- SWELLフック一覧: https://swell-theme.com/docs/php-hook/
- SWELLカスタマイザー: https://swell-theme.com/docs/customizer/
- SWELL子テーマ作成: https://swell-theme.com/docs/child-theme/

## ファイル構成
- functions.php → メイン（requireのみ）
- functions/hooks.php → アクション/フィルターフック
- functions/widgets.php → ウィジェットエリア
- style.css → 追加CSS（テーマヘッダー含む）

## コーディング規約
- PHPは8.0+の記法OK（アロー関数、名前付き引数など）
- CSSはSWELL既存クラスを尊重し、必要最小限のセレクタで上書き
- コメントは日本語でOK
```

このファイルを置いておくと、Claude Codeが起動時に読み込み、毎回「このプロジェクトはSWELL子テーマ開発です」と説明しなくて済みます。

## Hooksを使った自動化との組み合わせ

Claude CodeのHooks機能（コマンド実行の前後に任意のスクリプトを走らせる機能）と組み合わせることで、さらに便利になります。

たとえば、Claudeが `functions.php` を編集するたびに構文チェックを自動実行する設定：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "if echo '$CLAUDE_TOOL_INPUT' | grep -q 'functions'; then php -l /Applications/MAMP/htdocs/mysite/wp-content/themes/my-swell-child/functions.php && echo 'PHP syntax OK'; fi"
          }
        ]
      }
    ]
  }
}
```

PHPのシンタックスエラーがあればClaudeがすぐ気づいて修正してくれるようになります。

## ハマりどころ

### 1. fetchサーバーがページを取得できないケース

SWELL公式ドキュメントはほぼ問題なく取得できますが、JavaScriptで動的にコンテンツを生成しているページは空のHTMLが返ることがあります。その場合は、静的なドキュメントページを探すか、事前に必要な情報をコピーして `CLAUDE.md` に貼り付けておく方が確実です。

### 2. ドキュメントが長すぎてコンテキストを圧迫する

フック一覧など量が多いページを丸ごと読ませると、Claudeのコンテキストウィンドウを大量に消費します。

**対策**：必要なセクションのアンカーリンクを渡す（例：`https://swell-theme.com/docs/php-hook/#header`）か、「〇〇に関する部分だけ抜き出して」と伝えて要約させる。

### 3. MCPサーバーのバージョン更新

`@anthropic-ai/mcp-server-fetch` はアップデートが頻繁です。動かなくなったら `npm update -g @anthropic-ai/mcp-server-fetch` で最新版に上げてください。

### 4. CLAUDE.mdのパス

`CLAUDE.md` はClaude Codeを起動したディレクトリから親ディレクトリに向かって再帰的に読み込まれます。子テーマのディレクトリで起動している場合、WordPress のルートや `themes/` ディレクトリに `CLAUDE.md` を置いても読み込まれることを覚えておくと、複数テーマを管理するときに便利です。

## 導入前後の比較

| 作業 | 導入前 | 導入後 |
|------|--------|--------|
| フック名確認 | ブラウザでSWELL公式を検索 | Claudeに聞いてfetch取得 |
| カスタマイザーキー確認 | ドキュメントをCtrl+F検索 | 自然言語で質問して即回答 |
| フックのハルシネーション | 時々発生（存在しないフック名） | ほぼゼロ（ドキュメント参照） |
| 毎回のコンテキスト説明 | 必要 | CLAUDE.mdで自動化 |
| PHP構文エラー | 実行して初めて発覚 | Hooksで編集直後に検知 |

## まとめ

Claude CodeのMCPサーバー（fetch）を設定することで、SWELL公式ドキュメントをリアルタイムで参照しながら子テーマ開発ができるようになりました。設定は `settings.json` に数行追加するだけなので、SWELL + Claude Code で開発している方にはすぐ試せるのでおすすめです。

`CLAUDE.md` との組み合わせでプロジェクト固有のコンテキストを固定化し、HooksでPHP構文チェックを自動化することで、**Claude Codeが間違いに自分で気づいて修正するサイクル**が回るようになります。

副業案件で複数のSWELLサイトを管理している方は、案件ごとにリポジトリを作って `CLAUDE.md` を用意しておくと、プロジェクトのスイッチが楽になります。次回は、複数案件を管理するためのディレクトリ構成と、Claude Codeのプロジェクト切り替えワークフローについて書く予定です。
