---
title: "Claude CodeのHooks設定を複数SWELL案件で共有する―シンボリックリンクとCIチェックの整備"
emoji: "🔗"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "php", "副業"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-07-24-claude-code-hooks-deploy-check)では、Claude CodeのHooks機能を使ってSWELL子テーマのデプロイ前チェック（PHPシンタックス検証・WP Coding Standards確認・デプロイ対象ファイル一覧の自動生成）を自動化する方法を紹介しました。

最後に「Hooksの設定を複数案件で共有するためのシンボリックリンク管理と、CI的なチェックの整備について書く」と予告していたので、今回はそのテーマです。

前回の設定は「各案件ディレクトリに `.claude/hooks/` を独立配置する」構成でした。これは動きますが、案件が3〜4件に増えると**スクリプトの更新漏れ**という問題が発生します。A社案件のHooksを改善したのにB社・C社に反映し忘れる、というケースが実際に起きました。

今回は以下の2点を整備します。

1. **シンボリックリンクで共通Hooksを一元管理する**
2. **全案件を横断してCIチェックを走らせるスクリプトを作る**

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- PHP 8.1（MAMPに付属）
- PHP_CodeSniffer + WordPress Coding Standards
- Bash 5.x

## 実装内容

### 共通Hooksディレクトリの設計

まず、全案件から参照する「共通Hooks」の置き場所を決めます。私は `~/dev/swell-shared/` というディレクトリを作り、そこに共通スクリプトをまとめました。

```
~/dev/
└── swell-shared/
    ├── hooks/
    │   ├── post-edit-check.sh      # 編集後のPHP構文・WPCSチェック
    │   ├── pre-deploy-report.sh    # デプロイ対象ファイル一覧生成
    │   └── ci-check-all.sh         # 全案件一括チェック（新規追加）
    └── settings-template.json      # Hooks設定のテンプレート
```

スクリプトはここに1本だけ保管し、各案件からはシンボリックリンクで参照します。

### 各案件へのシンボリックリンク設置

各案件の `.claude/hooks/` を共通ディレクトリにリンクします。

```bash
# A社案件
mkdir -p /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a/.claude
ln -s ~/dev/swell-shared/hooks \
  /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a/.claude/hooks

# B社案件
mkdir -p /Applications/MAMP/htdocs/client-b/wp-content/themes/swell-child-b/.claude
ln -s ~/dev/swell-shared/hooks \
  /Applications/MAMP/htdocs/client-b/wp-content/themes/swell-child-b/.claude/hooks

# C社案件
mkdir -p /Applications/MAMP/htdocs/client-c/wp-content/themes/swell-child-c/.claude
ln -s ~/dev/swell-shared/hooks \
  /Applications/MAMP/htdocs/client-c/wp-content/themes/swell-child-c/.claude/hooks
```

リンクの確認：

```bash
ls -la /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a/.claude/
# hooks -> /Users/yourname/dev/swell-shared/hooks
```

これで `~/dev/swell-shared/hooks/post-edit-check.sh` を更新すれば、全案件に自動的に反映されます。

### settings.jsonのテンプレート化

`settings.json` はシンボリックリンクにできないため（パスが案件ごとに異なるため）、テンプレートから生成する方式にします。

`~/dev/swell-shared/settings-template.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash THEME_DIR/.claude/hooks/post-edit-check.sh \"$CLAUDE_TOOL_INPUT\" THEME_DIR"
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
            "command": "bash THEME_DIR/.claude/hooks/pre-deploy-report.sh THEME_DIR"
          }
        ]
      }
    ]
  }
}
```

テンプレート内の `THEME_DIR` を案件ごとのパスで置き換えるセットアップスクリプトを作ります。

`~/dev/swell-shared/setup-hooks.sh`:

