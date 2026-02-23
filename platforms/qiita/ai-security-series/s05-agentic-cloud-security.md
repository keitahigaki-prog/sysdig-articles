# AIで守る②：Agentic Cloud Security
〜人間の介在を最小化する自律型セキュリティの現在地〜

> **シリーズ一覧**
> - [#1 総論：二軸フレームワーク](./s01-ai-framework-overview.md)
> - [#2 AIを守る① - AI Workload Security](./s02-securing-ai-workloads.md)
> - [#3 AIを守る② - MCP Server のリスク](./s03-mcp-server-security.md)
> - [#4 AIで守る① - Sysdig Sage™ の設計思想](./s04-sysdig-sage-design.md)
> - **#5 AIで守る② - Agentic Cloud Security**（この記事）
> - [#6 総括 - Sysdig の戦略的ポジション](./s06-sysdig-strategy.md)

---

## はじめに

前回の記事（[#4](./s04-sysdig-sage-design.md)）では、Sysdig Sage™ が「意思決定を支援する AI」として設計されていることを解説しました。

この記事では Sage のさらなる進化形として、**Agentic Cloud Security** を扱います。

「提案する」から「自律的に行動する」へ。
この移行が何を意味するか、実際のワークフローで見ていきます。

---

## 「Agentic」とは何か

まず言葉を整理します。

```
AIの進化ステップ：

  Level 1：生成 AI（Generative AI）
    → テキスト・画像を生成する
    → 受け身のツール

  Level 2：アシスタント AI（Assistant AI）
    → 質問に答える、要約する
    → 情報を提示するツール（Sage の初期形）

  Level 3：エージェント型 AI（Agentic AI）
    → 目標が与えられると、自律的にタスクを計画・実行する
    → 意思決定し、行動するツール
    → Sysdig Sage™ の現在地
```

Sysdig はこのレベル3を「Agentic Cloud Security」と呼んでいます。

---

## 従来の脆弱性管理ワークフローの問題

Agentic の価値を理解するには、まず「今どれだけ人手がかかっているか」を確認する必要があります。

### 従来のワークフロー（5段階・フルマニュアル）

```
1. 発見（Discovery）
   └─ SIEM / スキャンツールがアラートを大量生成
   └─ 人間がリストを確認

2. 評価（Assessment）
   └─ それぞれの CVE の深刻度を調査
   └─ 複数のドキュメント・ツールを横断して確認

3. 優先順位付け（Prioritization）
   └─ どれから直すか？を人間が判断
   └─ 理論値（CVSS スコア）で判断 → 本当に危険なものを見落とす

4. 修復指示（Remediation）
   └─ 担当チームへの指示書を手書き
   └─ 「どのバージョンに上げるべきか」を調査

5. 検証（Verification）
   └─ 修復後に本当に直ったかを確認
   └─ ドキュメントに記録
```

このプロセスを繰り返すために、**週80時間** の手作業が発生するケースが報告されています。

<!-- スクリーンショット: 従来の脆弱性管理の工数を示す資料 or Sysdig のリサーチデータ -->
![Traditional Vulnerability Management Cost](./images/ss-s05-01-manual-cost.png)
*従来の脆弱性管理：週80時間の手作業が現実*

---

## Agentic Sage が変えること

Sysdig Sage™ がエージェント型に進化することで、この5段階が大きく変わります。

### 新しいワークフロー（Agentic版）

| フェーズ | 人間の作業 | Sage の自動化 | 効果 |
|---|---|---|---|
| **発見** | 承認のみ | セマンティック分析で本番環境を自動特定 | 見落とし排除 |
| **評価** | 例外対応のみ | CVE × 実環境エクスプロイト可能性を自動評価 | 評価精度の向上 |
| **優先順位付け** | 戦略的判断のみ | 影響度・拡散範囲・修正可能性で AI スコアリング | ノイズ50〜95%削減 |
| **修復指示** | レビューのみ | 修復効果最大化の方針 + 段階的手順を自動生成 | 指示書作成の自動化 |
| **検証・記録** | 監査対応のみ | 自動追跡 + レポート生成 | ドキュメント工数ゼロ |

```
結果：
  クリティカル脆弱性の修復時間
  「日単位」→「分単位」へ短縮
```

<!-- スクリーンショット: Agentic Sage のワークフロー画面（自動的に脆弱性をトリアージして優先順位をつけている状態） -->
![Agentic Sage Triage](./images/ss-s05-02-agentic-triage.png)
*Agentic Sage：脆弱性を自動トリアージして優先度付きで一覧化*

---

## ノイズ削減：50〜95%

セキュリティ運用で最大の問題の一つが **アラート疲れ（Alert Fatigue）** です。

```
現実：
  毎日数千件のアラートが生成される
  そのほとんどは低リスクまたは誤検知
  本当に危険なものが埋もれる
```

Agentic Sage のノイズ削減アプローチ：

```
低リスク警告の自動フィルタリング（50〜95%削減）
    ↓
残ったものだけを人間にエスカレーション
    ↓
人間は「本当に重要な判断」だけに集中できる
```

<!-- スクリーンショット: アラート一覧でノイズが除去されて重要アラートのみが表示されている状態 -->
![Alert Noise Reduction](./images/ss-s05-03-noise-reduction.png)
*ノイズ削減：50〜95% の低リスクアラートを自動フィルタリング*

---

## Jira 自動チケット：「提案」から「実行」へ

Agentic の最も具体的な例が **Jira チケットの自動作成** です。

```
従来：
  アナリストが調査 → 手動でチケット作成 → 開発者に依頼

Agentic Sage：
  Sage が調査 → 修復手順付きでチケットを自動作成 → 開発者に通知
```

チケットには以下が自動生成されます：

```markdown
## 対象
コンテナ: payment-service-v2.3
イメージ: registry.example.com/payment:2.3.1

## 脆弱性
CVE-2024-XXXX（CRITICAL）
パッケージ: openssl 3.0.2

## 優先度
HIGH（インターネット公開 × 実エクスプロイト確認済み）

## 推奨修復手順
1. openssl を 3.0.7 以上にアップデート
2. 再ビルド後、ステージング環境でテスト
3. 本番デプロイ後、ランタイム検知を24時間監視

## 修復効果
このイメージ修正により 4,300件の関連検出が解消される見込み
```

<!-- スクリーンショット: Sage が自動生成した Jira チケット or チケット作成の設定画面 -->
![Auto Jira Ticket](./images/ss-s05-04-jira-ticket.png)
*Sage が修復手順付きで Jira チケットを自動生成する*

---

## MCP Server as a Tool：外部 LLM からの操作

Agentic Security の拡張形として、**MCP Server をツールとして使う**方向があります。

```
Claude や ChatGPT などの外部 LLM
         │
         │ MCP Protocol
         ▼
  Sysdig MCP Server
         │
  ┌──────┼──────┐
  │      │      │
  ↓      ↓      ↓
検出  脆弱性  Kubernetes
情報   情報   クラスター状態

自動実行：
  Slack 通知
  PagerDuty アラート
  Jira チケット作成
  カスタムレポート PDF 生成
```

これにより、**Sysdig Secure の外側から** Agentic ワークフローをトリガーできます。

例えば、朝のスタンドアップ前に Claude に「昨夜のインシデントサマリーを作って」と頼むだけで、
Sysdig のデータから自動的にレポートが生成されます。

<!-- スクリーンショット: 外部 LLM（Claude や ChatGPT）から Sysdig MCP Server を経由してデータを取得している様子 -->
![MCP as Tool](./images/ss-s05-05-mcp-as-tool.png)
*MCP Server as a Tool：外部 LLM から Sysdig データに直接アクセスしてレポート生成*

---

## AI for SOC：日次業務への適用

Agentic Security は大規模インシデント対応だけではありません。
**SOC チームの日次業務** への適用が進んでいます。

Sysdig が公開している具体的なプロンプト例（日次）：

```
朝のルーティン：
  「今日最も対応が必要なクリティカルアラートを
   コンテキスト付きで上位5件教えて」

脆弱性確認：
  「今週新たに検出された Critical CVE で
   本番環境に影響するものをリストアップして」

インシデント後確認：
  「昨夜のアラートで対応済みでないものはあるか？」
```

これらのプロンプトにより、毎朝の確認作業が大幅に効率化されます。

---

## 「自律型」の注意点：人間の監督は不可欠

Agentic AI は強力ですが、**無条件に信頼すべきものではありません**。

Sysdig の設計でも、完全自律ではなく **Human-in-the-loop** の構造を維持しています。

```
Sage が自律的に実行できること：
  ├─ 調査・分析
  ├─ 優先順位付け
  ├─ チケット作成
  └─ 通知送信

人間の承認が必要なこと：
  ├─ 本番環境のコンテナ停止・削除
  ├─ セキュリティポリシーの変更
  └─ 重大なインフラ変更
```

「自律型」は「人間不要」ではありません。
人間を **戦略的判断** に集中させるための設計です。

---

## まとめ

Agentic Cloud Security が実現するもの：

```
Before：
  週80時間の手作業
  クリティカル脆弱性の対応に数日
  アラート疲れで本当の危機を見落とす

After：
  調査・優先順位付け・チケット作成は自動
  クリティカル脆弱性の対応は分単位
  人間は本当に重要な判断に集中
```

次の記事（最終回）では、ここまで見てきた「AIを守る」「AIで守る」の二軸を統合し、
**Sysdig の戦略的ポジションと勝ち筋**を整理します。

---

## 参考リンク

- [From triage to action: Agentic Cloud Security](https://www.sysdig.com/blog/from-triage-to-action-agentic-cloud-security)
- [The vision comes to life: Agentic cloud security with Sysdig Sage™](https://www.sysdig.com/blog/agentic-cloud-security-with-sysdig-sage)
- [Introducing intelligent vulnerability remediation](https://www.sysdig.com/blog/introducing-intelligent-vulnerability-remediation-powered-by-sysdig-sage)
- [AI for SOC teams: 5 cloud security prompts](https://www.sysdig.com/blog/ai-for-soc-teams-5-cloud-security-prompts-to-start-your-day-with-sysdig-sage)
- [Sysdig MCP Server: Bridging AI and cloud security insights](https://www.sysdig.com/blog/sysdig-mcp-server-bridging-ai-and-cloud-security-insights)
