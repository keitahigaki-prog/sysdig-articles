# Sysdig で読み解く AI 時代のクラウドセキュリティ【総論】
〜「AIを守る」×「AIで守る」二軸フレームワーク〜

> **シリーズ一覧**
> - **#1 総論：二軸フレームワーク**（この記事）
> - [#2 AIを守る① - AI Workload Security](./s02-securing-ai-workloads.md)
> - [#3 AIを守る② - MCP Server のリスク](./s03-mcp-server-security.md)
> - [#4 AIで守る① - Sysdig Sage™ の設計思想](./s04-sysdig-sage-design.md)
> - [#5 AIで守る② - Agentic Cloud Security](./s05-agentic-cloud-security.md)
> - [#6 総括 - Sysdig の戦略的ポジション](./s06-sysdig-strategy.md)

---

## はじめに

Sysdig の最近のリリースを追っていると、一見バラバラに見えるキーワードが続いています。

- **AI Workload Security**
- **Sysdig Sage™**
- **MCP Server**
- **Agentic Cloud Security**
- **AIBOM**
- **AI × CNAPP / AI × AWS / AI for SOC**

これらを個別の機能追加として見ると、点の集合に見えます。
しかし **一本の思想で貫くと、かなり明確なストーリーが見えてきます。**

このシリーズでは、Sysdig が AI をどう捉え、市場をどう見ていて、どこに勝ち筋を置いているのかを、**戦略レベルで分解**していきます。

---

## 前提：Sysdig は AI を「2つの側面」で見ている

Sysdig の AI 関連機能はすべて、以下の二軸のどちらか（あるいは両方）に属しています。

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│   軸①：AIを守る              軸②：AIで守る              │
│   Securing AI                AI for Security            │
│                                                          │
│   AIワークロード自体の        AIの能力を使って            │
│   リスクを制御する            セキュリティ運用を改善する   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

この二軸が、すべての記事・機能発表を貫く **背骨** です。

---

## 各軸の中身を俯瞰する

### 軸①：AIを守る（Securing AI）

AI の急速な普及により、組織のリスク対象面積は急拡大しています。

2023年以降、世界で **6,600万以上の新規 AI プロジェクト** が生まれました。
しかし **34% の GenAI ワークロードがインターネットに公開された状態** というのが現実です。

これに対して Sysdig が答えるのが **AI Workload Security** です。

| 機能・トピック | 位置づけ |
|---|---|
| **AI Workload Security** | AI パッケージ・モデルの可視化とリスク評価 |
| **AIBOM**（AI Bill of Materials） | AI コンポーネントの構成管理・追跡 |
| **MCP Server Security** | LLM 連携ツールの新たな攻撃面を守る |
| **規制対応** | EU AI Act / 米国 Executive Order への準備 |

### 軸②：AIで守る（AI for Security）

セキュリティ運用における人手不足・スキルギャップを、AI が補う方向性です。

| 機能・トピック | 位置づけ |
|---|---|
| **Sysdig Sage™** | 多段階推論で脅威を分析し、対応策を提案する AI アナリスト |
| **Agentic Security** | 人間の介在を最小化した自律型ワークフロー |
| **MCP Server（ツールとして）** | 外部 LLM から Sysdig データに直接アクセス |
| **AI for SOC** | SOC チームの日次業務をプロンプトで自動化 |

---

## この二軸には「逆説」がある

注目すべきは **MCP Server** です。

```
MCP Server は：
  ├─ 軸② として：外部 LLM から Sysdig を操作する「道具」
  └─ 軸① として：LLM 自体が新たな攻撃面になる「守る対象」
```

使う道具でありながら、守る対象でもある。
これが Sysdig の戦略の深さを示す象徴的な構造です。

<!-- スクリーンショット: シリーズ全体を示す図（二軸マトリクス）またはSysdig Secureのトップダッシュボード -->
![Two-Axis Framework Overview](./images/ss-s01-01-framework.png)
*Sysdig AI 戦略の二軸フレームワーク。全機能がこの2軸のどこかに位置する*

---

## シリーズのロードマップ

```
#1 総論（この記事）
│   └─ 二軸フレームワークの概説
│
├─── #2 AIを守る① - AI Workload Security
│        └─ GenAI時代のリスクと保護手法
│
├─── #3 AIを守る② - MCP Server のリスク
│        └─ AI連携ツールがもたらす新攻撃面
│
├─── #4 AIで守る① - Sysdig Sage™
│        └─ 「判断し・行動する」AIアナリスト
│
├─── #5 AIで守る② - Agentic Cloud Security
│        └─ 脆弱性管理の自律化
│
└─── #6 総括 - 戦略的ポジション
         └─ Sysdig の勝ち筋と AI 時代の CNAPP
```

---

## なぜ今このタイミングか

従来のクラウドセキュリティは「インフラを守る」ものでした。
AI 時代に入り、**守る対象そのものが変わった** のです。

```
従来：                   AI時代：
コンテナ                 コンテナ ＋ AI モデル
API                     API ＋ LLM エンドポイント
設定ミス                 設定ミス ＋ プロンプトインジェクション
CVE                     CVE ＋ トレーニングデータ汚染
```

そして守る側も変わりました。

```
従来：          AI時代：
人間が検知       AI が検知 ＋ 人間が判断
人間が対応       AI が提案 ＋ 人間が承認
人間が修復       AI が実行 ＋ 人間が監視
```

Sysdig はこの変化を **プラットフォームとして一体化する** ことに賭けています。

次の記事では、**軸①「AIを守る」** の最初の具体例として、
**AI Workload Security** を深掘りします。

---

## 参考リンク

- [AI Workload Security for CNAPP](https://www.sysdig.com/blog/ai-workload-security-for-cnapp)
- [Sysdig Sage™ 初公開](https://www.sysdig.com/blog/sysdig-sage-a-groundbreaking-ai-security-analyst)
- [Sysdig MCP Server](https://www.sysdig.com/blog/sysdig-mcp-server-bridging-ai-and-cloud-security-insights)
- [From triage to action: Agentic Cloud Security](https://www.sysdig.com/blog/from-triage-to-action-agentic-cloud-security)
