# Playwright Model Context Protocol (MCP) Server 解説

昨今世間で話題になりつつあるMCPというAI技術の紹介とそのMCPとブラウザテスト自動化のためのフレームワークを組み合わせ、自然言語によってブラウザの自動操作を実現できるPlaywright MCP Serverを紹介できればと思っています。

## Model Context Protocol (MCP) とは

MCPサーバー（Model Context Protocol サーバー）は、AIの機能を大幅に拡張するための標準化されたプロトコルです。「AIのUSBポート」とも例えられ、AIが外部データやツールと安全かつ効率的に連携できるようにする仕組みです。

## なぜMCPが必要なのか

従来のAIモデルには大きな制約がありました。
- インターネット検索はできても、自分のPC内のファイルは読めない
- テキスト生成が得意でも、当たり前ではあるが、リアルタイムデータや他便利ツールから孤立している

MCPはこれらの制約を取り除き、AIが外部世界と効率的に相互作用できる「橋渡し」の役割を果たします。

## MCP概要図

https://qiita.com/Takenoko4594/items/cc36ca3043f11ca175c1

## Playwright MCP Server とは

Playwright MCP Server は、Playwright を使用してブラウザ自動化機能を提供する Model Context Protocol (MCP) サーバーです。このサーバーにより、playwrightのコードを通じてではなく、自然言語によりウェブページと対話できます。


## Playwright MCP Server のデモ

## まとめ
この技術により、テストシナリオがUIに入出力しやすい形になっていれば、その内容をそのまま入力させてE2Eテストが実行できるのでは？

## LangChain
- LLMを用いたアプリケーション開発を効率的に行うためのpythonライブラリ。これを活用してMCPクライアントを実現。(これは、Github Copilotのチャットでもおそらく代替可能)
- Playwright MCPはMicrosoftが開発したオープンソースのMCP。LLMがWebブラウザを操作できるように設計されています。

## 参考資料
- https://zenn.dev/asap/articles/59b8dd06d44754