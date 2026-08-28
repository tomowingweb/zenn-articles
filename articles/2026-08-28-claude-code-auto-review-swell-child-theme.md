---
title: "Claude CodeでSWELL子テーマを自動コードレビューする仕組みを作った"
emoji: "🔍"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "mcp", "php"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-08-21-claude-code-mcp-swell-customizer)では、SWELLカスタマイザーの設定キーをMCPサーバーから即検索できる仕組みを構築しました。フックとカスタマイザーの両方をClaude Codeがオフラインで即答できる状態になり、副業SWELL案件の実装速度が大幅に上がっています。

前回の末尾に「次は **Claude Codeに子テーマのコードレビューを自動で走らせる** 仕組みを試す」と書きました。今回はその実践報告です。

SWELL子テーマを実装した後、気になるのが以下の点です。

- **SWELLテーマとの互換性**は問題ないか（廃止フックを使っていないか、優先度の設定ミスがないか）
- **WordPress標準のコーディング規約**に沿っているか（エスケープ漏れ、nonce未検証など）
- **パフォーマンスに影響する書き方**をしていないか（不要なクエリ、フロントでの不要な処理など）

これらを毎回手動で確認するのは時間がかかります。Claude Codeを使ってファイル保存のたびに自動レビューが走る環境を作りました。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- Node.js 20.x
- `swell-hooks-mcp` v1.1.0（前回構築したMCPサーバー）
- `@modelcontextprotocol/sdk` v1.x

## 設計方針

完全に自動化することよりも、**レビューのトリガーを明示的にして結果を確認しやすくする**ことを優先しました。理由は2つです。

1. ファイル保存ごとに自動レビューが走ると、軽微な変更でも毎回出力が出てノイズになる
2. レビュー結果は「見て判断する」もので、自動マージや自動修正は副業案件では怖い

最終的に採用した設計は次のとおりです。

```
Claude Code Hooks（Stop時）
  └── php-review.sh を実行
        └── 変更ファイル一覧を取得（git diff）
              └── Claude Codeに /review コマンドを発火
                    └── swell-hooks-mcp でフック互換性を照合
                          └── レビュー結果を review-report.md に書き出し
```

Claude CodeのStopフック（セッション終了時）に紐付けることで、「実装の一区切り」のタイミングで自動レビューが走るようにしています。

## 実装内容

### 1. レビュー用カスタムスラッシュコマンドの作成

`.claude/commands/review-swell.md` を作成します。これが `/review-swell` コマンドの本体です。

```markdown
# SWELL子テーマ コードレビュー

以下の手順でSWELL子テーマのPHPファイルをレビューしてください。

## レビュー対象
$ARGUMENTS で指定されたファイル（未指定時は git diff --name-only HEAD で変更ファイルを対象）

## チェック項目

### 1. フック互換性（swell-hooks-mcp を使用）
- search_hooks で使用フック名を照合し、SWELLで有効なフックか確認
- 廃止済みフックが使われていないかチェック
- 優先度（priority）の設定が推奨範囲内か

### 2. WordPressセキュリティ
- ユーザー入力のサニタイズ（sanitize_text_field, intval 等）
- 出力のエスケープ（esc_html, esc_attr, wp_kses_post 等）
- nonceの検証（wp_verify_nonce）
- current_user_can による権限チェック

### 3. get_theme_mod の使い方
- search_customizer でキー名の正しさを確認
- デフォルト値の指定漏れがないか
- 型に合わせたキャストをしているか

### 4. パフォーマンス
- フロントエンドでのみ必要な処理が is_admin() で分岐されているか
- WP_Query を使う場合に posts_per_page や no_found_rows を適切に設定しているか
- 同じ処理を複数のフックで重複実行していないか

## 出力形式

レビュー結果を以下の形式で `review-report.md` に書き出してください：

```markdown
# SWELL子テーマ コードレビュー結果
日時: {日時}
対象ファイル: {ファイル名}

## 🔴 要修正
（致命的な問題・セキュリティリスク）

## 🟡 推奨修正
（品質改善・ベストプラクティス準拠）

## 🟢 問題なし
（確認済みの項目）

## 💡 改善提案
（次のアクションへの提案）
```
```

### 2. Stopフックの設定

