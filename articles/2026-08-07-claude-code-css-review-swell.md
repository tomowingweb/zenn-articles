---
title: "Claude CodeでSWELL子テーマのCSS変更を自動レビューする―HooksとPostToolUseで品質チェックを仕組み化"
emoji: "🎨"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "css", "副業"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-07-31-claude-code-hooks-shared-swell)では、Claude CodeのHooks設定を複数のSWELL子テーマ案件で共有するために、シンボリックリンクと一括CIチェックスクリプトを整備する方法を紹介しました。最後に「次回はCSS変更をClaude Codeに自動レビューさせる仕組みを作る」と予告しましたので、今回はその実装です。

副業でWordPress/SWELLのカスタマイズを請け負っていると、CSSの作業は意外と事故りやすいポイントです。具体的には次のような問題が発生しがちです。

- **レスポンシブ対応の漏れ**：PCでは確認したが、スマホ幅のメディアクエリを書き忘れた
- **SWELL固有クラスとの競合**：`.swell-block-button` など、SWELLが内部的に使うクラスを意図せず上書きしてしまう
- **ブレークポイントの不統一**：案件によって `768px` と `769px` が混在している
- **`!important` の乱用**：場当たり的に `!important` を追加し続けてカスケードが崩壊する

これらをレビューするのは人力でやると地味に時間がかかります。Claude CodeのHooks（`PostToolUse`）を使ってCSSファイルを編集するたびに自動チェックが走るようにすると、問題を書いた直後に気づけるようになります。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- PHP 8.1（MAMPに付属）
- Node.js 20.x（stylelint実行用）
- stylelint + stylelint-config-standard

## 実装内容

### 全体の設計

CSSレビューの仕組みは2段構えにします。

1. **stylelint**による静的チェック（構文エラー・CSSの書き方の問題）
2. **Claude Code自身**によるSWELL文脈を考慮したセマンティックレビュー

stylelintは機械的に速く、Claude Codeは文脈を読んで判断する、という役割分担です。両方ともHooksの `PostToolUse` で、CSSファイルへの編集直後に自動実行されます。

### stylelintのセットアップ

まず共通Hooksディレクトリ（`~/dev/swell-shared/`）にstylelintの設定を追加します。

```bash
cd ~/dev/swell-shared
npm init -y
npm install --save-dev stylelint stylelint-config-standard
```

`.stylelintrc.json` を作成します：

```json
{
  "extends": "stylelint-config-standard",
  "rules": {
    "no-duplicate-selectors": true,
    "declaration-no-important": [true, { "severity": "warning" }],
    "color-no-invalid-hex": true,
    "unit-no-unknown": true,
    "selector-class-pattern": null,
    "custom-property-empty-line-before": null
  },
  "ignoreFiles": [
    "**/*.php",
    "**/*.js"
  ]
}
```

`declaration-no-important` は `warning` 扱いにしています。`!important` を完全に禁止するとSWELLのデフォルトスタイルを上書きしたい場面で詰まるため、警告どまりにして把握できるようにします。

### CSS自動チェックスクリプトの作成

`~/dev/swell-shared/hooks/post-css-review.sh` を新規作成します：

