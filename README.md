# Sysdig 技術記事リポジトリ

Sysdig / クラウドネイティブセキュリティ / 監視に関する技術記事を一元管理するリポジトリです。

---

## 📂 ディレクトリ構成

```
articles/
├── README.md
├── SCREENSHOT_REQUIREMENTS.md
├── VERIFICATION_REPORT.md
│
├── *.md                        # ドラフト・メイン原稿
├── series/                     # AI × Sysdig セキュリティシリーズ
│
└── platforms/
    ├── qiita/                  # Qiita投稿用（フロントマター付き）
    │   └── ai-security-series/ # AIセキュリティシリーズ（Qiita版）
    ├── zenn/                   # Zenn投稿用
    ├── comparison/             # OSS比較記事（各プラットフォーム版）
    └── medium/                 # Medium投稿用
```

---

## 📚 記事一覧

### 🔐 Lambda / サーバレスセキュリティ

| ファイル | 内容 |
|----------|------|
| [lambda-serverless-security.md](./lambda-serverless-security.md) | Lambdaは果たして安全か？― サーバレスの"見えないリスク"を解剖する |

### 🛡️ Runtime Security / CNAPP

| ファイル | 内容 |
|----------|------|
| [runtime-native-observability.md](./runtime-native-observability.md) | Runtime-native Security の思想と実装 |
| [cnapp-hash-detection.md](./cnapp-hash-detection.md) | Hash検知が漏れた事象から考えるCNAPP運用のベストプラクティス |
| [cnapp-hash-detection-qiita.md](./cnapp-hash-detection-qiita.md) | 同上（Qiita投稿用） |
| [ids_vs_sysdig_article.md](./ids_vs_sysdig_article.md) | IDS vs Sysdig ― 検知思想の比較 |
| [sysdig_apm_article.md](./sysdig_apm_article.md) | APMではなぜ守れないのか |
| [sysdig-ai-strategy-qiita.md](./sysdig-ai-strategy-qiita.md) | Sysdig AI戦略を読み解く（Qiita版） |

### 🔍 Sysdig 製品解説

| ファイル | 内容 |
|----------|------|
| [sysdig-monitor-complete-guide.md](./sysdig-monitor-complete-guide.md) | Sysdig Monitor 徹底解説（Prometheus比較・CNAPP統合） |
| [sysdig-cli-scanner-demo.md](./sysdig-cli-scanner-demo.md) | Sysdig CLI Scanner デモ解説 |
| [Sysdig_CLI_Scanner_挙動検証レポート.md](./Sysdig_CLI_Scanner_挙動検証レポート.md) | CLI Scanner 挙動検証レポート |
| [sysdig-vulnerability-scanning-comprehensive-guide.md](./sysdig-vulnerability-scanning-comprehensive-guide.md) | 脆弱性スキャン完全ガイド |
| [sysdig-malware-test-guide.md](./sysdig-malware-test-guide.md) | マルウェア検知テストガイド |
| [sysdig-malware-verification-manual.md](./sysdig-malware-verification-manual.md) | マルウェア検知 検証マニュアル |
| [stratoshark-qiita-article.md](./stratoshark-qiita-article.md) | Stratoshark 解説記事（Qiita版） |

### ☸️ Kubernetes / インフラ

| ファイル | 内容 |
|----------|------|
| [kubernetes-security-deep-dive.md](./kubernetes-security-deep-dive.md) | Kubernetes セキュリティ詳解 |
| [kubernetes-prometheus-falco-guide.md](./kubernetes-prometheus-falco-guide.md) | Kubernetes × Prometheus × Falco 実践ガイド |
| [kubernetes-japan-market-future.md](./kubernetes-japan-market-future.md) | Kubernetes 日本市場の展望 |
| [oss-monitoring-tools-comparison.md](./oss-monitoring-tools-comparison.md) | OSS監視ツール徹底比較（Prometheus/Zabbix/Nagios等） |

### 🏗️ プラットフォームエンジニアリング / OSS