`.claude/settings.json` に Stopフックを追加します。

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "*.php",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/php-review.sh"
          }
        ]
      }
    ]
  }
}
```

`matcher` に `*.php` を指定することで、PHPファイルが変更に含まれるセッション終了時のみフックが発火します。

### 3. php-review.sh の作成

`.claude/hooks/php-review.sh` を作成します。

```bash
#!/bin/bash
# SWELL子テーマ PHPレビュートリガー

set -euo pipefail

# 変更されたPHPファイルを取得
CHANGED_PHP=$(git diff --name-only HEAD 2>/dev/null | grep '\.php$' || true)

# 変更PHPがなければスキップ
if [ -z "$CHANGED_PHP" ]; then
  echo "PHPファイルの変更なし。レビューをスキップします。"
  exit 0
fi

echo "=== SWELL子テーマ PHPレビュー ==="
echo "変更ファイル:"
echo "$CHANGED_PHP"
echo ""
echo "Claude Codeで /review-swell を実行してください:"
echo "  /review-swell $CHANGED_PHP"
echo ""

# レビュー実行フラグファイルを作成（次回起動時に自動レビューを促す）
echo "$CHANGED_PHP" > .claude/.pending-review
echo "レビュー待ちファイルを .claude/.pending-review に記録しました。"
```

Stopフック時点ではClaude Codeの対話セッションが終了しているため、スクリプト内から Claude Code を呼び出すことはできません。そのため「次回の起動時に自動で `/review-swell` を実行する」仕組みと組み合わせています。

### 4. StartUpフックで自動レビューを発火

`.claude/settings.json` に StartUpフックを追加します。

```json
{
  "hooks": {
    "Stop": [...],
    "StartUp": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/check-pending-review.sh"
          }
        ]
      }
    ]
  }
}
```

`check-pending-review.sh` では、`.claude/.pending-review` が存在する場合にその内容を標準出力に流します。Claude Code は StartUp フックの出力をコンテキストに含めるため、「前回セッションでレビュー待ちのファイルがある」という情報を自然に受け取れます。

```bash
#!/bin/bash
# 保留中レビューチェック

PENDING=".claude/.pending-review"

if [ -f "$PENDING" ]; then
  FILES=$(cat "$PENDING")
  echo "【前回セッションのレビュー待ちファイルがあります】"
  echo "$FILES"
  echo ""
  echo "/review-swell を実行してSWELL子テーマのコードレビューを開始することを推奨します。"
  rm -f "$PENDING"
fi
```

これにより、次回 `claude` を起動した時点で Claude Code が「レビューを実行しますか？」と自発的に提案してくれるようになります。

### 5. 実際のレビュー実行例

```
> /review-swell functions.php includes/custom-post-type.php
```

Claude Code は以下の流れでレビューを実行します。

**① フック互換性チェック（MCPツール呼び出し）**

```
search_hooks(query="swell_before_header")
→ フック名: swell_before_header（有効）、推奨優先度: 10
```

```
search_hooks(query="swell_before_single_content")
→ フック名: swell_before_single_content（有効）、推奨優先度: 10
```

**② カスタマイザーキーのチェック**

```
search_customizer(query="header_height")
→ key: header_height、型: integer、デフォルト: 80
```

`functions.php` で `get_theme_mod('headerheight')` と書いていた箇所（タイポ）が発見されました。

**③ セキュリティチェック**

`custom-post-type.php` 内に `$_POST` を直接使っている箇所があり、`sanitize_text_field()` の適用漏れを指摘されました。

**④ レポート出力（review-report.md）**

```markdown
# SWELL子テーマ コードレビュー結果
日時: 2026-08-28 10:23
対象ファイル: functions.php, includes/custom-post-type.php

## 🔴 要修正
- **[functions.php:47]** `get_theme_mod('headerheight', 80)` → 正しいキーは `header_height`
  - search_customizer で確認済み。タイポによりデフォルト値のみが常に返る状態。

## 🟡 推奨修正
- **[custom-post-type.php:89]** `$_POST['meta_value']` を直接使用
  - `sanitize_text_field($_POST['meta_value'])` に修正することを推奨
- **[functions.php:62]** `add_action('swell_before_header', ...)` の優先度が 99
  - SWELLの推奨優先度は 10。意図して遅延させている場合はコメントで明記を

