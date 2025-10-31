# Sysdig関連記事

このリポジトリには、Sysdigに関する技術記事が含まれています。

## 📚 記事一覧

### メイン記事

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
├── README.md                              # このファイル
├── sysdig-monitor-complete-guide.md       # メイン記事
├── SCREENSHOT_REQUIREMENTS.md             # スクリーンショット要件
├── platforms/                             # プラットフォーム別バージョン
│   ├── zenn/
│   │   └── sysdig-monitor-guide-zenn.md
│   ├── qiita/
│   │   └── sysdig-monitor-guide-qiita.md
│   └── medium/
│       └── sysdig-monitor-guide-medium.md
└── images/                                # 画像ファイル（今後追加予定）
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
