---
title: "Claude CodeでSWELL子テーマ開発を爆速化した話"
emoji: "🐈"
type: "tech"
topics: ["wordpress", "swell", "claudecode", "子テーマ", "css"]
published: false
---

## はじめに

WordPressの有料テーマ「SWELL」を使ったWeb制作で、子テーマのカスタマイズ作業にかなりの時間を取られていました。「このブロックだけフォントサイズを変えたい」「スクロール時にヘッダーの背景色を変えたい」といった細かい要望が積み重なり、CSSもfunctions.phpも肥大化していく一方。

そこで今週から本格的に **Claude Code** をSWELL子テーマ開発に組み込んでみたところ、作業スピードと品質が大きく変わりました。この記事では、具体的にどんな使い方をしているかをまとめます。

---

## 環境

- WordPress 6.8
- SWELL 2.11.x
- 子テーマ構成（`swell-child/`）
- MAMP（ローカル開発環境、PHP 8.2）
- Claude Code（CLI、VS Code拡張）
- Node.js / npm（Zenn CLI用）

---

## 今週やった主な作業

### 1. カスタムCSSの整理とコメント補完

子テーマの `style.css` が200行を超えてきたタイミングで、Claude Codeに整理を依頼しました。

```bash
# style.cssを読み込ませて整理を依頼
claude "このstyle.cssを読んで、セクションごとにコメントを追加してください。
スタイルの重複があれば指摘してください。"
```

Claude Codeは既存のクラス名からSWELLのブロック構造を推測し、以下のようにセクション分けしてくれました。

```css
/* ==============================================
   01. グローバルリセット・基本設定
   ============================================== */
:root {
  --accent-color: #3a7bd5;
  --text-color: #333;
}

/* ==============================================
   02. ヘッダーカスタマイズ
   ============================================== */
#site-header {
  border-bottom: 2px solid var(--accent-color);
}

/* ==============================================
   03. 記事本文スタイル
   ============================================== */
.p-article__body h2 {
  padding-left: 0.8em;
  border-left: 4px solid var(--accent-color);
}
```

重複していたメディアクエリも整理され、可読性が一気に上がりました。

---

### 2. functions.phpへのフック追加

SWELLではウィジェットやショートコードを `functions.php` で拡張することが多いですが、「どのフックを使えばいいか」「優先度はどう設定するか」で迷うことが多々あります。

今週はフッターに著作権表示＋SNSリンクを追加する実装をClaude Codeに依頼しました。

```bash
claude "SWELLの子テーマfunctions.phpに、フッターのコピーライト表示を
カスタマイズするコードを追加してください。
SNSリンク（Twitter/X、Instagram）も表示したいです。"
```

生成されたコード：

```php
<?php
/**
 * フッターカスタムコンテンツの追加
 */
add_action( 'swell_footer_before_copyright', 'my_footer_social_links' );
function my_footer_social_links() {
    ?>
    <div class="footer-social-links">
        <a href="https://twitter.com/your_account" target="_blank" rel="noopener noreferrer"
           aria-label="X（旧Twitter）">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                <path d="M18.244 2.25h3.308l-7.227 8.26 8.502 11.24H16.17l-4.714-6.231-5.401 6.231H2.744l7.737-8.835L1.254 2.25H8.08l4.253 5.622zm-1.161 17.52h1.833L7.084 4.126H5.117z"/>
            </svg>
        </a>
        <a href="https://instagram.com/your_account" target="_blank" rel="noopener noreferrer"
           aria-label="Instagram">
            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="currentColor">
                <path d="M12 2.163c3.204 0 3.584.012 4.85.07 3.252.148 4.771 1.691 4.919 4.919.058 1.265.069 1.645.069 4.849 0 3.205-.012 3.584-.069 4.849-.149 3.225-1.664 4.771-4.919 4.919-1.266.058-1.644.07-4.85.07-3.204 0-3.584-.012-4.849-.07-3.26-.149-4.771-1.699-4.919-4.92-.058-1.265-.07-1.644-.07-4.849 0-3.204.013-3.583.07-4.849.149-3.227 1.664-4.771 4.919-4.919 1.266-.057 1.645-.069 4.849-.069zM12 0C8.741 0 8.333.014 7.053.072 2.695.272.273 2.69.073 7.052.014 8.333 0 8.741 0 12c0 3.259.014 3.668.072 4.948.2 4.358 2.618 6.78 6.98 6.98C8.333 23.986 8.741 24 12 24c3.259 0 3.668-.014 4.948-.072 4.354-.2 6.782-2.618 6.979-6.98.059-1.28.073-1.689.073-4.948 0-3.259-.014-3.667-.072-4.947-.196-4.354-2.617-6.78-6.979-6.98C15.668.014 15.259 0 12 0zm0 5.838a6.162 6.162 0 100 12.324 6.162 6.162 0 000-12.324zM12 16a4 4 0 110-8 4 4 0 010 8zm6.406-11.845a1.44 1.44 0 100 2.881 1.44 1.44 0 000-2.881z"/>
            </svg>
        </a>
    </div>
    <?php
}

add_filter( 'swell_copyright_text', 'my_custom_copyright' );
function my_custom_copyright( $text ) {
    return '© ' . date('Y') . ' Your Site Name. All Rights Reserved.';
}
```

