---
title: "Claude CodeのHooks機能でSWELL子テーマのデプロイ前チェックを自動化する"
emoji: "🔍"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "php", "副業"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-07-17-claude-code-multi-project-swell)では、Claude Codeで複数のSWELL副業案件を効率よく管理するディレクトリ構成とワークフローを紹介しました。最後に「次回はHooks機能を使ったデプロイ前チェックを書く」と予告したので、今回はそのテーマです。

副業でWordPress/SWELLの子テーマカスタマイズをしていると、「ローカルで動いていたのに本番に上げたら壊れた」という経験が一度はあるはずです。特に多いのが以下のケースです。

- PHPのシンタックスエラー（閉じ括弧の抜け・文字コードのBOM混入など）
- WordPressコーディング規約違反（セキュリティフィルター未通過の変数など）
- FTP転送時の対象ファイル漏れ

これらをClaude CodeのHooks機能を使って、**ファイル編集の直後・コミット直前に自動チェックする仕組み**を構築しました。今回はその設定方法と実際の使い勝手をまとめます。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- PHP 8.1（MAMPに付属）
- PHP_CodeSniffer + WordPress Coding Standards
- Node.js 20.x（Hooks内のスクリプト実行用）

## Claude CodeのHooks機能とは

Hooksは、Claude Codeが特定のツール（ファイル編集・コマンド実行など）を使う**前後に、任意のシェルコマンドを自動実行する機能**です。設定は `.claude/settings.json` の `hooks` セクションに記述します。

トリガーの種類は主に以下の4つです。

| トリガー | タイミング |
|----------|------------|
| `PreToolUse` | ツール実行前 |
| `PostToolUse` | ツール実行後 |
| `Notification` | Claude からの通知時 |
| `Stop` | Claude が応答を止めるとき |

今回のデプロイ前チェックでは `PostToolUse`（ファイル編集後）と `Stop`（Claudeが作業完了を宣言するとき）を主に使います。

## 事前準備：PHP_CodeSnifferの導入

WP Coding Standardsのチェックには PHP_CodeSniffer が必要です。Composerでインストールします。

```bash
# Composerがない場合
brew install composer

# PHP_CodeSnifferのインストール
composer global require squizlabs/php_codesniffer

# WordPress Coding Standardsの追加
composer global require wp-coding-standards/wpcs
phpcs --config-set installed_paths ~/.composer/vendor/wp-coding-standards/wpcs
```

動作確認：

```bash
phpcs --standard=WordPress --version
# PHP_CodeSniffer version x.x.x (stable) by Squiz (http://www.squiz.net)
```

## Hooks設定

### プロジェクトルートに設定ファイルを作る

子テーマのルート（例：`swell-child-a/`）に `.claude/settings.json` を作成します。これにより、この案件のClaude Codeセッションだけにチェックが適用されます。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a/.claude/hooks/post-edit-check.sh \"$CLAUDE_TOOL_INPUT\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "bash /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a/.claude/hooks/pre-deploy-report.sh"
          }
        ]
      }
    ]
  }
}
```

### PostToolUse：ファイル編集直後のチェック

`.claude/hooks/post-edit-check.sh` を以下の内容で作成します。

```bash
#!/bin/bash
# ファイル編集後に PHP 構文チェックと WPCS チェックを実行

THEME_DIR="/Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a"
PHP_BIN="/Applications/MAMP/bin/php/php8.1.x/bin/php"

# 編集されたファイルパスをtool_inputのJSONから取り出す
EDITED_FILE=$(echo "$1" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('file_path',''))" 2>/dev/null)

if [ -z "$EDITED_FILE" ]; then
  exit 0
fi

# PHPファイルの場合のみチェック
if [[ "$EDITED_FILE" == *.php ]]; then
  echo "=== PHP構文チェック: $EDITED_FILE ==="
  "$PHP_BIN" -l "$EDITED_FILE"
  PHP_RESULT=$?

  if [ $PHP_RESULT -ne 0 ]; then
    echo "❌ PHPシンタックスエラーが検出されました。修正してください。"
    exit 1
  fi

  echo "=== WordPress Coding Standards チェック ==="
  phpcs --standard=WordPress --severity=5 --extensions=php "$EDITED_FILE"
  WPCS_RESULT=$?

  if [ $WPCS_RESULT -ne 0 ]; then
    echo "⚠️  WP Coding Standards の警告があります（上記を確認してください）"
    # 警告は exit 1 にせず、Claudeに情報として渡す（終了コード0）
    exit 0
  fi

  echo "✅ チェック完了：問題なし"
fi
```

PHPのシンタックスエラーはHookの終了コードを `1` にしてClaudeに通知し、自動修正を促します。WP Coding Standardsの警告は情報として表示するだけにしておき、Claude Codeが判断できるようにしています。

### Stop：作業完了時のデプロイレポート生成

Claudeが「作業が完了しました」と宣言するタイミングで、変更されたファイルの一覧レポートを自動生成します。

`.claude/hooks/pre-deploy-report.sh` を作成します。

```bash
#!/bin/bash
# Claudeの作業完了時にデプロイ対象ファイルのレポートを生成

THEME_DIR="/Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a"
REPORT_FILE="$THEME_DIR/.claude/last-deploy-files.txt"

cd "$THEME_DIR" || exit 0

