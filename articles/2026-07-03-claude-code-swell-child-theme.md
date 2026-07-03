---
title: "Claude CodeでSWELL子テーマ開発を効率化した話"
emoji: "🐈"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "childtheme", "php"]
published: false
---

## はじめに

WordPress + SWELLテーマを使った制作案件をこなしつつ、子テーマのカスタマイズをどう効率よく進めるか、ずっと悩んでいました。SWELLはノーコードでも十分カスタマイズできますが、細かいデザイン調整やPHPフックを使った機能追加になると、コードを書く場面が増えてきます。

そこで最近取り入れたのが **Claude Code** をローカル開発環境（MAMP）と組み合わせたワークフローです。「AIにコードを生成してもらう」というより、「開発パートナーとして会話しながら実装を進める」感覚で、子テーマ開発のスピードが体感で2〜3倍になりました。

この記事では、Claude CodeとSWELL子テーマ開発の具体的な組み合わせ方と、実際にハマったポイントをまとめます。

## 環境

- WordPress 6.7.x
- SWELLテーマ（最新バージョン）
- MAMP（ローカル開発環境 / Mac）
- Claude Code（CLI）
- VS Code

## 子テーマの基本構成

SWELLの子テーマは最低限以下のファイルがあれば動きます。

```
my-swell-child/
├── style.css
├── functions.php
└── (必要に応じてfunctions/やcss/など)
```

`style.css` には必ずテーマヘッダーを記述します。

```css
/*
Theme Name: SWELL Child
Template: swell
Version: 1.0.0
*/
```

`functions.php` で親テーマのスタイルを読み込みます。

```php
<?php
add_action('wp_enqueue_scripts', function () {
    wp_enqueue_style(
        'swell-parent-style',
        get_template_directory_uri() . '/style.css'
    );
});
```

この基本構成をClaudeに伝えると、以降の質問がずっとスムーズになります。

## Claude Codeを使った実際のワークフロー

### 1. プロジェクトルートで起動

まずMAMPのWordPressディレクトリを開いてClaude Codeを起動します。

```bash
cd /Applications/MAMP/htdocs/mysite/wp-content/themes/my-swell-child
claude
```

起動後、最初に子テーマの構成とSWELLのフック一覧のリンク（SWELL公式ドキュメント）を伝えます。Claude Codeはファイルを直接読み込めるので、既存の`functions.php`も一緒に渡します。

### 2. フック追加をペアプログラミング感覚で進める

たとえばSWELLのブログカードのタイトル下に公開日を追加したいとき、普通ならSWELLのフック一覧ドキュメントを調べて、該当フックを探して…という手順が必要です。

Claude Codeへの依頼はこんな感じ：

```
ブログカードのサムネイル下、タイトルの下に公開日を表示したい。
SWELLの `swell_hook_after_blogcard_title` フックを使って実装して。
フォーマットは「YYYY年MM月DD日」
```

返ってくるコード：

```php
add_action('swell_hook_after_blogcard_title', function () {
    if (is_singular()) return;
    $date = get_the_date('Y年n月j日');
    echo '<p class="blogcard-date">' . esc_html($date) . '</p>';
});
```

このままコピペして動作確認→必要なら微調整、という流れです。

### 3. カスタムCSSの生成

SWELLはカスタマイザーからCSSを書けますが、子テーマの`style.css`に入れて管理したいケースも多いです。

```
ブログカードのhoverで画像が1.05倍にズームして、影が付くようにしたい。
既存のSWELL構造を壊さないように `.blogcard` セレクタを起点にして
```

Claude Codeはファイルを見た上でセレクタを正確に提案してくれるので、SWELL特有のクラス名ミスが減りました。

## カスタムウィジェットエリアの追加

副業案件でよくある「ヘッダー直下に告知バーを出したい」という要望も、Claude Codeで効率化できます。

### register_sidebar

```php
add_action('widgets_init', function () {
    register_sidebar([
        'name'          => '告知バー',
        'id'            => 'notice-bar',
        'before_widget' => '<div class="notice-bar-widget">',
        'after_widget'  => '</div>',
        'before_title'  => '<p class="notice-bar-title">',
        'after_title'   => '</p>',
    ]);
});
```

### SWELLへのフック挿入

```php
add_action('swell_hook_header_before', function () {
    if (!is_active_sidebar('notice-bar')) return;
    echo '<div class="notice-bar">';
    dynamic_sidebar('notice-bar');
    echo '</div>';
});
```

こういう定型的なコードはClaudeが一発で出してくれます。自分はSWELLのどのフックが正しいかを確認するだけでよく、ドキュメントを行き来する時間が大幅に削減されました。

## WP-CLIとの組み合わせ

MAMPでWP-CLIを使うと、子テーマの有効化やプラグイン管理がターミナルで完結します。Claude Codeのターミナルからそのまま実行できるのも便利です。

```bash
# 子テーマを有効化
wp theme activate my-swell-child --path=/Applications/MAMP/htdocs/mysite

# カスタムフィールドを確認
wp post meta list 1 --path=/Applications/MAMP/htdocs/mysite
```

Claude Codeに「このWP-CLIコマンドの出力を見て問題があれば指摘して」と渡すと、出力結果を解析してくれます。

## ハマりどころ

### 1. SWELLフック名の確認が必須

Claude CodeはSWELLの独自フックについて学習データが完全ではありません。フック名のタイポや、存在しないフックを自信満々に提案してくることがあります。

**対策**：SWELLの公式フック一覧ページをコンテキストに渡す、または提案されたフック名をSWELLの`swell-functions.php`でgrepして確認する。

```bash
grep -r "swell_hook_" /Applications/MAMP/htdocs/mysite/wp-content/themes/swell/ | grep "do_action"
```

### 2. functions.phpの肥大化

Claude Codeと一緒に機能を次々追加していくと、`functions.php`が長くなります。

**対策**：早めにファイルを分割する。

```php
// functions.php
require_once get_stylesheet_directory() . '/functions/widgets.php';
require_once get_stylesheet_directory() . '/functions/blogcard.php';
require_once get_stylesheet_directory() . '/functions/custom-css.php';
```

Claude Codeにこの構成を伝えておくと、新機能追加時にどのファイルに書けばよいか提案してくれます。

### 3. 本番環境との差異

MAMPでは動いたのに本番（さくらインターネットなど）で動かない、というケースがあります。PHP バージョン差異が主な原因です。

Claude Codeに「PHP 7.4でも動くように書いて」と明示するか、ローカルのPHPバージョンを本番と合わせておくのが確実です。

## まとめ

Claude CodeとSWELL子テーマ開発の組み合わせで実感したメリットはこちらです。

| 作業 | Before | After |
|------|--------|-------|
| フック調査 | 都度ドキュメント検索 | Claudeに聞いて即候補を取得 |
| PHP定型コード | ゼロから書く or コピー改変 | 要件を伝えて生成 |
| CSS調整 | セレクタ確認→手書き | 既存ファイル参照で生成 |
| デバッグ | エラーログを目で追う | エラー内容を渡して解析 |

**次のステップ**として、Claude CodeのMCP（Model Context Protocol）を使ってSWELL公式ドキュメントを直接参照させる仕組みを試してみたいと思っています。フック名の確認をAI側でできるようになれば、さらに精度が上がるはずです。

副業Web制作でSWELLを扱っている方は、ぜひClaude Codeをローカル開発環境に組み込んでみてください。最初の設定さえ終われば、すぐに恩恵を感じられると思います。
