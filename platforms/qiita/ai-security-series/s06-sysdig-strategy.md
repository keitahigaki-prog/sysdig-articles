# 総括：Sysdig の戦略的ポジション
〜AI 時代の CNAPP が描く競合優位性と二軸戦略の本質〜

> **シリーズ一覧**
> - [#1 総論：二軸フレームワーク](./s01-ai-framework-overview.md)
> - [#2 AIを守る① - AI Workload Security](./s02-securing-ai-workloads.md)
> - [#3 AIを守る② - MCP Server のリスク](./s03-mcp-server-security.md)
> - [#4 AIで守る① - Sysdig Sage™ の設計思想](./s04-sysdig-sage-design.md)
> - [#5 AIで守る② - Agentic Cloud Security](./s05-agentic-cloud-security.md)
> - **#6 総括 - Sysdig の戦略的ポジション**（この記事）

---

## はじめに

このシリーズでは、一見バラバラに見える Sysdig の機能群を
「AIを守る（Securing AI）× AIで守る（AI for Security）」の二軸で読み解いてきました。

最終回は戦略レベルの総括です。

- なぜ二軸が必要なのか
- MCP の逆説が意味すること
- Sysdig の勝ち筋はどこにあるのか

---

## シリーズの全体像を振り返る

```
軸①：AIを守る（Securing AI）

  #2 AI Workload Security
    ├─ GenAI ワークロードの可視化・インベントリ
    ├─ 3類型のリスク（プロンプトインジェクション等）
    ├─ CNAPP との統合（Risk Prioritization / Attack Path）
    └─ AIBOM・規制対応

  #3 MCP Server のリスク
    ├─ 確率論的な実行主体（LLM）がもたらす新攻撃面
    ├─ LLMjacking / 認証情報漏洩 / プロンプトインジェクション
    └─ 4つの対策柱

軸②：AIで守る（AI for Security）

  #4 Sysdig Sage™ の設計思想
    ├─ チャットボットではなく意思決定支援ツール
    ├─ 多段階推論 × 文脈認識 × エージェントアーキテクチャ
    └─ CSPM / 脆弱性管理 / CDR の3ドメイン統合

  #5 Agentic Cloud Security
    ├─ 「提案する」から「自律的に行動する」へ
    ├─ 週80時間の手作業を自動化
    └─ MCP Server as a tool
```

<!-- スクリーンショット: Sysdig Secure のダッシュボード全体像（全機能が統合されている状態） -->
![Sysdig CNAPP Overview](./images/ss-s06-01-cnapp-overview.png)
*Sysdig Secure の統合ダッシュボード：AI Workload Security・Sage・CDR が一体化*

---

## 全機能をマトリクスで整理する

| 機能・トピック | AIを守る | AIで守る | 両方 |
|---|:---:|:---:|:---:|
| AI Workload Security | ✅ | | |
| AIBOM | ✅ | | |
| AI × AWS | ✅ | | |
| MCP Server セキュリティ | ✅ | | |
| Sysdig Sage™ | | ✅ | |
| Agentic Security | | ✅ | |
| AI for SOC | | ✅ | |
| **MCP Server（ツールとして）** | | | **✅** |

**MCP Server だけが両軸に存在します。**

---

## MCP の逆説：使う道具でもあり、守る対象でもある

このシリーズを通じて最も重要な構造的インサイトが、MCP の逆説です。

```
MCP Server の二面性：

  ┌─────────────────────────────────────────────┐
  │                                             │
  │  軸①（守る対象）として：                    │
  │  LLM × 外部ツール連携が生む新攻撃面         │
  │  LLMjacking / 認証情報漏洩 / プロンプトインジェクション │
  │                                             │
  │  軸②（使う道具）として：                    │
  │  外部 LLM から Sysdig データに自律アクセス   │
  │  Agentic ワークフローの拡張                 │
  │                                             │
  └─────────────────────────────────────────────┘
```

これは偶然の一致ではありません。

**AI の能力と AI のリスクは表裏一体** であり、
その両方を同一プラットフォームで扱えることが、CNAPP としての差別化になっています。

---

## 競合他社が選んでいる道

現在の市場で多くの CNAPP / セキュリティベンダーが選んでいる戦略：

```
パターンA：AIで守る（AI for Security）のみ
  └─ 既存製品に AI チャットボット・AI 分析を追加
  └─ 差別化は弱い（機能追加の印象）
  └─ AI ワークロード自体の保護は未対応

パターンB：AIを守る（Securing AI）のみ
  └─ AI セキュリティ専門ベンダー
  └─ 既存の CNAPP との統合コストが高い
  └─ 運用の分断が発生

パターンC（Sysdig）：両軸を同一プラットフォームで
  └─ 新ツール不要・既存 CNAPP の中で AI を守れる
  └─ Sage によって守る側にも AI が活用される
  └─ MCP が使う道具かつ守る対象として機能
```

---

## Sysdig の勝ち筋：プラットフォームとしての一体化

Sysdig の競合優位性は「機能の数」ではありません。

**一体化している** ことが強みです。

```
AI Workload Security が検知したリスク
         ↓
Attack Path Analysis で攻撃経路を可視化
         ↓
Sage が「何から直すべきか」を提案
         ↓
Agentic Sage が修復チケットを自動作成
         ↓
MCP Server 経由で外部 LLM からも参照可能
         ↓
Falco が修復後も継続してランタイムを監視
```

このループが **同一プラットフォームで完結する** ことが価値です。
別ツールを組み合わせると、このループのどこかで断絶が起きます。

<!-- スクリーンショット: Sysdig の CNAPP フルサイクル（検知→分析→対応→監視）を示すアーキテクチャ図 -->
![Sysdig Platform Loop](./images/ss-s06-02-platform-loop.png)
*Sysdig CNAPP のフルサイクル：AI Workload Security → Sage → Agentic → Falco が統合*

---

## AI 時代に問われる「何を守るべきか」の再定義

このシリーズを通じて、ひとつの問いに行き着きます。

```
従来のクラウドセキュリティが守っていたもの：
  コンテナ / Kubernetes / クラウド設定 / API

AI 時代に加わる守る対象：
  + AI モデル・ウェイト
  + トレーニングデータ
  + LLM エンドポイント
  + MCP Server とその接続先
  + プロンプトの入出力経路
```

そして守る手段も変わりました。

```
従来：
  人間がルールを書き → アラートを確認 → 対応する

AI 時代：
  AI がパターンを学習し → 自律的に分析 → 提案・実行する
  人間は戦略的判断に集中する
```

---

## Sysdig が賭けていること

最後に、Sysdig の戦略を一文で表現するとすれば：

> **「AI 時代のインフラを守るには、AI 時代の CNAPP が必要だ」**

そしてその CNAPP は：
- AI を **守れる** こと
- AI を **使って守れる** こと
- その両方を **同じプラットフォームで** できること

これが Sysdig が賭けているポジションです。

---

## シリーズのまとめ

| 記事 | 主なテーマ | キーメッセージ |
|---|---|---|
| [#1 総論](./s01-ai-framework-overview.md) | 二軸フレームワーク | バラバラに見える機能を一本の思想で読む |
| [#2 AI Workload Security](./s02-securing-ai-workloads.md) | AI リスクの実態 | 34%の GenAI ワークロードが公開状態 |
| [#3 MCP リスク](./s03-mcp-server-security.md) | 新攻撃面 | LLM の自律性がセキュリティの境界を変える |
| [#4 Sysdig Sage™](./s04-sysdig-sage-design.md) | AI アナリストの設計 | チャットボットではなく意思決定支援 |
| [#5 Agentic Security](./s05-agentic-cloud-security.md) | 自律型ワークフロー | 週80時間の手作業を分単位に |
| [#6 戦略総括](./s06-sysdig-strategy.md) | 競合優位性 | 一体化がプラットフォームの価値 |

---

## おわりに

「AIを守る」と「AIで守る」。

この二軸は矛盾しません。むしろ **同じコインの表と裏** です。

AI ワークロードを守るためには、AI の能力で運用効率を上げなければスケールできない。
AI で守るためには、AI ワークロード自体のリスクを理解していなければ判断が歪む。

Sysdig が CNAPP に AI を統合した理由は、ここにあります。

> セキュリティ製品は、時代の変化を前提に設計されている。
> AI 時代のインフラを守るには、AI 時代の CNAPP が必要です。

---

## 参考リンク（シリーズ全体）

**AI ワークロードセキュリティ**
- [AI Workload Security for CNAPP](https://www.sysdig.com/blog/ai-workload-security-for-cnapp)
- [AI Workload Security for AWS](https://www.sysdig.com/blog/ai-workload-security-for-aws)
- [The risks of rapid AI adoption](https://www.sysdig.com/blog/sysdigs-ai-workload-security-the-risks-of-rapid-ai-adoption)
- [5 Steps to Securing AI Workloads](https://www.sysdig.com/blog/5-steps-to-securing-ai-workloads)

**Sysdig Sage™ / Agentic Security**
- [Sysdig Sage™ 初公開](https://www.sysdig.com/blog/sysdig-sage-a-groundbreaking-ai-security-analyst)
- [AI-powered CNAPP with Sysdig Sage™](https://www.sysdig.com/blog/ai-powered-cnapp-with-sysdig-sage)
- [From triage to action: Agentic Cloud Security](https://www.sysdig.com/blog/from-triage-to-action-agentic-cloud-security)
- [Intelligent vulnerability remediation](https://www.sysdig.com/blog/introducing-intelligent-vulnerability-remediation-powered-by-sysdig-sage)
- [AI for SOC: 5 cloud security prompts](https://www.sysdig.com/blog/ai-for-soc-teams-5-cloud-security-prompts-to-start-your-day-with-sysdig-sage)

**MCP Server**
- [Sysdig MCP Server 公開](https://www.sysdig.com/blog/sysdig-mcp-server-bridging-ai-and-cloud-security-insights)
- [Why MCP server security is critical](https://www.sysdig.com/blog/why-mcp-server-security-is-critical-for-ai-driven-enterprises)
- [MCP + Amazon Q Developer](https://www.sysdig.com/blog/shifting-left-with-ai-and-mcp-sysdig-amazon-q-developer)
- [AI echolocation with Snyk MCP](https://www.sysdig.com/blog/ai-echolocation-of-cloud-risks-using-sysdig-and-snyk-mcp-servers)
