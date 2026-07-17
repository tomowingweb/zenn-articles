---
title: "Claude Codeで複数のSWELL副業案件を効率よく管理する"
emoji: "🗂️"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "副業", "wpcli"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-07-10-claude-code-mcp-swell-docs)では、fetch MCPサーバーを使ってSWELL公式ドキュメントをリアルタイム参照する方法を紹介しました。最後に「次回は複数案件の管理方法を書く」と予告したので、今回はそのテーマを掘り下げます。

副業でWordPress制作をしていると、複数のクライアントサイトを同時に抱えることが珍しくありません。「A社サイトはSWELL子テーマ、B社サイトは別の子テーマ、C社サイトはプラグイン開発」という状況で、いちいちコンテキストを切り替えながら作業するのは地味にストレスです。

Claude Codeは**プロジェクトごとに独立した `CLAUDE.md` とセッション履歴**を持てるため、複数案件の管理と相性が非常によいです。今回は私が実際に使っているディレクトリ構成と、Claude Codeのプロジェクト切り替えワークフローをまとめます。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- WP-CLI
- VS Code + Remote Containers（案件によって）

## ディレクトリ構成の設計

### MAMPのhtdocsを案件別に整理する

MAMPのDocumentRoot配下を案件ごとに分けるのが基本です。私は以下の構成を使っています。

```
/Applications/MAMP/htdocs/
├── client-a/                # A社サイト
│   └── wp-content/
│       └── themes/
│           └── swell-child-a/
│               ├── CLAUDE.md       ← 案件固有のコンテキスト
│               ├── style.css
│               └── functions.php
├── client-b/                # B社サイト
│   └── wp-content/
│       └── themes/
│           └── swell-child-b/
│               ├── CLAUDE.md
│               ├── style.css
│               └── functions.php
└── client-c/                # C社（プラグイン開発）
    └── wp-content/
        └── plugins/
            └── my-plugin-c/
                ├── CLAUDE.md
                └── my-plugin-c.php
```

各案件のルートに `CLAUDE.md` を置くのがポイントです。Claude Codeはそのディレクトリで起動すると自動的に `CLAUDE.md` を読み込むため、「この案件はどんなサイトか」をいちいち説明しなくてよくなります。

### CLAUDE.mdのテンプレート

案件ごとの `CLAUDE.md` はほぼ同じ構造で書いています。テンプレートはこちらです。

```markdown
# クライアントA社サイト SWELL子テーマ

## プロジェクト概要
- 種別: WordPressサイト（SWELL子テーマカスタマイズ）
- 目的: コーポレートサイト + ブログ
- WordPress: 6.7.x / SWELL: 最新版
- PHP: 8.1

## ローカル環境
- パス: /Applications/MAMP/htdocs/client-a/
- URL: http://localhost:8888/client-a/
- WP-CLI: wp --path=/Applications/MAMP/htdocs/client-a/

## テーマ構成
- 親テーマ: swell
- 子テーマ: swell-child-a
- style.css → CSS上書き（追加のみ）
- functions.php → requireのみ
- functions/hooks.php → アクション/フィルターフック
- functions/shortcodes.php → カスタムショートコード

## 案件固有の注意事項
- ヘッダーロゴは SVG（クライアント指定）
- お問い合わせフォームはContact Form 7を使用
- SEOプラグインはRank Mathを導入済み
- Google Analytics 4のgtag.jsを直接埋め込む（プラグイン不使用）

## よく使うSWELLフック
- swell_hook_header_before → ヘッダー前の告知バー
- swell_hook_after_footer → フッター後のスクリプト挿入
- swell_hook_after_blogcard_title → ブログカードカスタマイズ

## 参照ドキュメント
- SWELLフック一覧: https://swell-theme.com/docs/php-hook/
- SWELLカスタマイザー: https://swell-theme.com/docs/customizer/
```

