# 技術記事リポジトリ

このリポジトリには、監視ツールとクラウドネイティブに関する技術記事が含まれています。

## 📚 記事一覧

### 3. Sysdig で読み解く AI 時代のクラウドセキュリティ【シリーズ】（2026年版）

「AIを守る（Securing AI）× AIで守る（AI for Security）」の二軸で Sysdig の AI 戦略を読み解く6本シリーズ。

| # | 記事 | テーマ |
|---|---|---|
| #1 | [総論：二軸フレームワーク](./platforms/qiita/ai-security-series/s01-ai-framework-overview.md) | バラバラに見える機能を一本の思想で読む |
| #2 | [AIを守る① - AI Workload Security](./platforms/qiita/ai-security-series/s02-securing-ai-workloads.md) | 34%の GenAI ワークロードが公開状態という現実 |
| #3 | [AIを守る② - MCP Server のリスク](./platforms/qiita/ai-security-series/s03-mcp-server-security.md) | AI 連携ツールがもたらす新攻撃面 |
| #4 | [AIで守る① - Sysdig Sage™ の設計思想](./platforms/qiita/ai-security-series/s04-sysdig-sage-design.md) | チャットボットではなく「判断し・行動する」AI |
| #5 | [AIで守る② - Agentic Cloud Security](./platforms/qiita/ai-security-series/s05-agentic-cloud-security.md) | 週80時間の手作業を分単位に |
| #6 | [総括 - Sysdig の戦略的ポジション](./platforms/qiita/ai-security-series/s06-sysdig-strategy.md) | プラットフォーム一体化の価値 |

- Qiita版: [`platforms/qiita/ai-security-series/`](./platforms/qiita/ai-security-series/)
- タグ: `#CNAPP` `#Sysdig` `#RuntimeSecurity` `#AI` `#CloudSecurity`

### 独立記事：セキュリティ製品は"設計思想"で使え（2026年版）

- **[Hash検知が漏れた事象から考えるCNAPP運用のベストプラクティス](./platforms/qiita/cnapp-hash-detection.md)**
  - 実際のマルウェア検知事象をもとに、Hash/YARA/Falco の多層防御を解説
  - タグ: `#CNAPP` `#Sysdig` `#RuntimeSecurity` `#Falco`

---

### 1. OSS監視ツール徹底比較（2025年版）

- **[OSS監視ツール徹底比較：あなたの環境に合う選択はどれか](./oss-monitoring-tools-comparison.md)**
  - Prometheus、Zabbix、Nagiosなど主要OSS監視ツールの完全比較ガイド
  - 内容:
    - Prometheus + Grafana vs Zabbix の詳細比較
    - データ収集モデル、スケーラビリティ、適用シーン
    - 実践的な選定フローチャート
    - TCO（Total Cost of Ownership）分析
    - ユースケース別推奨構成（4パターン）
    - FAQ（10問）
  - プラットフォーム別バージョン:
    - [Zenn版](./platforms/comparison/oss-monitoring-comparison-zenn.md)
    - [Qiita版](./platforms/comparison/oss-monitoring-comparison-qiita.md)
    - [Medium版](./platforms/comparison/oss-monitoring-comparison-medium.md)

### 2. Sysdig Monitor 徹底解説

- **[Sysdig Monitor をみんなに使って欲しい 徹底解説](./sysdig-monitor-complete-guide.md)**
  - Prometheusの100%マネージドサービスとして、そしてCNAPPプラットフォームとしてのSysdig Monitorを徹底解説
  - 内容:
    - Prometheusとの技術比較（Mermaid図付き）
    - CNAPP統合の価値
    - 実践的なセットアップ手順
    - ユースケース別の活用例
    - コスト比較（ROI計算）
    - トラブルシューティング
    - FAQ（10問）

### プラットフォーム別バージョン

記事を各プラットフォームに最適化したバージョンを用意しています：

- **[Zenn版](./platforms/zenn/sysdig-monitor-guide-zenn.md)**
  - Zenn Front Matter付き
  - 絵文字とメッセージボックス対応
  - タグ: `kubernetes`, `prometheus`, `monitoring`, `sysdig`, `cloudnative`