```bash
#!/bin/bash
# 使い方: ./setup-hooks.sh /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a

THEME_DIR="$1"

if [ -z "$THEME_DIR" ]; then
  echo "Usage: $0 <theme-dir>"
  exit 1
fi

CLAUDE_DIR="$THEME_DIR/.claude"
mkdir -p "$CLAUDE_DIR"

# シンボリックリンク設置
if [ ! -L "$CLAUDE_DIR/hooks" ]; then
  ln -s ~/dev/swell-shared/hooks "$CLAUDE_DIR/hooks"
  echo "✅ hooks リンクを設置しました"
fi

# settings.json 生成
sed "s|THEME_DIR|$THEME_DIR|g" ~/dev/swell-shared/settings-template.json \
  > "$CLAUDE_DIR/settings.json"

echo "✅ settings.json を生成しました: $CLAUDE_DIR/settings.json"
```

新しい案件が増えたときは `./setup-hooks.sh <テーマパス>` を1回実行するだけです。

### 共通スクリプトのパス対応

前回のスクリプトではテーマパスをハードコードしていましたが、共通化に伴い**引数でパスを受け取る**ように修正します。

`~/dev/swell-shared/hooks/post-edit-check.sh`（抜粋）:

```bash
#!/bin/bash
TOOL_INPUT="$1"
THEME_DIR="$2"
PHP_BIN="/Applications/MAMP/bin/php/php8.1.x/bin/php"

EDITED_FILE=$(echo "$TOOL_INPUT" | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print(d.get('file_path',''))" 2>/dev/null)

if [ -z "$EDITED_FILE" ]; then
  exit 0
fi

if [[ "$EDITED_FILE" == *.php ]]; then
  echo "=== PHP構文チェック: $EDITED_FILE ==="
  "$PHP_BIN" -l "$EDITED_FILE"
  PHP_RESULT=$?
  if [ $PHP_RESULT -ne 0 ]; then
    echo "❌ PHPシンタックスエラーが検出されました。"
    exit 1
  fi

  echo "=== WordPress Coding Standards チェック ==="
  phpcs --standard=WordPress --severity=5 --extensions=php "$EDITED_FILE"
  echo "✅ チェック完了"
fi
```

`pre-deploy-report.sh` も同様に第1引数で `THEME_DIR` を受け取るよう修正します。

### 全案件一括CIチェックスクリプト

複数案件を横断して「現在どの案件にPHPエラーがあるか」を一目で確認できるスクリプトを追加します。

`~/dev/swell-shared/hooks/ci-check-all.sh`:

```bash
#!/bin/bash
# 全SWELL子テーマ案件の一括PHPチェック

PHP_BIN="/Applications/MAMP/bin/php/php8.1.x/bin/php"
THEMES_BASE="/Applications/MAMP/htdocs"
ERROR_FOUND=0

echo "=========================================="
echo "  SWELL子テーマ 全案件 PHP チェック"
echo "  $(date '+%Y-%m-%d %H:%M:%S')"
echo "=========================================="

for THEME_DIR in "$THEMES_BASE"/*/wp-content/themes/swell-child-*/; do
  CLIENT=$(echo "$THEME_DIR" | sed 's|.*/htdocs/\([^/]*\)/.*|\1|')
  echo ""
  echo "--- $CLIENT ---"

  ERRORS=$(find "$THEME_DIR" -name "*.php" -not -path "*/.git/*" \
    -exec "$PHP_BIN" -l {} \; 2>&1 | grep -v "No syntax errors")

  if [ -n "$ERRORS" ]; then
    echo "❌ PHPエラーあり:"
    echo "$ERRORS"
    ERROR_FOUND=1
  else
    echo "✅ PHPエラーなし"
  fi

  # 変更ファイルがあれば表示
  if git -C "$THEME_DIR" rev-parse --git-dir > /dev/null 2>&1; then
    CHANGED=$(git -C "$THEME_DIR" diff HEAD --name-only 2>/dev/null)
    if [ -n "$CHANGED" ]; then
      echo "📝 未コミットの変更:"
      echo "$CHANGED" | sed 's/^/   /'
    fi
  fi
done

echo ""
echo "=========================================="
if [ $ERROR_FOUND -eq 0 ]; then
  echo "  ✅ 全案件チェック完了：PHPエラーなし"
else
  echo "  ❌ エラーが検出されました。上記を確認してください。"
fi
echo "=========================================="
exit $ERROR_FOUND
```

このスクリプトを `~/.zshrc` にエイリアス登録しておくと便利です。