# Gitがある場合：前回コミットからの変更ファイルを出力
if git rev-parse --git-dir > /dev/null 2>&1; then
  echo "=== デプロイ対象ファイル一覧 (git diff HEAD) ==="
  git diff HEAD --name-only | tee "$REPORT_FILE"
  STAGED=$(git diff --cached --name-only)
  if [ -n "$STAGED" ]; then
    echo "=== ステージ済みファイル ==="
    echo "$STAGED" | tee -a "$REPORT_FILE"
  fi
else
  # Gitがない場合：直近24時間に変更されたファイルを出力
  echo "=== 直近24時間に変更されたPHP/CSSファイル ==="
  find "$THEME_DIR" -name "*.php" -newer "$THEME_DIR/style.css" -not -path "*/.git/*" \
    | tee "$REPORT_FILE"
  find "$THEME_DIR" -name "*.css" -newer "$THEME_DIR/style.css" -not -path "*/.git/*" \
    | tee -a "$REPORT_FILE"
fi

echo ""
echo "📋 上記のファイルをFTPで本番環境にアップしてください。"
echo "   レポートを保存しました: $REPORT_FILE"
```

このスクリプトはGitの差分を使ってデプロイ対象ファイルを自動判定します。Gitを使っていない案件の場合はファイルのタイムスタンプで代替しています。

## 実際の動作

設定後、Claude Codeで子テーマのPHPファイルを編集してもらうと、以下のような流れで自動チェックが走ります。

```
[Claude] functions/hooks.php を編集します...
[Hook実行] post-edit-check.sh
=== PHP構文チェック: functions/hooks.php ===
No syntax errors detected in functions/hooks.php
=== WordPress Coding Standards チェック ===
FILE: functions/hooks.php
...
FOUND 2 ERRORS, 1 WARNING AFFECTING 3 LINES

3 | ERROR | [x] Expected 1 space after "if" keyword
7 | ERROR | [x] Expected "Yoda condition"; found non-Yoda condition
...
❌ PHPシンタックスエラーが検出されました。修正してください。

[Claude] 指摘を確認しました。修正します...
```

Claude CodeはHookからのフィードバックを受け取ると、自動的に修正に取り掛かります。「修正してください」と言わなくても、エラーを見てすぐ直してくれるのが便利です。

## ハマりどころ

### 1. MAMPのPHPパスが通っていない

Hooksのコマンドはログインシェルではなく非インタラクティブシェルで実行されるため、`~/.zshrc` に設定したPATHが読み込まれないことがあります。

**対策**：スクリプト内でPHPのフルパス（`/Applications/MAMP/bin/php/php8.1.x/bin/php`）を指定する。MAMPのバージョンによってパスが異なるので、`ls /Applications/MAMP/bin/php/` で確認してください。

### 2. $CLAUDE_TOOL_INPUTのJSON形式

`$CLAUDE_TOOL_INPUT` の形式はClaude Codeのバージョンによって変わることがあります。`python3 -c "import sys,json; ..."` で解析していますが、動作しない場合は以下でデバッグできます。

```bash
# Hookスクリプトの先頭に追加してデバッグ
echo "$CLAUDE_TOOL_INPUT" >> /tmp/hook-debug.log
```

実際の出力を確認して、キー名を調整してください。

### 3. phpcs が見つからない

Composerでグローバルインストールした `phpcs` が Hooks 内で見つからない場合、PATHが解決されていません。

**対策**：`settings.json` のコマンドにフルパスで記述する。

```json
"command": "/Users/yourname/.composer/vendor/bin/phpcs --standard=WordPress ..."
```

### 4. Hooks が実行されない

`.claude/settings.json` の JSON が壊れているとHooksが無効になります。JSONバリデーターで構文を確認してください。また、ファイルのパーミッションにも注意が必要です。

```bash
chmod +x .claude/hooks/post-edit-check.sh
chmod +x .claude/hooks/pre-deploy-report.sh
```

## 運用してみての感想

2週間ほど使ってみた感想です。

**よかった点**

- PHP構文エラーのまま本番にアップするミスがゼロになった
- Claudeが編集後すぐにエラーに気づいて修正まで完結してくれるため、私が確認する回数が減った
- デプロイ対象ファイルの一覧が自動生成されるので、FTPのアップロード漏れが減った

**課題**

- WP Coding Standardsのルールが厳しすぎて、警告が大量に出る場合がある。今は重要度5以上に絞っているが、プロジェクトに合わせたカスタムルールセットが必要かもしれない
- Hookのシェルスクリプトが複数案件で別々に管理になるため、更新が漏れやすい。シンボリックリンクで共通スクリプトに向けることで解決できそう

## まとめ

Claude CodeのHooks機能を活用することで、SWELL子テーマ開発のデプロイ前チェックを自動化できました。設定のポイントをまとめます。

| 設定 | 効果 |
|------|------|
| PostToolUse + php -l | 編集直後にPHP構文エラーを検知・即修正 |
| PostToolUse + phpcs | WP Coding Standards 違反を即通知 |
| Stop + gitの差分 | 作業完了時にデプロイ対象ファイルを自動出力 |

副業のWordPress案件では「スピード感」と「本番を壊さない安心感」の両立が重要です。Claude Codeが自分でチェックして自分で直すサイクルが回るようになると、人間は意思決定と最終確認に集中できるようになります。

次回は、このHooksの設定を複数案件で共有するためのシンボリックリンク管理と、CI的な観点でのチェックを整備する方法について書く予定です。

---

本記事で紹介したスクリプトは以下の点に注意してください。MAMPのPHPパスやテーマのパスは環境に合わせて書き換えてください。スクリプトをそのままコピーして動かない場合は、デバッグ用の `echo` を仕込んでログを確認するのが早道です。