SWELLのフック名（`swell_footer_before_copyright`）を正確に把握していたのが助かりました。公式ドキュメントを調べる手間が省けました。

---

### 3. CLAUDE.mdによるプロジェクト仕様の定義

Claude Codeをチームメンバーのように扱うには、プロジェクトの仕様を `CLAUDE.md` に書いておくのが効果的です。今週は子テーマ用の `CLAUDE.md` を整備しました。

```markdown
# SWELL子テーマ開発ルール

## プロジェクト概要
- テーマ：SWELL 2.11.x の子テーマ
- 対象サイト：コーポレートサイト（中小企業向け）
- PHPバージョン：8.2

## コーディングルール
- CSSは BEM記法 を使用（ブロック: `p-`, コンポーネント: `c-`, ユーティリティ: `u-`）
- PHPのコールバック関数名はすべて `my_` プレフィックスを付ける
- jQueryは使用しない（バニラJSで書く）
- メディアクエリのブレークポイント：
  - SP: max-width 767px
  - TB: 768px ～ 1024px
  - PC: min-width 1025px

## SWELLフックリスト（よく使うもの）
- `swell_head_prepend`: <head>の先頭
- `swell_footer_before_copyright`: フッターコピーライト前
- `swell_singular_before_content`: 投稿本文の前

## ファイル構成
- style.css: スタイル全般
- functions.php: フック・フィルター追加
- js/custom.js: カスタムJS（vanilla）
```

これを置いておくだけで、「BEMで書いて」「my_プレフィックスつけて」と毎回指示しなくて済みます。

---

## ハマりどころと解決策

### SWELLのフック名が古い情報だった

SWELL 2.x系でフック名が変わっているものがあり、Claude Codeが古いバージョンのフック名を出してくることがありました。

**対処法：** フック名が不明なときはSWELLの本体ファイルをgrep確認する。

```bash
# SWELLのフォルダを検索
grep -r "do_action('swell_" /path/to/swell/ --include="*.php" | grep -v ".min." | head -20
```

これをClaude Codeに渡すと、正確なフック名で再生成してくれます。

### カスタムCSSがSWELLのスタイルに負ける

詳細度（specificity）の問題で子テーマのスタイルが効かないケースがありました。

```bash
claude "このCSSルールがSWELLのデフォルトスタイルに負けています。
詳細度を上げて上書きしてください。ただし!importantは使わないでください。"
```

Claude Codeは詳細度を適切に調整したセレクタを提案してくれました。`!important` を使わずに解決できるのが◎。

---

## 使ってみての感想

| 作業 | 以前 | Claude Code導入後 |
|------|------|------------------|
| functions.phpへのフック追加 | 30分（調査含む） | 5分 |
| CSSの整理・コメント追加 | 60分 | 10分 |
| SVGアイコンの実装 | 20分 | 3分 |
| CLAUDE.md整備 | ― | 1回のみ（以降ゼロ） |

特にSWELL固有のフック・フィルター調査の時間が大幅に削減されました。

---

## まとめ

Claude CodeをSWELL子テーマ開発に組み込むポイントをまとめます。

1. **`CLAUDE.md` にプロジェクト仕様を書く** ── コーディングルール、ブレークポイント、よく使うフック名を定義しておく
2. **SWELLのフック名は必ずソースで確認** ── 古い情報が出ることがあるので、grep結果を渡して修正させる
3. **`!important` 禁止ルールをCLAUDE.mdに入れる** ── 詳細度の問題を正しいアプローチで解決させる

次のステップとして、Gutenbergブロックのカスタムスタイルをblock.jsonとClaude Codeで管理する方法も試していきたいと思います。

もしSWELL開発でClaude Codeを使っている方がいれば、ぜひ活用法を教えてください！
