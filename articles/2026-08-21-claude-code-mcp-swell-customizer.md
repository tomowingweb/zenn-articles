---
title: "SWELLカスタマイザー設定キーをMCPサーバーに追加してClaude Codeから即検索する"
emoji: "🎛️"
type: "tech"
topics: ["wordpress", "claudecode", "swell", "mcp", "php"]
published: false
---

## はじめに

[前回の記事](https://zenn.dev/tomowingweb/articles/2026-08-14-claude-code-custom-mcp-swell-hooks)では、SWELLのフック情報をローカルJSONで持つ独自MCPサーバーを構築しました。フック名をキーワード検索したり、「〇〇したい」という意図からフックを提案させたりする仕組みで、実際にSWELL案件の実装速度が上がりました。

前回の末尾に「次は**カスタマイザー設定キー**も同じMCPサーバーで検索できるようにしたい」と書きました。今回はその実践です。

SWELLのカスタマイザーは設定項目が多く、`get_theme_mod()` で取得するときに**設定キー名を忘れる**ことがしばしばあります。たとえば「ブログカードのサムネイル比率を変えるキーは何だっけ」「ヘッダーの高さを取得するキーは？」といった確認が手元のメモや公式ドキュメントを開くたびに発生します。

今回は `swell-hooks-mcp` に `search_customizer`・`get_customizer` の2つのツールを追加し、カスタマイザー設定キーをClaude Codeから即座に引ける状態にします。

## 環境

- WordPress 6.7.x + SWELLテーマ（最新）
- MAMP Pro（ローカル開発環境 / Mac）
- Claude Code（CLI）v1.x
- Node.js 20.x
- `@modelcontextprotocol/sdk` v1.x（前回から継続）

## 設計方針

前回は「フックデータ（`swell-hooks.json`）」だけを持っていました。今回は「カスタマイザーデータ（`swell-customizer.json`）」を並列で追加し、同じMCPサーバーから提供します。ツール数が増えるだけで、サーバー全体の構造は変わりません。

```
swell-hooks-mcp/
├── index.js              ← ツール追加（search_customizer / get_customizer）
└── data/
    ├── swell-hooks.json  ← 前回から継続
    └── swell-customizer.json  ← 今回追加
```

## 実装内容

### 1. カスタマイザーデータJSONの作成

`data/swell-customizer.json` を新規作成します。SWELL公式ドキュメントおよび実際のカスタマイザーUIを確認しながら、よく使う設定キーをまとめます。

```json
[
  {
    "key": "logo_image",
    "area": "header",
    "label": "ロゴ画像",
    "description": "ヘッダーに表示するロゴ画像のアタッチメントIDを返す。0の場合はロゴ未設定。",
    "type": "image_id",
    "default": 0,
    "example": "$logo_id = get_theme_mod('logo_image', 0);\nif ($logo_id) {\n    echo wp_get_attachment_image($logo_id, 'full', false, ['class' => 'site-logo']);\n}",
    "use_cases": ["ロゴ画像の出力", "ロゴの有無による条件分岐"]
  },
  {
    "key": "logo_height",
    "area": "header",
    "label": "ロゴ高さ（PC）",
    "description": "PCヘッダーでのロゴ最大高さ（px）。デフォルト60。",
    "type": "integer",
    "default": 60,
    "example": "$height = get_theme_mod('logo_height', 60);\necho '<style>.site-logo { max-height: ' . intval($height) . 'px; }</style>';",
    "use_cases": ["ロゴ高さのダイナミックCSS出力"]
  },
  {
    "key": "header_height",
    "area": "header",
    "label": "ヘッダー高さ（PC）",
    "description": "PCヘッダーの高さ（px）。デフォルト80。固定ヘッダーのpadding計算に使う。",
    "type": "integer",
    "default": 80,
    "example": "$header_h = get_theme_mod('header_height', 80);\necho '<style>body { scroll-padding-top: ' . intval($header_h) . 'px; }</style>';",
    "use_cases": ["スクロールオフセットの計算", "固定ヘッダー分のpadding調整"]
  },
  {
    "key": "header_bg_color",
    "area": "header",
    "label": "ヘッダー背景色",
    "description": "ヘッダー背景のカラーコード（# 付き）。デフォルト #ffffff。",
    "type": "color",
    "default": "#ffffff",
    "example": "$color = get_theme_mod('header_bg_color', '#ffffff');\necho '<style>.l-header { background-color: ' . esc_attr($color) . '; }</style>';",
    "use_cases": ["ヘッダー背景色のCSS出力", "スクロール前後で色を変えるJS実装時の参照"]
  },
  {
    "key": "footer_bg_color",
    "area": "footer",
    "label": "フッター背景色",
    "description": "フッター背景のカラーコード。デフォルト #222222。",
    "type": "color",
    "default": "#222222",
    "example": "$color = get_theme_mod('footer_bg_color', '#222222');\necho '<style>.l-footer { background-color: ' . esc_attr($color) . '; }</style>';",
    "use_cases": ["フッター背景色の動的変更"]
  },
  {
    "key": "main_color",
    "area": "color",
    "label": "メインカラー",
    "description": "テーマ全体のメインカラー。ボタン・リンク・見出しのアクセントカラーとして使われる。",
    "type": "color",
    "default": "#1DA1F2",
    "example": "$main = get_theme_mod('main_color', '#1DA1F2');\n// JavaScriptへ渡す例\nwp_add_inline_script('my-script', 'const THEME_COLOR = \"' . esc_js($main) . '\";');",
    "use_cases": ["メインカラーをJSやCSSカスタムプロパティへ渡す", "動的なブランドカラー適用"]
  },
  {
    "key": "base_font_size",
    "area": "typography",
    "label": "ベースフォントサイズ",
    "description": "サイト全体の基本フォントサイズ（px）。デフォルト16。",
    "type": "integer",
    "default": 16,
    "example": "$size = get_theme_mod('base_font_size', 16);\necho '<style>:root { font-size: ' . intval($size) . 'px; }</style>';",
    "use_cases": ["rem計算基準のフォントサイズ適用"]
  },
  {
    "key": "content_width",
    "area": "layout",
    "label": "コンテンツ幅",
    "description": "記事本文エリアの最大幅（px）。デフォルト720。",
    "type": "integer",
    "default": 720,
    "example": "$w = get_theme_mod('content_width', 720);\necho '<style>.l-contents { max-width: ' . intval($w) . 'px; }</style>';",
    "use_cases": ["記事幅を動的にCSS適用", "読みやすさのレイアウト調整"]
  },
  {
    "key": "sidebar_position",
    "area": "layout",
    "label": "サイドバー位置",
    "description": "サイドバーを「right」「left」「none」から選択。デフォルト right。",
    "type": "select",
    "default": "right",
    "options": ["right", "left", "none"],
    "example": "$pos = get_theme_mod('sidebar_position', 'right');\nif ($pos === 'none') {\n    // サイドバーなし時の処理\n}",
    "use_cases": ["サイドバー有無による条件分岐", "レイアウト切り替えロジック"]
  },
  {
    "key": "blog_card_thumbnail_ratio",
    "area": "card",
    "label": "ブログカードサムネイル比率",
    "description": "ブログカード（リンクカード）のサムネイル縦横比。デフォルト 16:9。",
    "type": "select",
    "default": "16:9",
    "options": ["16:9", "4:3", "1:1"],
    "example": "$ratio = get_theme_mod('blog_card_thumbnail_ratio', '16:9');\n$ratio_class = str_replace(':', '-', $ratio); // 例: '16-9'\necho '<div class=\"thumb ratio-' . esc_attr($ratio_class) . '\">';",
    "use_cases": ["カードのサムネイル比率を動的なクラスで制御"]
  },
  {
    "key": "show_breadcrumb",
    "area": "navigation",
    "label": "パンくずリスト表示",
    "description": "パンくずリストの表示可否。1で表示、0で非表示。デフォルト 1。",
    "type": "boolean",
    "default": 1,
    "example": "if (get_theme_mod('show_breadcrumb', 1)) {\n    swell_breadcrumb(); // SWELLのパンくず出力関数\n}",
    "use_cases": ["パンくずリストの条件出力"]
  },
  {
    "key": "toc_enabled",
    "area": "content",
    "label": "目次の自動生成",
    "description": "記事内の目次を自動生成するかの設定。1で有効。デフォルト 1。",
    "type": "boolean",
    "default": 1,
    "example": "if (!get_theme_mod('toc_enabled', 1)) {\n    remove_filter('the_content', 'swell_insert_toc');\n}",
    "use_cases": ["特定条件での目次非表示制御"]
  },
  {
    "key": "sns_share_buttons",
    "area": "sns",
    "label": "SNSシェアボタン設定",
    "description": "表示するSNSシェアボタンの配列。デフォルト ['twitter', 'facebook', 'line']。",
    "type": "array",
    "default": ["twitter", "facebook", "line"],
    "example": "$buttons = get_theme_mod('sns_share_buttons', ['twitter', 'facebook', 'line']);\nif (in_array('twitter', (array)$buttons)) {\n    // X（Twitter）ボタン出力\n}",
    "use_cases": ["カスタマイザーで選択されたSNSボタンだけ出力する"]
  },
  {
    "key": "google_analytics_id",
    "area": "analytics",
    "label": "Google Analytics 計測ID",
    "description": "GA4の計測ID（G-XXXXXXXXXX形式）。空文字の場合はトラッキングコードを出力しない。",
    "type": "string",
    "default": "",
    "example": "$ga_id = get_theme_mod('google_analytics_id', '');\nif ($ga_id) {\n    // GTMまたはgtag.jsを出力\n}",
    "use_cases": ["GAトラッキングコードの動的出力", "測定IDの有無で条件分岐"]
  },
  {
    "key": "404_content",
    "area": "content",
    "label": "404ページのコンテンツ",
    "description": "404エラーページに表示するテキスト。HTMLタグ使用可。デフォルトはSWELL定義のメッセージ。",
    "type": "textarea",
    "default": "",
    "example": "$msg = get_theme_mod('404_content', '');\nif ($msg) {\n    echo wp_kses_post($msg);\n}",
    "use_cases": ["404ページのカスタムメッセージ表示"]
  }
]
```

### 2. index.jsへのツール追加

前回の `index.js` に `search_customizer` と `get_customizer` の2ツールを追加します。差分のみ示します。

#### ListToolsRequestSchema ハンドラーへの追加

```javascript
// 既存のフックツール3つの後に追加
{
  name: 'search_customizer',
  description: 'キーワードでSWELLのカスタマイザー設定キーを検索する。設定キー名・ラベル・エリア・説明を対象に部分一致で検索。',
  inputSchema: {
    type: 'object',
    properties: {
      query: {
        type: 'string',
        description: '検索キーワード（日本語可）。例: "ロゴ", "color", "font"'
      }
    },
    required: ['query']
  }
},
{
  name: 'get_customizer',
  description: 'カスタマイザー設定キーを指定して詳細情報（型・デフォルト値・PHPサンプルコード）を取得する。',
  inputSchema: {
    type: 'object',
    properties: {
      key: {
        type: 'string',
        description: '設定キー名。例: "main_color"'
      }
    },
    required: ['key']
  }
}
```

#### CallToolRequestSchema ハンドラーへの追加

`index.js` のデータ読み込み部分と、ツール実装部分を以下のように更新します。

```javascript
'use strict';

const { Server } = require('@modelcontextprotocol/sdk/server/index.js');
const { StdioServerTransport } = require('@modelcontextprotocol/sdk/server/stdio.js');
const { CallToolRequestSchema, ListToolsRequestSchema } = require('@modelcontextprotocol/sdk/types.js');
const path = require('path');
const fs = require('fs');

// データ読み込み（フック + カスタマイザー）
const DATA_DIR = path.join(__dirname, 'data');
const hooks = JSON.parse(fs.readFileSync(path.join(DATA_DIR, 'swell-hooks.json'), 'utf8'));
const customizer = JSON.parse(fs.readFileSync(path.join(DATA_DIR, 'swell-customizer.json'), 'utf8'));

const server = new Server(
  { name: 'swell-hooks-mcp', version: '1.1.0' },
  { capabilities: { tools: {} } }
);

// （ListToolsRequestSchema ハンドラーは上記のツール定義を含む形で更新）

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  // ── フック系ツール（前回から変更なし）──────────────────
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
    const scored = hooks.map(h => {
      let score = 0;
      const text = (h.description + ' ' + h.use_cases.join(' ')).toLowerCase();
      intent.split(/[\s　、。]+/).forEach(word => {
        if (word.length > 1 && text.includes(word)) score += 2;
      });
      if (intent.includes(h.area)) score += 3;
      return { hook: h, score };
    });
    const suggestions = scored.filter(s => s.score > 0).sort((a, b) => b.score - a.score).slice(0, 5);
    if (suggestions.length === 0) {
      return { content: [{ type: 'text', text: `"${args.intent}" に対応するフックが見つかりませんでした。` }] };
    }
    const text = suggestions.map(({ hook: h }) =>
      `### ${h.name} (${h.type})\n${h.description}\n\`\`\`php\n${h.example}\n\`\`\``
    ).join('\n\n');
    return { content: [{ type: 'text', text: `**"${args.intent}" に適したフック候補 (${suggestions.length}件)**\n\n${text}` }] };
  }

  // ── カスタマイザー系ツール（今回追加）─────────────────

  if (name === 'search_customizer') {
    const q = (args.query || '').toLowerCase();
    const results = customizer.filter(c =>
      c.key.toLowerCase().includes(q) ||
      c.label.toLowerCase().includes(q) ||
      c.description.toLowerCase().includes(q) ||
      c.area.toLowerCase().includes(q) ||
      (c.use_cases || []).some(uc => uc.toLowerCase().includes(q))
    );
    if (results.length === 0) {
      return { content: [{ type: 'text', text: `"${args.query}" に一致するカスタマイザー設定は見つかりませんでした。` }] };
    }
    const text = results.map(c =>
      `### ${c.key}（${c.label}）\n- **エリア**: ${c.area}\n- **型**: ${c.type}  デフォルト: \`${JSON.stringify(c.default)}\`\n- ${c.description}`
    ).join('\n\n');
    return { content: [{ type: 'text', text: `**カスタマイザー検索結果: ${results.length}件**\n\n${text}` }] };
  }

  if (name === 'get_customizer') {
    const item = customizer.find(c => c.key === args.key);
    if (!item) {
      return { content: [{ type: 'text', text: `カスタマイザー設定 "${args.key}" は見つかりませんでした。search_customizer で検索してください。` }] };
    }
    const lines = [
      `## ${item.key}（${item.label}）`,
      `- **エリア**: ${item.area}`,
      `- **型**: ${item.type}`,
      `- **デフォルト値**: \`${JSON.stringify(item.default)}\``,
      `- **説明**: ${item.description}`,
    ];
    if (item.options) {
      lines.push(`- **選択肢**: ${item.options.join(' / ')}`);
    }
    if (item.use_cases) {
      lines.push(`- **ユースケース**: ${item.use_cases.join('、')}`);
    }
    lines.push(`\n**PHPサンプル**:\n\`\`\`php\n${item.example}\n\`\`\``);
    return { content: [{ type: 'text', text: lines.join('\n') }] };
  }

  return { content: [{ type: 'text', text: `不明なツール: ${name}` }], isError: true };
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  process.stderr.write('swell-hooks-mcp v1.1.0: 起動しました（フック + カスタマイザー対応）\n');
}

