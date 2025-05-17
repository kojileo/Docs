# Snyk導入の提案書

## 1. Snykとは

- キャッチコピー：AI 時代の開発を迅速に安全に
- 概要：世界の革新的な企業が信頼を寄せるAI を活用した開発者ファーストのセキュリティプラットフォーム
  - [AWS](https://aws.amazon.com/jp/blogs/psa/vulnerability-management-with-snyk/)
  - [Google](https://snyk.io/jp/partners/google/)
  - [salesforce](https://snyk.io/jp/case-studies/salesforce/)
- [機能：Snyk Code](https://snyk.io/jp/product/snyk-code/)
  - 開発者により、開発者のために開発された静的アプリケーションセキュリティテストにより、コードのセキュリティを確保
- [機能：Snyk Open Source](https://snyk.io/jp/product/open-source-security-management/)
  - Snyk Open Source は、業界最先端のセキュリティとアプリケーションのインテリジェンスに裏付けられた高度なソフトウェアコンポジション解析 (SCA) を提供
- [機能：Snyk Container](https://snyk.io/jp/product/container-vulnerability-management/)
  - ソフトウェア開発ライフサイクル (SDLC) 全体で脆弱性を可視化し、優先順位をつけて修正できる、コンテナおよび Kubernetes 向けセキュリティ。
- [機能：Snyk IaC](https://snyk.io/jp/product/infrastructure-as-code-security/)
  - インフラのライフサイクル全体で高セキュリティな開発手法を組み込み、セキュリティ問題の対策を積極的に実施できる可視性と専門知識を開発者に提供して、クラウドで 100% の IaC カバレッジを実現
- [機能：Snyk の API & Web](https://snyk.io/jp/product/dast-api-web/)
  - すべての API、ウェブアプリ、シングルページ アプリケーション (SPA) を検出してセキュリティをテストし、調査結果を修正する方法に関する詳しい手順を入手
- [機能：Snyk AppRisk](https://snyk.io/jp/product/snyk-apprisk/)
  - アプリケーションセキュリティプログラムの構築、管理、拡張
  
## 2. メリット

### 2.1 セキュリティ面
- 既知の脆弱性の早期発見と修正
  - 平均修正時間を72日短縮
  - セキュリティスキャンの速度が2.4倍向上
- Snykによるリアルタイムのセキュリティモニタリングと通知
  - [世界最高峰の脆弱性データベース](https://blog.zzzmisa.com/try-snyk/)による包括的な保護

### 2.2 開発効率
- 開発者フレンドリーなツール
  - コードに対する VS Code の高速なスキャンにより、リアルタイムに結果を検出し、修正案を提案、対策の実施が可能
- 自由で自動化可能なセキュリティチェック
  - コーディング時、プルリクエスト時、特定のブランチマージ時、デプロイ後、定期等様々なタイミング自動で実施可能

### 2.3 コスト削減とROI
- 開発チームの工数削減
  - 手動セキュリティレビューの自動化により、開発者の生産性が向上
  - セキュリティチェックの工数を約70%削減

- コンプライアンス対応の効率化
  - 主要なセキュリティ認証への対応を自動化
    - ISO認証：品質管理システムの国際規格
    - PCI DSS：クレジットカード情報のセキュリティ基準
    - SOC 2 Type 2：情報セキュリティ管理の認証
    - FED Ramp：米国政府向けクラウドサービスのセキュリティ認証
  - 監査対応の工数削減とリスク低減

## 3. 効果検証方法

### 3.1 現状分析（ベースライン測定）
- プロジェクトの現状把握
  - 依存関係の管理方法の確認
  - 現在のセキュリティ対策の有無

- 開発プロセスの現状
  - セキュリティレビューの実施状況
  - 開発者のセキュリティ意識調査

### 3.2 パイロット導入期間（3ヶ月）の効果検証
- 脆弱性の初回スキャン結果
  - 検出された脆弱性の数と重大度

- 開発プロセスへの影響
  - 開発者の作業効率への影響
  - 開発者からのフィードバック収集

### 3.3 本格導入後（6ヶ月）の効果検証
- セキュリティ面での効果
  - 新規脆弱性の検出数推移
  - 脆弱性の修正状況
  - セキュリティリスクの可視化状況
  - セキュリティインシデントの発生状況

- 開発効率面での効果
  - 開発者の生産性への影響

- コスト面での効果
  - セキュリティ対策の工数削減

### 3.4 効果検証の実施方法
- 定期的なレビュー
  - 週次：脆弱性検出数、修正状況の確認
  - 月次：工数削減効果の計測
  - 四半期：セキュリティ体制の改善状況確認

- 測定指標（KPI）の設定
  - 脆弱性検出数
  - 脆弱性修正率
  - 開発者満足度

- 効果検証レポート
  - 定期的なレポート作成
  - 改善点の特定と対策
  - 成功事例の共有

### 3.5 期待される効果
- 短期的な効果（3ヶ月）
  - 既存の脆弱性の特定
  - セキュリティリスクの可視化
  - 開発プロセスへのセキュリティチェックの組み込み

- 中長期的な効果（6ヶ月）
  - セキュリティ体制の確立
  - 開発効率の向上
  - セキュリティインシデントの予防

### 4 参考
- [Snyk公式サイト](https://snyk.io/jp/?utm_medium=paid-search&utm_source=bing&utm_campaign=brand_jp&utm_content=br_bmm&ad_id=84662967251362&adgroup_id=1354600910864486&campaign_id=532710637&utm_term=snyk&match_type=p&msclkid=de6a12443dc7118f43387a6c56c709cb)
- [2023 年の Snyk カスタマーバリュー (顧客価値) 調査](https://snyk.io/jp/reports/customer-value-study/)
- [妥協のないコンプライアンス](https://snyk.io/jp/platform/compliance/)
- [DevSecOps超入門](https://qiita.com/Halkichi_web/items/dfdc4ed8f4af69eea183)
- [Snyk の記事一覧](https://dev.classmethod.jp/articles/snyk-shift-left-202307/)


