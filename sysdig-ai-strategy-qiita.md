# Sysdig が描く AI セキュリティの全体戦略
〜「AIを守る」×「AIで守る」二軸で読み解く CNAPP の現在地〜

## はじめに

Sysdig の最近の動きを追っていると、一見バラバラに見えるリリースが続いています。

- AI Workload Security
- Sysdig Sage™
- MCP Server
- Agentic Cloud Security
- AIBOM
- AI × CNAPP / AI × AWS / AI for SOC

これらをそれぞれ独立した機能として見ると、点の集合に見えます。
しかし、**一本の思想で貫くと、かなり明確なストーリーが見えてきます。**

---

## 前提：Sysdig は AI を「2つの側面」で見ている

Sysdig が描く AI セキュリティには、明確な二軸があります。

```
┌─────────────────────────────────────────────┐
│                                             │
│   ① AIを守る     ②  AIで守る               │
│   Securing AI    AI for Security            │
│                                             │
│   AIワークロード  Sysdig Sage™               │
│   自体のリスクを  Agentic Security           │
│   制御する        MCP Server                 │
│                                             │
└─────────────────────────────────────────────┘
```

この二軸が、すべての記事・機能発表を貫く **背骨** です。

以降、それぞれを分解していきます。

---

## 軸① AIを守る（Securing AI）

### AI 急速導入がもたらすリスクの実態

2023年以降、世界で **6,600万以上の新規 AI プロジェクト** が生まれました。
しかし速度優先の導入により、セキュリティ対策が後回しになるケースが頻発しています。

Sysdig のリサーチチームが特定した主要リスクは3つです。

| リスク | 概要 |
|---|---|
| **プロンプトインジェクション** | 攻撃者が工作したプロンプトで、モデルの意図した機能を迂回・機密情報取得 |
| **敵対的攻撃** | モデルを混乱させる入力を設計し、不正な出力・データ漏洩を引き起こす |
| **トロイの木馬型LLM汚染** | 訓練データに悪意あるトリガーを埋め込み、特定入力で不正コマンドを実行させる |

さらに現実的な数字として：

> **34% の GenAI ワークロードがインターネットに公開された状態**

という状況が報告されています。

<!-- スクリーンショット: Sysdig のリサーチレポートや統計データのビジュアル（34%の公開リスクなど） -->
![AI Workload Risk Statistics](./images/ss-ai-01-risk-stats.png)
*GenAI ワークロードのリスク実態（Sysdig Cloud-Native Security Report より）*

---

### AI Workload Security：CNAPP に統合された AI 保護

Sysdig の回答が **AI Workload Security** です。既存の CNAPP フレームワークに AI ワークロード保護を統合し、以下を実現します。

**検知・可視化**

- OpenAI / TensorFlow 等の AI エンジン・パッケージの自動検出
- 公開されている AI パッケージの識別
- ワークロード内の疑わしい活動の検出

**リスク評価との統合**

- **Risk Prioritization** ページでのスタックランク化されたリスク表示
- **Attack Path Analysis** による攻撃経路の可視化
- 脆弱性とエクスプロイト実行状況の関連付け

<!-- スクリーンショット: Sysdig Secure の AI Workload Security ダッシュボード（AI パッケージインベントリ・リスク一覧） -->
![AI Workload Security Dashboard](./images/ss-ai-02-ai-workload-dashboard.png)
*AI Workload Security ダッシュボード：AI パッケージの自動インベントリと公開リスクの可視化*

<!-- スクリーンショット: Attack Path Analysis で AI ワークロードへの攻撃経路が表示された画面 -->
![Attack Path Analysis - AI Workload](./images/ss-ai-03-attack-path.png)
*Attack Path Analysis：AI ワークロードへの攻撃経路を Cloud Attack Graph で可視化*

:::note info
**Cloud Attack Graph とは**
Sysdig の Risk Prioritization エンジンの中核。CVE・設定ミス・ランタイムイベント・ネットワーク公開状態を統合し、「本当に危険なリスク」を自動的にスタックランキングする機能。
:::

**規制対応**

