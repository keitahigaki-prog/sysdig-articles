# AIで守る①：Sysdig Sage™ の設計思想
〜チャットボットではなく「判断し・行動する」AI アナリスト〜

> **シリーズ一覧**
> - [#1 総論：二軸フレームワーク](./s01-ai-framework-overview.md)
> - [#2 AIを守る① - AI Workload Security](./s02-securing-ai-workloads.md)
> - [#3 AIを守る② - MCP Server のリスク](./s03-mcp-server-security.md)
> - **#4 AIで守る① - Sysdig Sage™ の設計思想**（この記事）
> - [#5 AIで守る② - Agentic Cloud Security](./s05-agentic-cloud-security.md)
> - [#6 総括 - Sysdig の戦略的ポジション](./s06-sysdig-strategy.md)

---

## はじめに

ここから軸②「**AIで守る（AI for Security）**」に入ります。

多くの SaaS セキュリティ製品が「AI 機能」を追加しています。
しかしその多くは、既存機能に **チャットボット UI を付け加えたもの** です。

Sysdig Sage™ は設計思想が根本的に異なります。

---

## 「チャットボット」と「Sage」の違い

まずここを正確に理解することが重要です。

```
一般的な AI アシスタント（チャットボット型）：

  ユーザー ──→ 質問
               ↓
         ドキュメント検索
               ↓
  ユーザー ←── 回答・要約

  = 「情報を提示する」ツール
```

```
Sysdig Sage™ の設計：

  ユーザー ──→ 現象の観察（UI上のデータ）
               ↓
         ■ 多段階推論
         ■ 文脈認識（何が起きているか理解）
         ■ 根本原因特定
         ■ 対応策生成
         ■ UI 上の関連ビューへのリンク
               ↓
  ユーザー ←── 「次に何をすべきか」の提案

  = 「意思決定を支援する」ツール
```

Sysdig の表現：

> 「基本的なクエリと要約を超えた、多段階推論と文脈認識を備えた真のセキュリティ分析ツール」

---

## Sage の3つの核心能力

### 能力① 多段階推論（Multi-step Reasoning）

従来のシングルターン質問応答ではなく、
**複雑な脅威を層状に分析**していきます。

```
第1層：「このアラートは何か？」
  ↓
第2層：「このコンテナで何が起きていたか？」
  ↓
第3層：「なぜこのプロセスがこの動作をしたか？」
  ↓
第4層：「侵害の可能性はどこから始まったか？」
  ↓
結論：「根本原因はXで、対応優先度はYです」
```

人間のセキュリティアナリストが行う調査プロセスを、
AI が自律的に辿ります。

<!-- スクリーンショット: Sysdig Sage の多段階推論プロセスが見えるチャット画面 -->
![Sage Multi-step Reasoning](./images/ss-s04-01-sage-reasoning.png)
*Sage の多段階推論：フォローアップ質問を重ねながら根本原因に到達する*

### 能力② ガイド付き対応（Guided Response）

Sage は説明で終わりません。
**「次に何をすべきか」まで提案します。**

提案の種類：

| 提案タイプ | 内容 |
|---|---|
| **即時対応策** | 今すぐコンテナを停止すべきか、隔離すべきか |
| **予防戦略** | 同様の事象が再発しないための設定変更 |
| **プロセス改善** | 検知・対応フローの改善提案 |

これは **Sysdig Threat Research チームの知見** と連携して生成されます。
最新の脅威インテリジェンスに基づいた提案が可能です。

### 能力③ UI との統合（Contextual Linking）

Sage の回答は、**Sysdig Secure の UI 上で関連するビューに直接リンク**されます。

```
Sage：「このイベントはコンテナAで発生したドリフトです。
       詳細はこちら ──→ [コンテナAのタイムライン]
       関連する脆弱性はこちら ──→ [CVE-XXXX-XXXX]」
```

チャットから確認画面へのシームレスな移動が可能で、
調査にかかる時間を大幅に短縮します。

<!-- スクリーンショット: Sage の回答内に UI へのリンクが埋め込まれた状態 -->
![Sage Contextual Linking](./images/ss-s04-02-sage-context-link.png)
*Sage の文脈リンク：回答内から直接関連ビューへジャンプ*

---

## 自律型エージェントアーキテクチャ

Sage の内部構造は「単一のLLM」ではありません。

```
Sysdig Sage™ の内部：

  ┌─────────────────────────────────────────┐
  │          Orchestrator Agent             │
  │    （全体の推論フローを制御）             │
  └────────┬──────────┬──────────┬──────────┘
           │          │          │
  ┌────────▼──┐  ┌────▼──────┐ ┌▼───────────┐
  │ Threat    │  │  Vuln     │ │  Posture   │
  │ Analysis  │  │  Analysis │ │  Analysis  │
  │  Agent    │  │  Agent    │ │  Agent     │
  └───────────┘  └───────────┘ └────────────┘
```

複数の専門化された AI エージェントが協働することで、
単一モデルでは難しい **「意味のある、文脈を考慮した推奨事項」** を生成します。

---

## Sage が統合する3つのセキュリティドメイン

Sage は CNAPP の全ドメインに対して AI 支援を提供します。

```
┌──────────────────────────────────────────────────────┐
│                  Sysdig Sage™                        │
│         AI-powered Intelligence Layer                │
└────────────┬──────────────┬──────────────┬───────────┘
             │              │              │
    ┌────────▼─────┐ ┌──────▼──────┐ ┌───▼───────────┐
    │    CSPM      │ │  脆弱性管理  │ │  CDR          │
    │  クラウド態勢 │ │  優先順位付け│ │  クラウド検知・│
    │  管理        │ │  + 修復提案  │ │  対応         │
    └──────────────┘ └─────────────┘ └───────────────┘
```

**インシデント対応時間を最大76%削減** というデータが公表されています。

---

## AI グラフ検索（Graph Search）：自然言語でクラウド全体を検索

Sage の具体的な機能の一つが **Graph Search** です。

```
従来の検索：
  SQL / SysQL を手書きする
  例）SELECT * FROM workloads WHERE vuln_severity = 'CRITICAL' AND ...

Graph Search：
  自然言語で質問する
  例）「Critical脆弱性と公開エクスポージャーを両方持つワークロードはどれか？」
       ↓ Sage が自動で SysQL に変換して実行
       ↓ 結果を表示 + 解説
```

クラウド全体の資産・リスク・イベントを、**SysQL の知識なしに横断検索**できます。

<!-- スクリーンショット: Sysdig Sage の Graph Search UI（自然言語クエリを入力して結果が表示された状態） -->
![Sage Graph Search](./images/ss-s04-03-graph-search.png)
*Graph Search：「Critical脆弱性があり公開されているワークロード」を自然言語で一発検索*

---

## 脆弱性管理への AI 支援

Sage の脆弱性管理支援は具体的です。

開発チームが直面する現実：

```
1日に報告される CVE 数：数百〜数千件
実際に対処が必要なもの：ごく一部
しかし「どれが本当に危険か」を判断するのに
専門家が何時間もかかる
```

Sage の支援：

1. **優先度の自動判断** — CVE の理論値ではなく、実際の環境でのエクスプロイト可能性を評価
2. **段階的な修復指示** — アプリケーション依存関係を壊さない安全な修復手順を提示
3. **修復効果の最大化** — 「1つのイメージ修正で最大7,000件の検出を削減」する優先度を計算

<!-- スクリーンショット: Sage の脆弱性修復提案画面（CVE 一覧 + 対応優先度 + 修復手順） -->
![Sage Vulnerability Remediation](./images/ss-s04-04-vuln-remediation.png)
*Sage による脆弱性修復提案：修復効果が最大になるイメージを優先して提示*

---

## 80言語対応という設計判断

Sage は **80言語以上** に対応しています。

これは「多言語サポート」という機能追加ではなく、
**グローバルなセキュリティチームが母語で使える**という設計思想の現れです。

日本語でも Sage に問いかけることができます。

---

## まとめ

Sysdig Sage™ は「チャットボットを付けた」製品ではありません。

```
設計の核心：
  多段階推論 × 文脈認識 × エージェントアーキテクチャ
       ↓
  「情報の提示」ではなく「意思決定の支援」
```

しかし Sage の能力はここで終わりではありません。
次の記事では、Sage が **「提案する」からさらに進んで「自律的に行動する」** ところまで進化した
**Agentic Cloud Security** を解説します。

---

## 参考リンク

- [Sysdig Sage™: A groundbreaking AI security analyst](https://www.sysdig.com/blog/sysdig-sage-a-groundbreaking-ai-security-analyst)
- [AI-powered CNAPP with Sysdig Sage™](https://www.sysdig.com/blog/ai-powered-cnapp-with-sysdig-sage)
- [Introducing Sysdig Sage for Search](https://www.sysdig.com/blog/introducing-sysdig-sage-for-search)
- [AI for SOC teams: 5 cloud security prompts](https://www.sysdig.com/blog/ai-for-soc-teams-5-cloud-security-prompts-to-start-your-day-with-sysdig-sage)
- [How Sysdig Sage delivers AI-powered vulnerability management](https://www.sysdig.com/blog/how-sysdig-sage-delivers-ai-powered-real-world-vulnerability-management)
