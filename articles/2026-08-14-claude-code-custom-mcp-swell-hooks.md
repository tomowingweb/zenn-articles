---
title: "SWELLフック専用カスタムMCPサーバーをClaude Codeに組み込む"
emoji: "🔌"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "mcp", "php"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-08-07-claude-code-css-review-swell)では、Claude CodeのHooksを使ってCSS変更を自動レビューする仕組みを整えました。最後に「次はMCP経由でSWELLのフィルターフック一覧をClaude Codeに渡し、"このレイアウト変更にはどのフックを使えばいいか"を聞きながら実装する構成を試す」と予告していましたので、今回がその実践編です。

[2026-07-10の記事](https://zenn.dev/tomowingweb/articles/2026-07-10-claude-code-mcp-swell-docs)では `@anthropic-ai/mcp-server-fetch` を使ってSWELL公式ドキュメントをリアルタイム取得する方法を紹介しました。あの仕組みは手軽で便利ですが、副業で日常的に使うと**課題**が見えてきます。

- 毎回ページをフェッチするためレスポンスに数秒かかる
- ドキュメントのHTMLを解析するためコンテキスト消費が大きい
- 「ヘッダーを制御するフック」のように曖昧な質問には答えにくい
- SWELLのバージョンアップで意図せずURL・構造が変わることがある

これらを解決するために、今回は**SWELLフック情報を JSON で持つ独自MCPサーバー**を Node.js で作ります。Claude Code がそのサーバーに問い合わせることで、オフラインでも高速・高精度にフック提案が受けられるようになります。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- Node.js 20.x（MCPサーバー実行用）
- `@modelcontextprotocol/sdk` v1.x

## 全体設計

今回作るものの全体像は次のとおりです。

```
Claude Code
    │  (MCPプロトコル / stdio)
    ▼
swell-hooks-mcp（独自MCPサーバー）
    │
    ├─ tools:
    │   ├─ search_hooks(query)   → フックをキーワード検索
    │   ├─ get_hook(name)        → フックの詳細を取得
    │   └─ suggest_hooks(intent) → 「〇〇したい」に合うフックを提案
    │
    └─ swell-hooks.json（フックデータDB）
```

MCPサーバーはstdioトランスポートで起動し、Claude Codeが子プロセスとして管理します。フックデータはJSONファイルとして手元に持つため、ネット接続なしで即座に応答します。

## 実装

### 1. ディレクトリ構成

前回整備した共通ディレクトリに MCP サーバーを追加します。

```
~/dev/swell-shared/
├── hooks/                    # Claude Code Hooks スクリプト（既存）
├── mcp/
│   └── swell-hooks-mcp/
│       ├── package.json
│       ├── index.js          # MCPサーバー本体
│       └── data/
│           └── swell-hooks.json  # フックデータ
└── settings-template.json    # Hooks設定テンプレート（既存）
```

### 2. パッケージの初期化

```bash
mkdir -p ~/dev/swell-shared/mcp/swell-hooks-mcp/data
cd ~/dev/swell-shared/mcp/swell-hooks-mcp
npm init -y
npm install @modelcontextprotocol/sdk
```

### 3. フックデータJSONの作成

SWELL公式ドキュメントを参照しながら、よく使うフックをJSONにまとめます。一度作っておけば、SWELL更新時に差分だけ追加すればよくなります。

`data/swell-hooks.json`:

```json
[
  {
    "name": "swell_before_header",
    "type": "action",
    "area": "header",
    "description": "ヘッダーHTMLの直前に出力を挿入する",
    "example": "add_action('swell_before_header', function() { echo '<div class=\"notice\">お知らせ</div>'; });",
    "use_cases": ["ヘッダー上部にお知らせバーを追加", "ヘッダー前にカスタムHTML挿入"]
  },
  {
    "name": "swell_after_header",
    "type": "action",
    "area": "header",
    "description": "ヘッダーHTMLの直後に出力を挿入する",
    "example": "add_action('swell_after_header', function() { echo '<nav class=\"breadcrumb\">...</nav>'; });",
    "use_cases": ["ヘッダー下にパンくずリストを固定表示", "ナビゲーション追加"]
  },
  {
    "name": "swell_before_footer",
    "type": "action",
    "area": "footer",
    "description": "フッターHTMLの直前に出力を挿入する",
    "example": "add_action('swell_before_footer', function() { get_template_part('template-parts/cta'); });",
    "use_cases": ["CTAバナーをフッター上部に固定表示", "ページ下部への追加コンテンツ"]
  },
  {
    "name": "swell_after_footer",
    "type": "action",
    "area": "footer",
    "description": "フッターHTMLの直後に出力を挿入する",
    "example": "add_action('swell_after_footer', function() { echo '<div id=\"cookie-notice\">...</div>'; });",
    "use_cases": ["クッキー同意バナー", "フッター後のモーダルHTMLを挿入"]
  },
  {
    "name": "swell_before_content",
    "type": "action",
    "area": "content",
    "description": "記事コンテンツの直前に出力を挿入する",
    "example": "add_action('swell_before_content', function() { if (is_single()) echo '<div class=\"toc\">...</div>'; });",
    "use_cases": ["目次の手動挿入", "記事の前書きを動的追加"]
  },
  {
    "name": "swell_after_content",
    "type": "action",
    "area": "content",
    "description": "記事コンテンツの直後に出力を挿入する",
    "example": "add_action('swell_after_content', function() { get_template_part('template-parts/author-box'); });",
    "use_cases": ["著者プロフィールボックスの追加", "記事後の関連記事リスト"]
  },
  {
    "name": "swell_before_sidebar",
    "type": "action",
    "area": "sidebar",
    "description": "サイドバーの直前に出力を挿入する",
    "example": "add_action('swell_before_sidebar', function() { echo '<div class=\"sidebar-top-ad\">広告</div>'; });",
    "use_cases": ["サイドバー上部に広告挿入", "サイドバー前の区切り要素"]
  },
  {
    "name": "swell_post_thumbnail",
    "type": "filter",
    "area": "post",
    "description": "投稿のサムネイル（アイキャッチ）HTMLをフィルターする",
    "example": "add_filter('swell_post_thumbnail', function($html, $post_id) { return '<div class=\"thumb-wrap\">' . $html . '</div>'; }, 10, 2);",
    "use_cases": ["アイキャッチ画像をラッパーdivで囲む", "サムネイルのレイアウト変更"]
  },
  {
    "name": "swell_entry_card_title",
    "type": "filter",
    "area": "card",
    "description": "記事カードのタイトルHTMLをフィルターする",
    "example": "add_filter('swell_entry_card_title', function($title) { return mb_strimwidth($title, 0, 40, '…'); });",
    "use_cases": ["記事カードのタイトル文字数制限", "タイトルへの動的テキスト追加"]
  },
  {
    "name": "swell_archive_title",
    "type": "filter",
    "area": "archive",
    "description": "アーカイブページのH1タイトルをフィルターする",
    "example": "add_filter('swell_archive_title', function($title) { return 'ブログ一覧：' . $title; });",
    "use_cases": ["アーカイブタイトルの変更", "カテゴリー名に接頭辞を追加"]
  },
  {
    "name": "swell_ogp_title",
    "type": "filter",
    "area": "seo",
    "description": "OGP（SNSシェア用）のタイトルをフィルターする",
    "example": "add_filter('swell_ogp_title', function($title) { return $title . ' | サイト名'; });",
    "use_cases": ["SNSシェア時のタイトルをカスタマイズ", "OGPタイトルへのサイト名付与"]
  },
  {
    "name": "swell_body_classes",
    "type": "filter",
    "area": "layout",
    "description": "bodyタグのclass属性をフィルターする",
    "example": "add_filter('swell_body_classes', function($classes) { if (is_page('about')) $classes[] = 'page-about'; return $classes; });",
    "use_cases": ["ページ別のbodyクラス追加", "条件に応じたCSSフック用クラスの付与"]
  },
  {
    "name": "swell_nav_menus",
    "type": "filter",
    "area": "navigation",
    "description": "登録されるナビゲーションメニューの一覧をフィルターする",
    "example": "add_filter('swell_nav_menus', function($menus) { $menus['campaign'] = 'キャンペーンナビ'; return $menus; });",
    "use_cases": ["カスタムメニューエリアの追加", "ナビゲーションメニューの増設"]
  },
  {
    "name": "swell_widget_areas",
    "type": "filter",
    "area": "widget",
    "description": "登録されるウィジェットエリアをフィルターする",
    "example": "add_filter('swell_widget_areas', function($areas) { $areas[] = ['id' => 'custom-sidebar', 'name' => '追加サイドバー']; return $areas; });",
    "use_cases": ["ウィジェットエリアの追加", "固有の表示エリアをウィジェット化"]
  },
  {
    "name": "swell_customizer_sections",
    "type": "filter",
    "area": "customizer",
    "description": "カスタマイザーのセクション定義をフィルターする",
    "example": "add_filter('swell_customizer_sections', function($sections) { /* カスタムセクション追加 */ return $sections; });",
    "use_cases": ["カスタマイザーへの独自設定セクション追加"]
  }
]
```

実際のSWELLにはさらに多くのフックがあります。使う機会が生じるたびに追加していくのが現実的です。

### 4. MCPサーバー本体（index.js）

```javascript
#!/usr/bin/env node
'use strict';

const { Server } = require('@modelcontextprotocol/sdk/server/index.js');
const { StdioServerTransport } = require('@modelcontextprotocol/sdk/server/stdio.js');
const { CallToolRequestSchema, ListToolsRequestSchema } = require('@modelcontextprotocol/sdk/types.js');
const path = require('path');
const fs = require('fs');

// フックデータの読み込み
const DATA_PATH = path.join(__dirname, 'data', 'swell-hooks.json');
const hooks = JSON.parse(fs.readFileSync(DATA_PATH, 'utf8'));

// MCPサーバーの初期化
const server = new Server(
  { name: 'swell-hooks-mcp', version: '1.0.0' },
  { capabilities: { tools: {} } }
);

// ── ツール定義 ─────────────────────────────────────

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: 'search_hooks',
      description: 'キーワードでSWELLのフックを検索する。フック名・説明・エリア・ユースケースを対象に部分一致で検索。',
      inputSchema: {
        type: 'object',
        properties: {
          query: {
            type: 'string',
            description: '検索キーワード（日本語可）。例: "ヘッダー", "footer", "thumbnail"'
          }
        },
        required: ['query']
      }
    },
    {
      name: 'get_hook',
      description: 'フック名を指定して詳細情報（説明・サンプルコード・ユースケース）を取得する。',
      inputSchema: {
        type: 'object',
        properties: {
          name: {
            type: 'string',
            description: 'フック名。例: "swell_before_header"'
          }
        },
        required: ['name']
      }
    },
    {
      name: 'suggest_hooks',
      description: '「〇〇したい」という意図を渡すと、適切なSWELLフックを提案する。',
      inputSchema: {
        type: 'object',
        properties: {
          intent: {
            type: 'string',
            description: 'やりたいこと。例: "ヘッダーの上に告知バーを追加したい"'
          }
        },
        required: ['intent']
      }
    }
  ]
}));

// ── ツール実装 ─────────────────────────────────────

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'search_hooks') {
    const q = (args.query || '').toLowerCase();
    const results = hooks.filter(h =>
      h.name.toLowerCase().includes(q) ||
      h.description.toLowerCase().includes(q) ||
      h.area.toLowerCase().includes(q) ||
      h.use_cases.some(uc => uc.toLowerCase().includes(q))
    );

    if (results.length === 0) {
      return { content: [{ type: 'text', text: `"${args.query}" に一致するフックは見つかりませんでした。` }] };
    }

    const text = results.map(h =>
      `### ${h.name} (${h.type} / ${h.area})\n${h.description}\n**ユースケース**: ${h.use_cases.join('、')}`
    ).join('\n\n');

    return { content: [{ type: 'text', text: `**検索結果: ${results.length}件**\n\n${text}` }] };
  }

  if (name === 'get_hook') {
    const hook = hooks.find(h => h.name === args.name);
    if (!hook) {
      return { content: [{ type: 'text', text: `フック "${args.name}" は見つかりませんでした。` }] };
    }
    const text = [
      `## ${hook.name}`,
      `- **種別**: ${hook.type}`,
      `- **エリア**: ${hook.area}`,
      `- **説明**: ${hook.description}`,
      `- **ユースケース**: ${hook.use_cases.join('、')}`,
      `\n**サンプルコード**:\n\`\`\`php\n${hook.example}\n\`\`\``
    ].join('\n');
    return { content: [{ type: 'text', text }] };
  }

  if (name === 'suggest_hooks') {
    const intent = (args.intent || '').toLowerCase();
    // use_casesとdescriptionで関連度スコアを計算
    const scored = hooks.map(h => {
      let score = 0;
      const text = (h.description + ' ' + h.use_cases.join(' ')).toLowerCase();
      // 単語単位でスコア加算
      intent.split(/[\s　、。]+/).forEach(word => {
        if (word.length > 1 && text.includes(word)) score += 2;
      });
      // エリアの直接一致はボーナス
      if (intent.includes(h.area)) score += 3;
      return { hook: h, score };
    });

    const suggestions = scored
      .filter(s => s.score > 0)
      .sort((a, b) => b.score - a.score)
      .slice(0, 5);

    if (suggestions.length === 0) {
      return {
        content: [{
          type: 'text',
          text: `"${args.intent}" に対応するフックが見つかりませんでした。\nフックデータを追加するか、search_hooks で別のキーワードをお試しください。`
        }]
      };
    }

    const text = suggestions.map(({ hook: h }) =>
      `### ${h.name} (${h.type})\n${h.description}\n\`\`\`php\n${h.example}\n\`\`\``
    ).join('\n\n');

    return {
      content: [{
        type: 'text',
        text: `**"${args.intent}" に適したフック候補 (${suggestions.length}件)**\n\n${text}`
      }]
    };
  }

  return { content: [{ type: 'text', text: `不明なツール: ${name}` }], isError: true };
});