AI Workload Security は単なる技術的保護にとどまらず、**EU AI Act** や **米国 Executive Order** への対応も支援します。AIの「使えるが安全」な状態を証明する要件が高まる中、CNAPP 側からこれを可視化・記録する仕組みが必要になっています。

---

### MCP Server のセキュリティリスク：守る側が見落としがちな盲点

AI 活用の拡大に伴い、**MCP（Model Context Protocol）** が広く普及しています。
しかし MCP Server は従来の API と根本的に異なります。

> MCP は「確定的な操作」ではなく「自律的な判断」を LLM に委ねる。
> リクエストが常に検証されるわけではない。

Sysdig が警告する MCP の主要攻撃ベクター：

| 攻撃カテゴリ | 具体例 |
|---|---|
| **LLMjacking** | LLM への不正アクセスによるコード生成・ソーシャルエンジニアリング |
| **認証情報漏洩** | API キー・パスワードが LLM トレーニングデータに混入（実例：12,000件超） |
| **プロンプトインジェクション** | MCP 経由で LLM の判断を操作し、意図しないツール実行を誘発 |

Sysdig が推奨する MCP セキュリティの4柱：

```
1. 認証と認証情報管理    ← 短命・ローテーションする認証情報 + MFA
2. 入力検証・プロンプト制御 ← サニタイゼーション + 異常パターン監視
3. 細粒度の認可・コンテキスト分離 ← 最小権限 + ロールベースアクセス
4. 継続的な監視        ← リアルタイム監視 + 定期的レッドチーミング
```

:::note warn
**重要：MCP のリスクは経営課題**
Sysdig は「このリスクを技術的問題ではなく経営戦略の中核として扱うことを経営陣に求める」と明言しています。
:::

---

## 軸② AIで守る（AI for Security）

### Sysdig Sage™ の登場：チャットボットではなく「判断する AI」

**Sysdig Sage™** は、従来の AI アシスタントとは設計思想が異なります。

```
従来の AI チャットボット：
  質問 → 要約・回答

Sysdig Sage™：
  現象 → 多段階推論 → 文脈認識 → 意思決定支援 → 行動提案
```

Sage が実現する3つの能力：

**1. 多段階推論（Multi-step Reasoning）**

複雑なクラウド脅威を層状に分析。フォローアップ質問で深掘りしながら、ランタイムイベントの本質的な意味を明確にします。

**2. ガイド付き対応（Guided Response）**

脅威の「説明」にとどまらず、積極的な対応策・予防戦略・プロセス改善を提案。Sysdig Threat Research チームの知見と直接連携しています。

**3. 自律型エージェントアーキテクチャ**

複数の専門化された AI エージェントが協働し、「意味のある、文脈を考慮した推奨事項」を生成します。

<!-- スクリーンショット: Sysdig Sage のチャット UI（質問→多段階推論→対応提案のフロー） -->
![Sysdig Sage Chat Interface](./images/ss-ai-04-sage-chat.png)
*Sysdig Sage の UI：自然言語で質問すると多段階推論で脅威を分析し、対応策を提案*

---

### AI-powered CNAPP：Sage が統合する3ドメイン

Sage は単独機能ではなく、CNAPP の全ドメインを横断するインテリジェンス層として機能します。

```
┌──────────────────────────────────────────────────────┐
│                  Sysdig Sage™                        │
│         AI-powered Intelligence Layer                │
└────────────┬──────────────┬──────────────┬───────────┘
             │              │              │
    ┌────────▼─────┐ ┌──────▼──────┐ ┌───▼───────────┐
    │    CSPM      │ │  脆弱性管理  │ │  CDR          │
    │  クラウド態勢 │ │  優先順位付け│ │  クラウド検知・│
    │  管理        │ │  + 修復      │ │  対応         │
    └──────────────┘ └─────────────┘ └───────────────┘
```

**AI グラフ検索（Graph Search）**

自然言語クエリを SysQL に自動変換。例えば：

```
「Critical脆弱性と公開エクスポージャーを
  両方持つワークロードはどれか？」
  ↓ Sage が SysQL に変換して即座に回答
```