- **[Qiita版](./platforms/qiita/sysdig-monitor-guide-qiita.md)**
  - Qiitaタグコメント付き
  - 引用ブロックで記事説明

- **[Medium版](./platforms/medium/sysdig-monitor-guide-medium.md)**
  - Mediumスタイルのサブタイトル
  - シンプルなMarkdown

## 📊 記事の特徴

### 視覚的なダイアグラム（Mermaid）

記事には以下のMermaidダイアグラムが含まれています：

1. **Prometheusアーキテクチャ比較図**
   - OSS Prometheus vs Sysdig Monitorの技術スタック比較

2. **CNAPP統合フロー図**
   - MonitorとSecureの統合、Runtime Insightsの流れ

3. **セットアップフローチャート**
   - 初期設定からダッシュボード構築までのステップ

4. **コスト比較チャート**
   - 自前Prometheus vs Sysdig MonitorのROI可視化

### 実践的なコンテンツ

- ✅ 実際に動作するHelmコマンド
- ✅ values.yamlの具体例
- ✅ トラブルシューティング手順
- ✅ 10個のよくある質問と回答
- ✅ 3つのユースケース別活用例

## 🖼️ スクリーンショット

スクリーンショットの要件と撮影ガイドラインは [SCREENSHOT_REQUIREMENTS.md](./SCREENSHOT_REQUIREMENTS.md) を参照してください。

### 優先度HIGHの画像

- Kubernetes Cluster Overviewダッシュボード
- アラート設定画面
- Runtime Insights画面
- 脆弱性スキャン結果
- Threat Detection画面

## 📂 リポジトリ構成

```
articles/
├── README.md                                        # このファイル
├── VERIFICATION_REPORT.md                           # Sysdig記事の技術検証レポート
├── SCREENSHOT_REQUIREMENTS.md                       # スクリーンショット要件
│
├── oss-monitoring-tools-comparison.md               # OSS監視ツール比較記事（メイン）
├── sysdig-monitor-complete-guide.md                 # Sysdig Monitor記事（メイン）
│
├── platforms/                                       # プラットフォーム別バージョン
│   ├── comparison/                                  # OSS監視ツール比較
│   │   ├── oss-monitoring-comparison-zenn.md
│   │   ├── oss-monitoring-comparison-qiita.md
│   │   └── oss-monitoring-comparison-medium.md
│   ├── zenn/                                        # Sysdig記事
│   │   └── sysdig-monitor-guide-zenn.md
│   ├── qiita/
│   │   └── sysdig-monitor-guide-qiita.md
│   └── medium/
│       └── sysdig-monitor-guide-medium.md
│
└── images/                                          # 画像ファイル（今後追加予定）
    └── screenshots/
```

## 🚀 記事の使い方

### 1. そのまま読む

メイン記事 `sysdig-monitor-complete-guide.md` をGitHub上で直接読むことができます。Mermaidダイアグラムも正しく表示されます。

### 2. プラットフォームに投稿

各プラットフォーム用のファイルをコピーして投稿：

- **Zenn**: `platforms/zenn/` のファイルをZenn CLIまたはGitHub連携で公開
- **Qiita**: `platforms/qiita/` のファイルをQiitaエディタにコピー＆ペースト
- **Medium**: `platforms/medium/` のファイルをMediumエディタにインポート

### 3. カスタマイズ

記事をフォークして、自社の環境や経験に合わせてカスタマイズできます。

## 📝 記事について

これらの記事は、実際のPrometheus運用経験とSysdig Monitorの検証結果に基づいて作成されています。

**調査に使用した情報源**:
- Sysdig公式ドキュメント
- Sysdig公式ブログ
- Prometheus公式ドキュメント
- Gartner Market Guide for CNAPP
- BigCommerce導入事例

## 🤝 コントリビューション

誤字脱字の修正、情報の更新、追加のユースケースなど、コントリビューションを歓迎します。

1. このリポジトリをフォーク
2. 変更をコミット
3. プルリクエストを作成

## 📄 ライセンス

Copyright © 2025

本記事は教育目的で自由に利用できますが、商用利用の場合は著者にご連絡ください。

## 📧 お問い合わせ

記事に関する質問や フィードバックは、GitHubのIssueでお願いします。

---

**最終更新**: 2025-10-31
