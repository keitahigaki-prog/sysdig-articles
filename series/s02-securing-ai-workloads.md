# AIを守る①：AI Workload Security
〜GenAI時代のリスクと CNAPP による保護の実態〜

> **シリーズ一覧**
> - [#1 総論：二軸フレームワーク](./s01-ai-framework-overview.md)
> - **#2 AIを守る① - AI Workload Security**（この記事）
> - [#3 AIを守る② - MCP Server のリスク](./s03-mcp-server-security.md)
> - [#4 AIで守る① - Sysdig Sage™ の設計思想](./s04-sysdig-sage-design.md)
> - [#5 AIで守る② - Agentic Cloud Security](./s05-agentic-cloud-security.md)
> - [#6 総括 - Sysdig の戦略的ポジション](./s06-sysdig-strategy.md)

---

## はじめに

前回の記事（[#1 総論](./s01-ai-framework-overview.md)）では、Sysdig の AI 戦略を「AIを守る × AIで守る」の二軸で整理しました。

この記事では **軸①「AIを守る」** の核心である **AI Workload Security** を深掘りします。

「AI のセキュリティ」と聞くと、プロンプトの安全性や LLM の倫理的問題をイメージしがちです。
しかしここで扱うのは、**AI がクラウドネイティブなワークロードとして稼働する際のインフラレベルのリスク** です。

---

## AI 急速導入がもたらすリスクの実態

2023年以降、世界で **6,600万以上の新規 AI プロジェクト** が誕生しています。
この爆発的な速度の裏側で、セキュリティ対策が後回しになっています。

Sysdig が観測している現実：

> **34% の GenAI ワークロードがインターネットに公開された状態**

これは「意図して公開しているもの」だけではありません。
設定ミス・デフォルト設定の放置・アクセス制御の漏れが混在しています。

<!-- スクリーンショット: Sysdig の Cloud-Native Security Report または統計グラフ（34%公開状態など） -->
![AI Workload Exposure Statistics](./images/ss-s02-01-exposure-stats.png)
*GenAI ワークロードのリスク統計。公開状態にある AI パッケージの実態*

---

## AI ワークロードが持つ固有のリスク3類型

Sysdig Threat Research チームが特定した、AI ワークロード特有のリスクが3つあります。

### 1. プロンプトインジェクション

```
攻撃者が工作したプロンプトを入力
    ↓
モデルの「意図された機能」を迂回
    ↓
機密情報へのアクセス or 有害なコマンド実行
```

通常の SQL インジェクションと本質的に同じ原理ですが、
制御境界がコードではなく **自然言語** であるため、従来のバリデーションが機能しません。

### 2. 敵対的攻撃（Adversarial Attack）

言語モデルを混乱させる入力を意図的に設計し、モデルに不正な出力を生じさせる手法です。

```
軽微な操作：誤った回答の生成
重大な操作：機密データの漏洩 / 意図しないコマンド実行
```

### 3. トロイの木馬型 LLM 汚染

```
訓練データに悪意あるトリガーを埋め込む
    ↓
通常の利用ではまったく問題なく動作する
    ↓
特定の入力パターンが来た瞬間に活性化
    ↓
データ漏洩 or 不正コマンド実行
```

サプライチェーン攻撃のモデル版です。
公開されているファインチューニング済みモデルを使う際のリスクとして実在します。

| リスク類型 | 主な対策 | CNAPP でできること |
|---|---|---|
| プロンプトインジェクション | 入力バリデーション・サニタイゼーション | 異常なプロセス挙動の検知 |
| 敵対的攻撃 | モデルのロバスト性強化 | 不審な API アクセスパターンの検知 |
| LLM 汚染 | AIBOM による依存追跡 | ワークロード内パッケージの継続監視 |

---

## Sysdig AI Workload Security：機能の全体像

Sysdig は上記のリスクに対し、**既存の CNAPP フレームワークに AI 保護を統合**する形で回答しています。
新しいツールを導入するのではなく、すでに使っているプラットフォームの中で AI を守れる、という設計です。

<!-- スクリーンショット: Sysdig Secure の AI Workload Security ダッシュボード全体 -->
![AI Workload Security Dashboard](./images/ss-s02-02-dashboard.png)
*AI Workload Security ダッシュボード：AI パッケージのインベントリと公開リスクが一元管理される*

### 機能① 自動インベントリと可視化

**AI パッケージ・モデルを自動検出**し、ワークロード全体のインベントリを構築します。

- OpenAI / TensorFlow / PyTorch 等の AI エンジンの自動検出
- 公開されている AI パッケージの識別
- どのコンテナ・Pod に何の AI コンポーネントが存在するかの可視化

<!-- スクリーンショット: AI パッケージのインベントリ一覧（コンテナ名・AIパッケージ名・バージョン・公開状態） -->
![AI Package Inventory](./images/ss-s02-03-inventory.png)
*AI パッケージインベントリ：環境内の全 AI コンポーネントを自動一覧化*

### 機能② Risk Prioritization との統合

単なる検出で終わらず、**他のリスク要素と相関させてスタックランク化**します。

```
AI ワークロードのリスク評価に使われる要素：
  ├─ CVE（既知の脆弱性）
  ├─ インターネット公開状態
  ├─ ランタイムイベント（実際の挙動）
  └─ Attack Path Analysis（攻撃経路）
```

<!-- スクリーンショット: Risk Prioritization ページ（AI ワークロードが上位にランクされている状態） -->
![Risk Prioritization - AI Workloads](./images/ss-s02-04-risk-prioritization.png)
*Risk Prioritization：CVE・公開状態・ランタイムを統合してリスクをスタックランキング*

### 機能③ Attack Path Analysis

Cloud Attack Graph を使い、**AI ワークロードへの攻撃経路を可視化**します。

```
インターネット
     ↓（公開ポート）
AI サービス Pod
     ↓（過剰な権限）
クラウドストレージ（モデルウェイト・訓練データ）
     ↓
機密データの外部流出
```

このような連鎖を Attack Path として描き出すことで、
「どのリスクを先に塞ぐべきか」が直感的にわかります。

<!-- スクリーンショット: Attack Path Analysis で AI ワークロードへの攻撃経路が可視化されている画面 -->
![Attack Path - AI Workload](./images/ss-s02-05-attack-path.png)
*Attack Path Analysis：AI ワークロードを起点とした攻撃連鎖を視覚化*

---

## AIBOM：AI コンポーネントの構成管理

**AIBOM（AI Bill of Materials）** は、SBOM（ソフトウェア部品表）の AI 版です。

```
SBOM：アプリケーションを構成するソフトウェアの一覧
AIBOM：AI システムを構成するコンポーネントの一覧
  ├─ 使用モデル（名前・バージョン・出所）
  ├─ トレーニングデータソース
  ├─ AI フレームワーク・ライブラリ
  └─ API / 外部 AI サービス依存
```

AI ワークロードの透明性確保と、LLM 汚染リスクの追跡に不可欠な概念です。

---

## 規制対応：EU AI Act と米国 Executive Order

AI Workload Security は技術的な保護だけでなく、**コンプライアンスの側面**も担います。

| 規制 | 主な要件 | Sysdig での対応 |
|---|---|---|
| **EU AI Act** | ハイリスク AI の透明性・記録保持 | AIBOM + インベントリ |
| **米国 Executive Order** | AI 安全性・信頼性の評価 | Risk Prioritization |
| **各国のデータ保護法** | 訓練データの管理 | AIBOM + ランタイム監視 |

「AI を使う」ことが組織として証明可能な形で管理されているか。
これが問われる時代になっています。

---

## まとめ

AI ワークロードのリスクは、従来の「コンテナのセキュリティ」の延長にあります。
ただし守る対象に新しい層が加わった。

```
従来のリスク：    CVE / 設定ミス / 不審な挙動
AI 固有のリスク：  + プロンプトインジェクション
                  + 敵対的攻撃
                  + LLM 汚染
                  + モデルデータの公開露出
```

Sysdig AI Workload Security は、この新しい層を **既存の CNAPP の中** で処理することで、
別ツール導入のコストなしに AI 時代の保護を実現します。

次の記事では、AI を活用するインフラとして急拡大している **MCP Server** が
なぜ新たなセキュリティリスクになるのかを解説します。

---

## 参考リンク

- [AI Workload Security for CNAPP](https://www.sysdig.com/blog/ai-workload-security-for-cnapp)
- [AI Workload Security for AWS](https://www.sysdig.com/blog/ai-workload-security-for-aws)
- [The risks of rapid AI adoption](https://www.sysdig.com/blog/sysdigs-ai-workload-security-the-risks-of-rapid-ai-adoption)
- [5 Steps to Securing AI Workloads](https://www.sysdig.com/blog/5-steps-to-securing-ai-workloads)
- [A practical guide to securing AI workloads](https://www.sysdig.com/blog/ai-is-still-a-workload-a-practical-guide-to-securing-ai-workloads)