```bash
#!/bin/bash
# SWELL子テーマ CSS変更後の自動チェック
# 引数: $1=CLAUDE_TOOL_INPUT(JSON), $2=THEME_DIR

TOOL_INPUT="$1"
THEME_DIR="$2"
SHARED_DIR="$(dirname "$(dirname "$(readlink -f "$0")")")"
STYLELINT="$SHARED_DIR/node_modules/.bin/stylelint"
STYLELINTRC="$SHARED_DIR/.stylelintrc.json"

# 編集されたファイルパスを取得
EDITED_FILE=$(echo "$TOOL_INPUT" | python3 -c \
  "import sys,json; d=json.load(sys.stdin); print(d.get('file_path',''))" 2>/dev/null)

# CSSファイルでなければスキップ
if [[ "$EDITED_FILE" != *.css ]]; then
  exit 0
fi

echo ""
echo "======================================"
echo "  CSS自動レビュー: $(basename "$EDITED_FILE")"
echo "======================================"

# ── 1. stylelint チェック ──────────────────
echo ""
echo "【1/3】stylelint チェック"
if [ -f "$STYLELINT" ]; then
  "$STYLELINT" --config "$STYLELINTRC" "$EDITED_FILE" 2>&1
  LINT_RESULT=$?
  if [ $LINT_RESULT -ne 0 ]; then
    echo "⚠️  stylelintの警告・エラーを確認してください（上記参照）"
  else
    echo "✅ stylelint: 問題なし"
  fi
else
  echo "⚠️  stylelint が見つかりません: $STYLELINT"
  echo "   cd $SHARED_DIR && npm install を実行してください"
fi

# ── 2. SWELLクラス競合チェック ────────────
echo ""
echo "【2/3】SWELLクラス競合チェック"

# SWELLが内部的に使う代表的なクラス一覧
SWELL_CLASSES=(
  "swell-block-button"
  "swell-block-faq"
  "swell-block-step"
  "swell-block-profile"
  "swell-block-checklist"
  "p-blogCard"
  "p-entryCard"
  "p-topNav"
  "l-header"
  "l-footer"
  "l-sidebar"
  "c-btn"
)

CONFLICT_FOUND=0
for CLASS in "${SWELL_CLASSES[@]}"; do
  if grep -q "\.$CLASS" "$EDITED_FILE"; then
    echo "⚠️  SWELLクラスへの直接指定を検出: .$CLASS"
    grep -n "\.$CLASS" "$EDITED_FILE" | sed 's/^/   /'
    CONFLICT_FOUND=1
  fi
done

if [ $CONFLICT_FOUND -eq 0 ]; then
  echo "✅ SWELLクラス競合: 検出なし"
else
  echo "   → 子テーマでのSWELLコアクラス上書きは予期しないデザイン崩れを引き起こす可能性があります"
  echo "   → 必要な場合は親クラスを追加して詳細度を上げることを検討してください"
fi

# ── 3. メディアクエリ一覧表示 ──────────────
echo ""
echo "【3/3】使用中のブレークポイント一覧"
MQ_LIST=$(grep -n "@media" "$EDITED_FILE" | sed 's/^/   /')
if [ -n "$MQ_LIST" ]; then
  echo "$MQ_LIST"
  # ブレークポイントの値を抽出して重複・不統一を検出
  BREAKPOINTS=$(grep -o '[0-9]\+px' "$EDITED_FILE" | grep -E "^(375|576|768|769|900|1024|1200|1280)px$" | sort | uniq -c | sort -rn)
  if [ -n "$BREAKPOINTS" ]; then
    echo ""
    echo "   ブレークポイント使用頻度:"
    echo "$BREAKPOINTS" | sed 's/^/   /'
  fi
else
  echo "   @media クエリなし"
fi

echo ""
echo "======================================"
echo ""
```

スクリプトを実行可能にします：

```bash
chmod +x ~/dev/swell-shared/hooks/post-css-review.sh
```

### settings.jsonへの組み込み