// ── 起動 ──────────────────────────────────────────

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  // stdoutはMCPプロトコル用なのでログはstderrへ
  process.stderr.write('swell-hooks-mcp: 起動しました\n');
}

main().catch(err => {
  process.stderr.write(`Fatal: ${err.message}\n`);
  process.exit(1);
});
```

### 5. 実行権限の付与

```bash
chmod +x ~/dev/swell-shared/mcp/swell-hooks-mcp/index.js
```

### 6. Claude Codeへの組み込み

各SWELL子テーマプロジェクトの `.claude/settings.json` にMCPサーバーを追加します。

```json
{
  "mcpServers": {
    "swell-hooks": {
      "command": "node",
      "args": ["/Users/yourname/dev/swell-shared/mcp/swell-hooks-mcp/index.js"]
    }
  },
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
    ]
  }
}
```

既存のHooks設定はそのまま残し、`mcpServers` を追加するだけです。

前回作った `setup-hooks.sh` に MCP 設定の自動挿入も追加しておくと、新規案件セットアップがさらに楽になります。

### 7. 動作確認

Claude Codeを再起動して `/mcp` コマンドを実行します。

```
/mcp
```

`swell-hooks: connected` と表示されれば接続成功です。

## 実際の使い方

### フックをキーワードで探す

```
# Claude Codeへの指示
「ヘッダーに関係するSWELLのフックを調べて」
```

Claude Code は `search_hooks` ツールを呼び出し、`swell_before_header`・`swell_after_header` などのフックを一覧表示します。

### やりたいことからフックを提案させる

```
「SWELLのフッターの直前にCTAバナーを表示したい。どのフックを使えばいい？」
```

`suggest_hooks(intent="フッターの直前にCTAバナーを表示したい")` が呼ばれ、`swell_before_footer` とサンプルコードが返ります。そのままClaude Codeに「このフックで実装して」と続けると、`functions.php` に追記してくれます。

### フックの詳細とサンプルコードを確認

```
「swell_body_classes フックの使い方を教えて」
```

`get_hook(name="swell_body_classes")` で詳細情報とPHPサンプルコードが即座に返ります。

## ポイント

### stdioトランスポートを選んだ理由

MCPサーバーのトランスポート方式には `stdio`（標準入出力）と `HTTP/SSE`（ネットワーク）の2択があります。今回は `stdio` にしています。

- **起動が速い**：Claude Codeが子プロセスとして管理するため、設定するだけで自動起動・終了します
- **認証不要**：ローカル専用なので認証周りのコードが不要
- **ネット不要**：オフライン環境でも動作する（データがローカルJSONのため）

HTTP型は複数のプロジェクト・ツールから共有する場合に有利ですが、今回の用途では stdio で十分です。

### フックデータのメンテナンス戦略

SWELLはアップデートが頻繁で、バージョンによってフックが追加・変更されます。`swell-hooks.json` のメンテナンス方法は2つ考えています。

**方法1: 気づいたときに手動追加**
実装中に新しいフックを使ったら、その都度JSONに追加する。確実な情報だけが入るので品質が高い。

**方法2: fetch MCPと併用**
07-10記事で設定したfetch MCPも残しておき、JSONにないフックはfetchでSWELL公式を参照させる。カバレッジと精度の両立ができる。

副業規模では**方法1と方法2のハイブリッド**がよいと感じています。JSONに登録済みのフックは高速・高精度に応答し、未登録のフックは公式ドキュメントをフェッチして補完するイメージです。

### suggest_hooks のスコアリングを改善する

現在の `suggest_hooks` は単純なキーワードマッチングです。より賢くしたい場合、`use_cases` にタグを追加してタグマッチングにする方法があります。

```json
{
  "name": "swell_before_footer",
  "tags": ["footer", "cta", "fixed", "banner", "下部固定"],
  ...
}
```

タグを日本語でも英語でも登録しておくと、`suggest_hooks` でどちらの表現を使っても候補に上がりやすくなります。

### MCPサーバーのデバッグ

MCPサーバーがうまく動かないときのデバッグには、MCPのインスペクターが使えます。

```bash
npx @modelcontextprotocol/inspector node ~/dev/swell-shared/mcp/swell-hooks-mcp/index.js
```

ブラウザでUIが開き、ツールを直接叩いてレスポンスを確認できます。Claude Code経由でなく単体テストできるため、開発中は重宝します。

### 複数SWELL案件での共有

`index.js` のパスを絶対パスで `settings.json` に書いているため、どの案件の `settings.json` から呼んでも同じサーバーが起動します。フックデータは1箇所で管理され、全案件に反映されます。

前回整備した `setup-hooks.sh` を改修して、`settings.json` 生成時に `mcpServers` セクションも自動挿入するよう更新しておくと、新規案件セットアップが1コマンドで完結します。

## まとめ

今回構築したカスタムMCPサーバーの効果をまとめます。

| 比較項目 | fetch MCP（07-10方式） | 独自MCPサーバー（今回） |
|---|---|---|
| 応答速度 | 数秒（ページ取得）| 即座（ローカルJSON） |
| コンテキスト消費 | 大（HTML全体） | 小（必要情報のみ） |
| オフライン対応 | ✗ | ✅ |
| 情報精度 | 高（公式最新） | 高（手動管理・検証済み） |
| フックの意図検索 | ✗ | ✅（suggest_hooks） |
| SWELLバージョン追従 | 自動 | 手動更新が必要 |

fetch MCPはドキュメントを最新状態で参照できる強みがあり、独自MCPはスピードと意図検索の強みがあります。両方を設定して使い分けるのがベストだと感じています。

副業でのWordPress/SWELL案件は「フック名を調べる→コードを書く→動作確認する」のサイクルの繰り返しです。MCPサーバーが手元にあることで、「フック名を調べる」コストがほぼゼロになり、「コードを書く→動作確認する」に集中できるようになりました。

次回は、このMCPサーバーを拡張して**SWELLのカスタマイザー設定キーも検索できるように**する予定です。フックとカスタマイザー設定の両方をClaude Codeが即座に参照できる環境ができると、SWELLカスタマイズの大半をClaude Codeに任せられる状態に近づけると考えています。