この内容があれば、Claude Codeが起動直後から「このプロジェクトはA社のコーポレートサイト、Contact Form 7使用、GA4直接埋め込み」と把握した状態で会話できます。

## プロジェクト切り替えのワークフロー

### ターミナルのディレクトリ移動だけでOK

Claude Codeのプロジェクト切り替えは、**ターミナルで対象案件のディレクトリに移動して `claude` を起動するだけ**です。`CLAUDE.md` が自動ロードされるため、前の案件のコンテキストが残ることはありません。

```bash
# A社案件に切り替え
cd /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a
claude

# B社案件に切り替え
cd /Applications/MAMP/htdocs/client-b/wp-content/themes/swell-child-b
claude
```

私はシェルのエイリアスを設定して、より短いコマンドで切り替えられるようにしています。

```bash
# ~/.zshrc に追加
alias wpa="cd /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a && claude"
alias wpb="cd /Applications/MAMP/htdocs/client-b/wp-content/themes/swell-child-b && claude"
alias wpc="cd /Applications/MAMP/htdocs/client-c/wp-content/plugins/my-plugin-c && claude"
```

`wpa` と入力するだけでA社案件のClaude Codeが起動します。

### セッション履歴の活用

Claude Codeは `/resume` コマンドで過去のセッションを再開できます。「先週のあの作業の続き」という場面で非常に便利です。案件ごとにディレクトリが分かれていると、セッション履歴も自然に案件別で整理されます。

```
# Claude Code起動後
/resume
```

一覧が表示されるので、続きから再開したいセッションを選択します。

## WP-CLIで複数案件を管理する

複数のWordPressを管理するとき、WP-CLIの `--path` オプションで案件を切り替えながら操作できます。Claude Codeのターミナルから直接叩けるのでワークフローに自然に組み込めます。

### よく使うコマンド

```bash
# A社サイトのプラグイン一覧
wp plugin list --path=/Applications/MAMP/htdocs/client-a

# B社サイトのWordPressをアップデート
wp core update --path=/Applications/MAMP/htdocs/client-b

# C社サイトのユーザー一覧
wp user list --path=/Applications/MAMP/htdocs/client-c

# A社サイトのデータベースをエクスポート（バックアップ）
wp db export backup-$(date +%Y%m%d).sql --path=/Applications/MAMP/htdocs/client-a
```

Claude Codeに「A社サイトのプラグインを最新版に更新して、問題がないか確認して」と依頼すると、上記コマンドを組み合わせて実行してくれます。

### WP-CLIコマンドをClaudeに任せるメリット

WP-CLIのオプションは数が多く、毎回ドキュメントを調べるのが手間です。Claude Codeに自然言語で依頼すると適切なコマンドを生成してくれるので、覚える必要がなくなります。

```
# こんな依頼でOK
「A社サイトで投稿IDが100〜200番のものを下書きに戻して」
「B社サイトのContact Form 7の設定をJSONでエクスポートして」
```

## GitでSWELL子テーマをバージョン管理する

複数案件を抱えると、「どのクライアントにどの変更を入れたか」が混乱しがちです。子テーマをGitで管理することで、変更履歴が明確になります。

### 子テーマをGitリポジトリにする

```bash
cd /Applications/MAMP/htdocs/client-a/wp-content/themes/swell-child-a
git init
```

`.gitignore` は以下のように設定します。

```
# .gitignore
*.log
.DS_Store
node_modules/
```

画像やWordPressコアファイルはコミットしないため、子テーマのPHP・CSS・JSのみを管理します。

### Claude Codeとの連携

Claude Codeはgit操作もできます。変更をコミットする際に内容を確認してコミットメッセージを生成させる、というワークフローが便利です。

```
# Claude Codeへの依頼
「今回の変更内容を確認して、適切なgitコミットメッセージを日本語で提案して」
```

Conventionalコミット形式（`feat:`, `fix:`, `style:` など）を使うよう `CLAUDE.md` に書いておくと、コミットの粒度も統一されます。