main().catch(err => {
  process.stderr.write(`Fatal: ${err.message}\n`);
  process.exit(1);
});
```

### 3. バージョンを 1.1.0 に更新

`package.json` の `version` を `"1.1.0"` にしておきます。Claude Codeの `/mcp` ステータス画面でバージョンが表示されるため、更新後に「v1.1.0」に変わっていれば反映確認の目安になります。

```json
{
  "name": "swell-hooks-mcp",
  "version": "1.1.0",
  ...
}
```

### 4. 動作確認

Claude Codeを再起動して `/mcp` を実行し、サーバーが `connected` 状態なのを確認します。その後、以下のように自然言語で問い合わせできるようになります。

```
「SWELLでメインカラーを取得するカスタマイザーのキーは？」
```

`search_customizer(query="メインカラー")` が呼ばれ、`main_color` の詳細とPHPサンプルが返ります。

```
「get_theme_mod でヘッダー高さを取得したい」
```

`search_customizer(query="ヘッダー高さ")` → `header_height` がヒットし、`get_customizer(key="header_height")` でサンプルコードまで取得できます。

## ポイント

### データファイルを分けた理由

フックデータとカスタマイザーデータは性質が異なります。フックは「PHPの実行タイミングを制御する」もの、カスタマイザーは「設定値を読み出す」ものです。1つのJSONに混在させると検索結果が混乱するため、ファイルを分けてツールごとに明示的に使い分けています。

「フックもカスタマイザーも一緒にキーワード検索したい」という場面は実際には少なく、「今これはフックの問題か設定値の問題か」を判断してから検索する方が精度の高い結果を得やすいです。

### `get_theme_mod` のデフォルト値をJSONに持つメリット

`get_theme_mod('logo_height', 60)` のように第2引数でデフォルト値を渡すのがWordPressの作法ですが、このデフォルト値をJSONに記録しておくことで次のメリットがあります。

- Claude Codeに「このキーのデフォルトは何？」と聞いたとき、即答できる
- 複数案件でデフォルト値を統一するときのリファレンスになる
- カスタマイザーUIで変更された値と「変更なし＝デフォルト」を区別するロジックを書くときに参照できる

SWELLのソースコード（`/wp-content/themes/swell/`）を `grep -r 'get_theme_mod'` すると実際のデフォルト値を確認できますが、案件ごとに毎回調べるのは手間がかかります。JSONへの記録は一度だけのコストで、何度でも使えます。

### カスタマイザー設定のメンテナンス

SWELLのバージョンアップで設定キーが追加・廃止されることがあります。前回のフックデータと同様、以下の2つのメンテナンス方針を組み合わせています。

1. **使ったら追加する**：案件で `get_theme_mod()` を使ったキーをその都度JSONに追加していく
2. **SWELLリリースノートを参照する**：大型アップデート時にカスタマイザーの変更点を確認し、廃止キーを削除・新規キーを追記する

完全な一覧を最初から作ろうとすると工数がかかりすぎます。実務で使った記録として蓄積していく方が、精度の高いデータが育ちます。

### フックとカスタマイザーを組み合わせて使う例

MCPサーバーにフックとカスタマイザーの両方が乗ったことで、Claude Codeへの指示がより自然になります。

```
「サイトのメインカラーを読み取って、ヘッダー背景に動的に適用したい。
どのカスタマイザーキーを使えばいいか調べて、
それをSWELLのどのフックで出力すれば良い？」
```

この1文の指示でClaude Codeは：

1. `search_customizer("メインカラー")` → `main_color` を発見
2. `search_hooks("ヘッダー")` → `swell_before_header` 等を発見
3. 両方を組み合わせた実装コードを提案する

という流れが自動的に走ります。2つのツールが同じサーバーにあることで、Claude Codeがツールを連続して呼び出しやすくなっています。

### 型情報（`type`フィールド）の活用

JSONの各エントリに `"type": "color"` や `"type": "boolean"` を持たせることで、Claude Codeが値の扱い方を判断できます。

たとえば `color` 型のキーを使うコードを書いてもらうと、Claude Codeは `esc_attr()` でのサニタイズを自動で入れてくれる傾向があります。`boolean` 型であれば `(bool)get_theme_mod(...)` の明示的なキャストを提案してくれます。

型情報はドキュメントとしての役割だけでなく、Claude Codeへのヒントとして機能します。

### suggest_hooks との橋渡し

「〇〇したい」という意図から最適な**フック**を提案する `suggest_hooks` に対応する形で、「〇〇を取得したい」という意図から最適な**カスタマイザーキー**を提案する `suggest_customizer` を今後追加する予定です。

現時点では `search_customizer` でキーワード検索するだけですが、フックの `suggest_hooks` と同様にスコアリングロジックを実装すると、より曖昧な質問でも適切なキーを提案できるようになります。

## まとめ

今回の拡張で `swell-hooks-mcp` が対応できるようになったことをまとめます。

| ツール | 用途 |
|---|---|
| `search_hooks` | キーワードでSWELLフックを検索 |
| `get_hook` | フック詳細とPHPサンプルを取得 |
| `suggest_hooks` | 「〇〇したい」からフックを提案 |
| `search_customizer` | キーワードでカスタマイザー設定キーを検索 |
| `get_customizer` | 設定キーの詳細・デフォルト値・PHPサンプルを取得 |

前回のフックMCPが「どこに処理を書くか」を即答してくれる道具だったとすれば、今回の追加で「どの値を読み出すか」も即答できるようになりました。副業のSWELL案件では実装の多くがこの2つ（フック選定 + カスタマイザー値取得）の組み合わせで成り立っているため、両方をClaude Codeがオフラインで即座に答えられる環境は実用上の差が大きいです。

実際に今週の案件でヘッダーカスタマイズを行った際、フックとカスタマイザーキーの確認でドキュメントを一度も開かずに実装を完了できました。ローカルMCPサーバーへの投資対効果は高いと感じています。

次回は、MCPサーバーに蓄積されてきたデータ（フック + カスタマイザー）を活用して、**Claude Codeに子テーマのコードレビューを自動で走らせる**仕組みを試す予定です。実装後のコードをClaude Codeが自走してレビューし、SWELLとの互換性チェックや改善提案をまとめてくれるフローを構築します。
