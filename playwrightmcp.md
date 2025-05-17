# Playwright Model Context Protocol (MCP) Server 解説

## Model Context Protocol (MCP) とは

Model Context Protocol (MCP) は、大規模言語モデル (LLM) アプリケーションが外部のデータソースやツールとシームレスに統合できるようにするためのオープンプロトコルです。AIを搭載したIDEの構築、チャットインターフェースの強化、カスタムAIワークフローの作成など、MCPはLLMが必要なコンテキストに接続するための標準化された方法を提供します。

MCP は、主に以下の課題を解決することを目的としています:

*   **コンテキストの断片化:** LLMフレームワークごとにコンテキストデータの扱い方が異なり、インテグレーションが複雑になる問題。
*   **M×Nのインテグレーション問題:** 多数のAIアプリケーションとツール/データソース間の連携コスト。
*   **動的なコンテキストアクセス:** LLMがリアルタイム情報を外部システムから取得する際の困難さ。

## Playwright MCP Server とは

Playwright MCP Server は、Playwright を使用してブラウザ自動化機能を提供する Model Context Protocol (MCP) サーバーです。このサーバーにより、LLM はスクリーンショットや視覚的に調整されたモデルに頼ることなく、構造化されたアクセシビリティスナップショットを通じてウェブページと対話できます。

### 主な特徴

*   **高速かつ軽量:** ピクセルベースの入力ではなく、Playwright のアクセシビリティツリーを使用します。
*   **LLMフレンドリー:** 視覚モデルを必要とせず、構造化データのみで動作します。
*   **決定論的なツール適用:** スクリーンショットベースのアプローチで一般的な曖昧さを回避します。

## Playwright MCP Server のセットアップ

Playwright MCP Server は、通常、MCPクライアント（VS Code、Cursor、Claude Desktopなど）を通じてインストールおよび設定されます。

一般的な設定例 (クライアントの設定ファイル内):

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": [
        "@playwright/mcp@latest"
      ]
    }
  }
}
```

### 要件

*   Node.js 18 以降
*   VS Code、Cursor、Windsurf、Claude Desktop、またはその他のMCPクライアント

### 設定オプション

Playwright MCP Server は、起動時の引数を通じて様々な設定が可能です。主なオプションには以下のようなものがあります（`npx @playwright/mcp@latest --help` で確認可能）：

*   `--browser <browser>`: 使用するブラウザ (chrome, firefox, webkit, msedge) を指定します。
*   `--headless`: ブラウザをヘッドレスモードで実行します。
*   `--port <port>`: SSEトランスポートのリッスンポートを指定します。
*   `--user-data-dir <path>`: ユーザーデータディレクトリのパスを指定します。
*   `--vision`: デフォルトのアクセシビリティスナップショットの代わりにスクリーンショットを使用するVisionモードで実行します。
*   その他、プロキシ設定、許可/ブロックするオリジン、エミュレートするデバイスなどを指定できます。

詳細な設定方法やオプションについては、公式の `microsoft/playwright-mcp` GitHub リポジトリの README を参照してください。

## Playwright MCP Server が提供するツール

Playwright MCP Server は、LLM がブラウザを操作するための様々なツールを提供します。これらのツールは、デフォルトの「スナップショットモード」と、`--vision` フラグで有効になる「Visionモード」で利用可能です。

以下は提供される主なツール（機能）の例です。

**インタラクション系:**

*   `browser_snapshot`: 現在のページのアクセシビリティスナップショットを取得します。
*   `browser_click`: ウェブページ上の要素をクリックします。
*   `browser_type`: 編集可能な要素にテキストを入力します。
*   `browser_select_option`: ドロップダウンのオプションを選択します。
*   `browser_press_key`: キーボードのキーを押します。
*   `browser_drag`: ある要素から別の要素へドラッグ＆ドロップ操作を行います。
*   `browser_hover`: 要素の上にマウスカーソルを合わせます。

**ナビゲーション系:**

*   `browser_navigate`: 指定されたURLに移動します。
*   `browser_navigate_back`: 前のページに戻ります。
*   `browser_navigate_forward`: 次のページに進みます。

**リソース取得・保存系:**

*   `browser_take_screenshot`: 現在のページのスクリーンショットを撮ります。
*   `browser_pdf_save`: ページをPDFとして保存します。
*   `browser_network_requests`: ページ読み込み以降のネットワークリクエストのリストを取得します。
*   `browser_console_messages`: コンソールメッセージを取得します。

**ユーティリティ系:**

*   `browser_install`: 設定で指定されたブラウザをインストールします。
*   `browser_close`: ブラウザを閉じます。
*   `browser_resize`: ブラウザウィンドウのサイズを変更します。

**タブ操作系:**

*   `browser_tab_list`: ブラウザのタブを一覧表示します。
*   `browser_tab_new`: 新しいタブを開きます。
*   `browser_tab_select`: 指定したインデックスのタブを選択します。
*   `browser_tab_close`: タブを閉じます。

**テスト生成:**

*   `browser_generate_playwright_test`: 与えられたシナリオに基づいてPlaywrightのテストを生成します。

**Visionモード特有のツール:**

*   `browser_screen_capture`: スクリーンショットを取得します。
*   `browser_screen_move_mouse`: 指定された座標にマウスを移動します。
*   `browser_screen_click`: 指定された座標でクリックします。

各ツールの詳細なパラメータについては、`microsoft/playwright-mcp` のドキュメントを参照してください。

## まとめ

Playwright MCP Server は、Model Context Protocol を通じて、LLM が Playwright の強力なブラウザ自動化機能を利用できるようにする重要なコンポーネントです。これにより、LLM はより効率的かつ正確にウェブページと対話し、情報収集やタスク実行を行うことが可能になります。