| ファイル | 内容 |
|----------|------|
| [platform-engineering.md](./platform-engineering.md) | Platform Engineering の現在地 |
| [iac-imperative-vs-declarative.md](./iac-imperative-vs-declarative.md) | IaC：命令型 vs 宣言型 |
| [oss-user-first-presentation.md](./oss-user-first-presentation.md) | OSSをユーザーファーストで使う |

---

## 📖 シリーズ記事

### Sysdig で読み解く AI 時代のクラウドセキュリティ（2026年版）

「AIを守る（Securing AI）× AIで守る（AI for Security）」の二軸で Sysdig の AI 戦略を読み解く6本シリーズ。

| # | ファイル | テーマ |
|---|----------|--------|
| 1 | [s01-ai-framework-overview.md](./series/s01-ai-framework-overview.md) | 総論：二軸フレームワーク |
| 2 | [s02-securing-ai-workloads.md](./series/s02-securing-ai-workloads.md) | AIを守る① - AI Workload Security |
| 3 | [s03-mcp-server-security.md](./series/s03-mcp-server-security.md) | AIを守る② - MCP Server のリスク |
| 4 | [s04-sysdig-sage-design.md](./series/s04-sysdig-sage-design.md) | AIで守る① - Sysdig Sage™ の設計思想 |
| 5 | [s05-agentic-cloud-security.md](./series/s05-agentic-cloud-security.md) | AIで守る② - Agentic Cloud Security |
| 6 | [s06-sysdig-strategy.md](./series/s06-sysdig-strategy.md) | 総括 - Sysdig の戦略的ポジション |

Qiita版: [`platforms/qiita/ai-security-series/`](./platforms/qiita/ai-security-series/)

---

## 🗂️ プラットフォーム別（Qiita / Zenn）

### Qiita

| ファイル | 内容 |
|----------|------|
| [sysdig-monitor-guide-qiita.md](./platforms/qiita/sysdig-monitor-guide-qiita.md) | Sysdig Monitor 徹底解説（Qiita版） |
| [cnapp-hash-detection.md](./platforms/qiita/cnapp-hash-detection.md) | Hash検知ベストプラクティス（Qiita版） |
| [2025-12-08_sysdig-ids-ips-comparison.md](./platforms/qiita/2025-12-08_sysdig-ids-ips-comparison.md) | IDS/IPS比較（2025-12） |
| [2026-02-20_sysdig-malware-control-best-practices.md](./platforms/qiita/2026-02-20_sysdig-malware-control-best-practices.md) | マルウェア制御ベストプラクティス（2026-02） |
| [2025-01-15_tfdrift-falco-oss-journey.md](./platforms/qiita/2025-01-15_tfdrift-falco-oss-journey.md) | tfdrift-falco OSS開発の軌跡 |
| [2025-01-15_tfdrift-falco-progress-01.md](./platforms/qiita/2025-01-15_tfdrift-falco-progress-01.md) | tfdrift-falco 進捗レポート #1 |
| [2025-01-15_tfdrift-falco-progress-02.md](./platforms/qiita/2025-01-15_tfdrift-falco-progress-02.md) | tfdrift-falco 進捗レポート #2 |

### Zenn

| ファイル | 内容 |
|----------|------|
| [sysdig-monitor-guide-zenn.md](./platforms/zenn/sysdig-monitor-guide-zenn.md) | Sysdig Monitor 徹底解説（Zenn版） |
| [2025-01-15_tfdrift-falco-technical.md](./platforms/zenn/2025-01-15_tfdrift-falco-technical.md) | tfdrift-falco 技術解説（Zenn版） |

---

## 🚀 記事の投稿フロー

1. **ドラフト作成** → ルートの `*.md` に原稿を書く
2. **プラットフォーム用に調整** → `platforms/qiita/` または `platforms/zenn/` にフロントマター付きで配置
3. **GitHub に Push** → このリポジトリに集約
4. **各プラットフォームに投稿**
   - Qiita: `platforms/qiita/` のファイルをエディタにコピー＆ペースト
   - Zenn: `platforms/zenn/` のファイルをZenn CLIまたはGitHub連携で公開

---

## 📄 ライセンス

Copyright © 2026 keitahigaki-prog

本記事は教育目的で自由に利用できますが、商用利用の場合は著者にご連絡ください。

---

**最終更新**: 2026-02-28