## 案件間でのコードの再利用

複数案件を運営していると「A社で作ったこの機能、B社にも使えそう」という場面が増えます。Claude Codeを使ったコードの再利用パターンを紹介します。

### スニペットライブラリとしてのGitHub

よく使うSWELLカスタマイズコードをGitHubのプライベートリポジトリにまとめておくと、案件をまたいだ再利用が楽になります。

```
my-swell-snippets/
├── README.md
├── hooks/
│   ├── notice-bar.php        # 告知バー
│   ├── blogcard-date.php     # ブログカード公開日
│   └── custom-breadcrumb.php # パンくずリストカスタマイズ
└── css/
    ├── hover-effects.css     # ホバーエフェクト集
    └── responsive-fixes.css  # レスポンシブ調整
```

Claude Codeに「このスニペットをB社案件のスタイルに合わせて調整して」と依頼すると、コピー元を渡すだけで案件ごとの差分を生成してくれます。

## ハマりどころ

### 1. CLAUDE.mdの記述が古くなる

案件が進むにつれて、プラグインの追加・削除やPHPバージョンの変更が発生します。`CLAUDE.md` を更新し忘れると、Claudeが古い情報を元に提案してきます。

**対策**：月1回、または大きな構成変更のたびに `CLAUDE.md` を見直す。Claude Codeに「現在の状態でCLAUDE.mdを更新して」と依頼するのが一番確実です。

### 2. 案件を跨いだ操作ミス

コマンドに `--path` を付け忘れると、デフォルトのWordPressに操作が走ります。MAMP環境では別の案件に影響が出ることがあります。

**対策**：CLAUDE.mdに `wp コマンドは必ず --path=/Applications/MAMP/htdocs/client-a/ を付けること` と明記する。Claudeはこのルールを守ってコマンドを生成してくれます。

### 3. セッションが長くなるとコンテキストが薄れる

長いセッションを続けていると、会話の最初に話した内容をClaudeが参照しにくくなります。

**対策**：1〜2時間を目安に新しいセッションを立ち上げる。`CLAUDE.md` に重要な情報を集約しておけば、再起動してもすぐに文脈を復元できます。

### 4. 本番環境へのデプロイ管理

ローカルで確認した変更を本番環境にFTPでアップするとき、どのファイルを上書きしたか記録しておかないと混乱します。

**対策**：Gitの差分（`git diff HEAD~1`）をClaude Codeに渡して「この変更で本番に上げるべきファイルをリストアップして」と依頼する。FTPクライアント（Cyberduck等）でのアップロードチェックリストを自動生成できます。

## まとめ

Claude Codeで複数のSWELL副業案件を管理するポイントをまとめます。

| 工夫 | 効果 |
|------|------|
| 案件ごとにディレクトリを分けてCLAUDE.mdを設置 | 起動直後から案件コンテキストが有効 |
| シェルエイリアスで即時切り替え | プロジェクト切り替えの摩擦ゼロ |
| WP-CLIに`--path`を必ずCLAUDE.mdに記載 | 誤操作防止 |
| 子テーマをGitで管理 | 変更履歴が明確になり、案件間の差分管理が楽 |
| スニペットライブラリをGitHubに集約 | コードの再利用が容易 |

副業で複数のWordPress案件を動かしていると「どこで何を変えたか」が混乱しがちですが、Claude Codeは**「起動ディレクトリのCLAUDE.mdを読んでいる」**という特性をうまく使えば、それぞれの案件に専念したAIアシスタントとして機能してくれます。

次回は、Claude CodeのHooks機能を使ってSWELL子テーマの**デプロイ前チェックを自動化**する方法について書く予定です。ローカルで変更したファイルをFTPにアップする前に、PHP構文チェック・WP Coding Standards準拠チェックをClaude Code上で自動実行する仕組みを試しています。