<!-- スクリーンショット: Sysdig Sage の Graph Search UI（自然言語クエリ入力→結果表示） -->
![Sage Graph Search](./images/ss-ai-05-sage-graph-search.png)
*Graph Search：自然言語でクラウド全体の資産・リスクを横断検索*

---

### Agentic Cloud Security：Sage が「自律的に行動する」

2025年、Sysdig は Sage を **エージェント型 AI** として進化させました。
「回答する AI」から「行動する AI」へのシフトです。

従来の脆弱性管理ワークフロー：

```
人間が手動で実施していた5段階：
  発見 → 評価 → 優先順位付け → 修復指示 → 検証
  ↑ これを全部手作業でやっていた
```

Agentic Sage が自動化する内容：

| フェーズ | Sage の自動化 | 期待効果 |
|---|---|---|
| アセット発見 | セマンティック分析で本番環境を自動特定 | 見落とし排除 |
| 優先順位付け | 影響度・拡散範囲・修正可能性を AI スコアリング | 50〜95% のノイズ削減 |
| 修復 | 「1イメージ修正で7,000件の検出削減」を最大化 | 修復効率の劇的向上 |
| 自動チケット作成 | 修復指示付きで Jira に自動発行 | 人的ボトルネック解消 |
| 検証 | 自動追跡とレポート生成 | 改善の可視化 |

> **週80時間の手作業削減、クリティカル脆弱性の修復時間を「日」から「分」に短縮**

<!-- スクリーンショット: Agentic ワークフローの画面（Sage が自動的にチケット作成・修復提案しているフロー） -->
![Agentic Security Workflow](./images/ss-ai-06-agentic-workflow.png)
*Agentic Cloud Security：Sage が脆弱性トリアージから修復チケット作成まで自律的に実行*

---

### MCP Server：外部 AI から Sysdig を操作する

**Sysdig MCP Server** は、軸①（AIを守る）だけでなく軸②（AIで守る）にも属します。
ChatGPT や Claude などの外部 LLM から、Sysdig のセキュリティデータに直接アクセスできるようにする仕組みです。

```
外部 LLM（Claude / ChatGPT など）
         │
         │ MCP Protocol
         ▼
  Sysdig MCP Server
         │
         ├─ セキュリティ検出イベント
         ├─ 脆弱性情報
         ├─ Kubernetes クラスター状態
         └─ カスタムレポート生成

自動実行できるアクション：
  - Slack / PagerDuty への通知
  - Jira チケット自動作成
  - PDF / Markdown レポート生成
```

**実際のユースケース例：**

```
質問：「IngressNightmare に関連する
       脆弱なコンテナはどれか？
       本番環境への影響と
       修復優先度を教えて」

→ MCP Server が Sysdig API を自律的に呼び出し
  プロセス情報・CVE・エクスポージャーを統合分析
  CISO 向けの自動レポートを PDF で出力
```

<!-- スクリーンショット: MCP Server の接続設定画面 or LLM から Sysdig データを取得している様子 -->
![Sysdig MCP Server](./images/ss-ai-07-mcp-server.png)
*Sysdig MCP Server：外部 LLM から Sysdig のセキュリティデータに自律的にアクセス*

---

## ２つの軸が交わるところ：AI-powered CNAPP の全体像

整理すると、Sysdig の全機能はこのマトリクスに収まります。

| | **AIを守る（Securing AI）** | **AIで守る（AI for Security）** |
|---|---|---|
| **製品・機能** | AI Workload Security, AIBOM | Sysdig Sage™, Agentic Security |
| **クラウド連携** | AI × AWS（AI サービス保護） | AI × SOC（運用自動化） |
| **プロトコル** | MCP Server セキュリティ | MCP Server as a tool |
| **目的** | AI リスクの制御 | セキュリティ運用の効率化 |

**MCP が両軸に存在することが重要です。**

```
MCP Server は「使う」ものでありながら、「守る」対象でもある。
  └─ 軸② で外部 LLM から Sysdig を使う道具
  └─ 軸① で MCP 自体のセキュリティリスクを管理する対象
```