前回作った `settings-template.json` に CSS用のHooksを追加します：

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
          },
          {
            "type": "command",
            "command": "bash THEME_DIR/.claude/hooks/post-css-review.sh \"$CLAUDE_TOOL_INPUT\" THEME_DIR"
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

`Edit` に対して PHP チェック → CSS レビューの順に両方が実行されます。対象ファイルの拡張子で各スクリプト内にガードがあるため、PHPを編集してもCSSスクリプトはすぐ終了しますし、CSSを編集してもPHPスクリプトはスキップされます。

### 動作確認

CSS ファイルを Claude Code で編集すると、ターミナルに自動チェック結果が表示されます。

実際の出力例（`!important` 多用・SWELLクラス直接指定・ブレークポイント不統一が検出された場合）：

```
======================================
  CSS自動レビュー: style.css
======================================

【1/3】stylelint チェック
style.css
  124:3  ⚠  Unexpected invalid value "!important" in declaration (declaration-no-important)
  156:3  ⚠  Unexpected invalid value "!important" in declaration (declaration-no-important)

⚠️  stylelintの警告・エラーを確認してください（上記参照）

【2/3】SWELLクラス競合チェック
⚠️  SWELLクラスへの直接指定を検出: .c-btn
   89: .c-btn { background: #ff6600; }
   → 子テーマでのSWELLコアクラス上書きは予期しないデザイン崩れを引き起こす可能性があります
   → 必要な場合は親クラスを追加して詳細度を上げることを検討してください

【3/3】使用中のブレークポイント一覧
   45: @media (max-width: 768px) {
   78: @media (max-width: 769px) {
   112: @media (max-width: 768px) {

   ブレークポイント使用頻度:
      2 768px
      1 769px

======================================
```

768pxと769pxが混在しているのが一目でわかります。

## ポイント

### なぜClaude Code自身にレビューを「依頼」するのか

`post-css-review.sh` は機械的なチェックです。一方、Claude Codeは作業文脈を把握しているため、チェック結果を見てより深いレビューを求めることもできます。

```
# Claude Codeへの依頼（ファイル編集後）
「今のCSS変更で、SWELLのフロントページに影響しそうな部分を教えて」
```

Hooksの自動チェックが「問題を検出して通知する」役割を担い、Claude Codeが「問題の意味を解釈して修正案を提示する」役割を担う分担です。

### `matcher` の書き方

`"matcher": "Edit"` はClaude CodeのEditツール（ファイル編集）が実行されたときのみHooksが発火する設定です。`Write`（新規ファイル作成）もカバーしたい場合は `"matcher": "Edit|Write"` にします。Regex形式で書けるため柔軟にカスタマイズできます。

### スクリプトの出力とClaude Codeへの影響

Hooksスクリプトの `stdout` はClaude Codeのターミナル表示に出力されます。`stderr` に出力するとClaude Codeがエラーと判断する場合があるため、情報表示はすべて `stdout`（`echo`）に統一します。スクリプトが `exit 1` を返すと Claude Code はツール実行が失敗したと認識し、次のアクションを止めることがあるため、チェック結果がエラーであっても警告どまりにしたい場合は `exit 0` で終了するよう注意します。

### SWELLクラスリストのメンテナンス

SWELLはバージョンアップでクラス名が変わることがあります。`SWELL_CLASSES` 配列は定期的に最新のSWELLソースコードと照合してメンテナンスが必要です。SWELLの公式GitHubリポジトリをウォッチするか、SWELL公式のリリースノートを確認する習慣をつけておくと安心です。

### 複数CSSファイルへの対応

SWELL子テーマでは `style.css`（メインスタイル）のほかに `assets/css/` 以下に分割されたCSSを置くことがあります。スクリプトはファイルパスを引数で受け取る設計なので、どのCSSを編集しても自動でチェックが走ります。

## まとめ

CSSチェックの自動化をまとめます。

| チェック内容 | 手段 | タイミング |
|---|---|---|
| CSSシンタックス・スタイルルール | stylelint | 編集直後（PostToolUse） |
| SWELLコアクラスへの直接指定 | grepスクリプト | 編集直後（PostToolUse） |
| ブレークポイント一覧・不統一の可視化 | grepスクリプト | 編集直後（PostToolUse） |
| 文脈を考慮したセマンティックレビュー | Claude Code本体へ依頼 | 任意 |

副業でのCSS作業は「手戻り」が利益を直撃します。納品後にスマホ表示の崩れを報告されてから直すよりも、書いた直後に気づける仕組みを整えておくほうが、長期的な稼働効率は大きく上がります。

次回は、Claude CodeのMCP（Model Context Protocol）を使ってSWELLの公式ドキュメントを参照しながらカスタマイズを進める方法を試してみる予定です。今のところ、SWELLのフィルターフック一覧をMCP経由でClaude Codeに渡し、「このレイアウト変更にはどのフックを使えばいいか」を聞きながら実装する構成を考えています。