## 🟢 問題なし
- nonceの検証（wp_verify_nonce）：実装済み ✓
- 出力エスケープ：esc_html, esc_attr が適切に使用されている ✓
- is_admin() 分岐：フロント処理が正しく分岐されている ✓

## 💡 改善提案
- `suggest_customizer` ツールの実装（前々回の提案）を進めると、
  カスタマイザーキー選択ミスをコード作成前に防げる可能性がある
- `custom-post-type.php` のPOST処理は `class-meta-handler.php` に切り出すと
  責務が明確になりテストしやすくなる
```

## ポイント

### Stopフック ＋ StartUpフックの組み合わせが有効

Claude Code のフックには「セッション中のファイル変更にリアルタイム反応する」ような機能はなく、Stop/StartUp/PostToolUse などのタイミングに縛られます。今回の設計では：

- **Stop**：変更されたPHPファイルをリストアップして保存
- **StartUp**：次回起動時に「レビュー待ちあり」を伝える

この2段構えにすることで、「実装を終えて次に開いた瞬間にレビューを促す」自然なフローになりました。ファイル保存のたびに割り込まれず、一区切りついたタイミングで確認できるのが快適です。

### MCPサーバーとの連携でレビュー精度が上がる

汎用的なコードレビューツールはWordPressの文脈で正確なレビューをするのが難しく、SWELLのフック名やカスタマイザーキーの正誤については特に苦手です。今回 `swell-hooks-mcp` をレビューコマンド内で使うことで：

- フック名のタイポや廃止フックの使用を具体的なエラーとして検出できる
- カスタマイザーキーの間違い（今回の `headerheight` の例）を発見できる

汎用AIレビューと、SWELL固有のナレッジを持つMCPサーバーを組み合わせることで、副業案件の現場で使えるレビュー精度になりました。

### `review-report.md` を Git 管理する

レビュー結果を `review-report.md` に書き出し、Git で管理することにしました。`.gitignore` には追加せず、コミット履歴と一緒に残す運用です。

メリットは：
- 「このコミットの実装時にどのレビュー指摘があったか」が後から追跡できる
- 繰り返し指摘される項目を把握して、テンプレートのコーディング習慣を改善できる
- 案件の引き継ぎ時に「どこを気をつけたか」の文書として使える

副業の単発案件では特に、半年後に同じクライアントから修正依頼が来たときに「以前のレビューで指摘されていたが修正しなかった」箇所を素早く特定できるメリットがあります。

### カスタムスラッシュコマンドのポータビリティ

`.claude/commands/review-swell.md` はプロジェクトリポジトリに含まれるため、チームや別PCへの移行が簡単です。副業で複数のSWELL案件を持っている場合、各案件のリポジトリに同じコマンドを置くだけで同じレビュー体験が再現できます。

MCP設定（`claude_desktop_config.json` の `mcpServers` 設定）はグローバルに入れているため、どの案件でも `swell-hooks-mcp` のデータを参照できます。

## まとめ

今回実装した自動レビュー環境をまとめます。

| コンポーネント | 役割 |
|---|---|
| `/review-swell` コマンド | SWELLに特化したレビュー手順を定義 |
| `php-review.sh`（Stopフック） | 変更PHPファイルを記録 |
| `check-pending-review.sh`（StartUpフック） | 次回起動時にレビューを促す |
| `swell-hooks-mcp` | フック・カスタマイザーキーの正誤を照合 |
| `review-report.md` | レビュー結果をGit管理 |

これまでの連載で構築してきた `swell-hooks-mcp`（フック検索 → カスタマイザー対応 → 今回のコードレビュー連携）が、副業SWELL案件の開発ツールとして一つのまとまった形になってきました。

今週の案件では `get_theme_mod` のキー名タイポを本番反映前に発見できており、「気づかないまま常にデフォルト値が返る」というサイレントなバグを防げました。実際に役立っています。

次回は、このレビュー仕組みを発展させて **PR（プルリクエスト）作成時に自動でGitHub Actionsからレビューを走らせる** 構成を試す予定です。ローカルでのClaude Codeレビューに加えて、GitHub上での自動レビューも整備することで、副業案件でのコード品質管理をさらに体系化していきます。