これが Sysdig の戦略的な深さです。

<!-- スクリーンショット: Sysdig Secure のダッシュボード全体像（CNAPP として全機能が統合されている状態） -->
![Sysdig CNAPP Overview](./images/ss-ai-08-cnapp-overview.png)
*Sysdig Secure の統合ダッシュボード：AI Workload Security・Sage・CDR が一体化*

---

## Sysdig の戦略的ポジション

この二軸戦略を持つプレイヤーは現時点で多くありません。

```
多くの競合が選んでいる道：
  └─ AIで守る（AI for Security）のみ
     ← AI アシスタントを付け足すだけ

Sysdig が選んだ道：
  └─ AIを守る ＋ AIで守る
     ← AI 時代のインフラを CNAPP として包括的にカバー
```

**勝ち筋の本質：**

AI ワークロードは急増し、攻撃対象面積は拡大し続けます。
組織は「AI を安全に使いたい」と「セキュリティ運用の人手不足を AI で補いたい」の両方を同時に抱えています。

Sysdig はその両方に同じプラットフォームで答える、という設計です。

---

## 参考リンク

**AI ワークロードセキュリティ**

- [AI Workload Security for CNAPP](https://www.sysdig.com/blog/ai-workload-security-for-cnapp)
- [AI Workload Security for AWS](https://www.sysdig.com/blog/ai-workload-security-for-aws)
- [The risks of rapid AI adoption](https://www.sysdig.com/blog/sysdigs-ai-workload-security-the-risks-of-rapid-ai-adoption)
- [5 Steps to Securing AI Workloads](https://www.sysdig.com/blog/5-steps-to-securing-ai-workloads)
- [A practical guide to securing AI workloads](https://www.sysdig.com/blog/ai-is-still-a-workload-a-practical-guide-to-securing-ai-workloads)

**Sysdig Sage™ / Agentic Security**

- [Sysdig Sage™ 初公開](https://www.sysdig.com/blog/sysdig-sage-a-groundbreaking-ai-security-analyst)
- [AI-powered CNAPP with Sysdig Sage™](https://www.sysdig.com/blog/ai-powered-cnapp-with-sysdig-sage)
- [From triage to action: Agentic Cloud Security](https://www.sysdig.com/blog/from-triage-to-action-agentic-cloud-security)
- [Intelligent vulnerability remediation](https://www.sysdig.com/blog/introducing-intelligent-vulnerability-remediation-powered-by-sysdig-sage)
- [AI for SOC: 5 cloud security prompts](https://www.sysdig.com/blog/ai-for-soc-teams-5-cloud-security-prompts-to-start-your-day-with-sysdig-sage)
- [Sysdig Sage for Search](https://www.sysdig.com/blog/introducing-sysdig-sage-for-search)
- [AI-powered vulnerability management (2026)](https://www.sysdig.com/blog/how-sysdig-sage-delivers-ai-powered-real-world-vulnerability-management)

**MCP Server**

- [Sysdig MCP Server 公開](https://www.sysdig.com/blog/sysdig-mcp-server-bridging-ai-and-cloud-security-insights)
- [MCP Server セキュリティ論](https://www.sysdig.com/blog/why-mcp-server-security-is-critical-for-ai-driven-enterprises)
- [MCP + Amazon Q Developer](https://www.sysdig.com/blog/shifting-left-with-ai-and-mcp-sysdig-amazon-q-developer)
- [Snyk 連携 MCP](https://www.sysdig.com/blog/ai-echolocation-of-cloud-risks-using-sysdig-and-snyk-mcp-servers)

---

## おわりに

「AIを守る」と「AIで守る」。

この二軸は矛盾しません。むしろ **同じコインの表と裏** です。

AI ワークロードを守るためには、AI の能力で運用効率を上げなければスケールできない。
AI で守るためには、AI ワークロード自体のリスクを理解していなければ判断が歪む。

Sysdig が CNAPP に AI を統合した理由は、ここにあります。

> セキュリティ製品は、時代の変化を前提に設計されている。
> AI 時代のインフラを守るには、AI 時代の CNAPP が必要です。