```bash
# ~/.zshrc
alias wpcheck="bash ~/dev/swell-shared/hooks/ci-check-all.sh"
```

作業前に `wpcheck` を叩くと全案件の現状が一覧表示されます。

### Claude Codeからの一括チェック依頼

Claude Codeのセッション内から一括チェックを呼び出すこともできます。

```
# Claude Codeへの依頼
「全SWELL案件のPHPチェックを走らせて、エラーがあれば修正案を教えて」
```

Claude Codeは `ci-check-all.sh` を実行し、エラーが検出されたファイルを開いて修正案を提示してくれます。

## ポイント

### シンボリックリンクとgitの相性

子テーマをGitで管理している場合、`.claude/hooks` がシンボリックリンクになっていると `git status` に表示されることがあります。Gitにコミットしたくない場合は `.gitignore` に追加します。

```
# .gitignore
.claude/
```

逆に「Hookスクリプトを案件ごとのリポジトリで管理したい」という場合は、シンボリックリンクではなく `git submodule` を使う手もあります。ただし設定の複雑さが増すため、副業規模では素のシンボリックリンクで十分だと感じています。

### MAMPのPHPバージョン切り替え

案件によってMAMPのPHPバージョンが異なる場合（A社はPHP8.1、B社はPHP7.4など）、共通スクリプトの `PHP_BIN` をどう扱うかが問題になります。

対処方法として、`THEME_DIR` の `.claude/php-version` ファイルにバージョン文字列を書いておき、スクリプト内で読み込む方式にしました。

```bash
# .claude/php-version の内容
8.1.x

# スクリプト内での読み込み
PHP_VERSION=$(cat "$THEME_DIR/.claude/php-version" 2>/dev/null || echo "8.1.x")
PHP_BIN="/Applications/MAMP/bin/php/php${PHP_VERSION}/bin/php"
```

### ci-check-all.sh の実行速度

PHPファイルが多い案件では `find + php -l` の直列実行が遅くなります。`xargs -P 4` で並列化すると体感速度が向上しました。

```bash
find "$THEME_DIR" -name "*.php" -not -path "*/.git/*" \
  | xargs -P 4 -I{} "$PHP_BIN" -l {} 2>&1 \
  | grep -v "No syntax errors"
```

### Hooksスクリプトのテスト方法

スクリプトを更新したときの動作確認は、テスト用PHPファイルを使います。

```bash
# 意図的にシンタックスエラーを含むファイルを作成
echo "<?php echo 'test'" > /tmp/test-error.php

# スクリプトを直接実行して動作確認
echo '{"file_path":"/tmp/test-error.php"}' \
  | bash ~/dev/swell-shared/hooks/post-edit-check.sh \
    '{"file_path":"/tmp/test-error.php"}' \
    /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a
```

Claude Codeのセッション内で試す前に、コマンドラインで単体動作を確認する習慣をつけると、デバッグが楽になります。

## まとめ

Hooksの共通化と一括CIチェックの整備をまとめます。

| 工夫 | 効果 |
|------|------|
| `~/dev/swell-shared/hooks/` に共通スクリプト集約 | 更新が全案件に即反映 |
| `setup-hooks.sh` で案件初期化を自動化 | 新規案件の設定が1コマンドで完了 |
| `ci-check-all.sh` で全案件横断チェック | 複数案件のPHPエラーと未コミット変更を一望 |
| `.claude/php-version` で案件別PHPバージョン管理 | 共通スクリプトのまま複数PHPバージョンに対応 |

副業でのWordPress制作は「同時進行案件が増えるほど管理コストが膨らむ」という構造的な問題があります。Claude CodeのHooksを共通化することで、**どの案件でも同じ品質チェックが走る**状態を維持しやすくなりました。

次回は、このCI的なチェックをさらに発展させて、**SWELL子テーマのCSS変更をClaude Codeに自動でレビューさせる**仕組みを試してみる予定です。具体的には、CSSの変更をHooksで検知して、メディアクエリの漏れ・SWELL固有クラスとの競合・ブレークポイントの一貫性などを自動チェックするアイデアを持っています。
