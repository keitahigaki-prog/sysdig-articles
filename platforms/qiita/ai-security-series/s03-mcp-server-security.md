# AIを守る②：MCP Server のセキュリティリスク
〜AI 連携ツールがもたらす新攻撃面とエンタープライズの対策〜

> **シリーズ一覧**
> - [#1 総論：二軸フレームワーク](./s01-ai-framework-overview.md)
> - [#2 AIを守る① - AI Workload Security](./s02-securing-ai-workloads.md)
> - **#3 AIを守る② - MCP Server のリスク**（この記事）
> - [#4 AIで守る① - Sysdig Sage™ の設計思想](./s04-sysdig-sage-design.md)
> - [#5 AIで守る② - Agentic Cloud Security](./s05-agentic-cloud-security.md)
> - [#6 総括 - Sysdig の戦略的ポジション](./s06-sysdig-strategy.md)

---

## はじめに

前回の記事（[#2](./s02-securing-ai-workloads.md)）では、AI ワークロード自体のリスクを扱いました。

この記事では「AIを守る」の第2弾として、**MCP（Model Context Protocol）** というレイヤーを扱います。

MCP は AI を「使いやすくする」技術です。
しかし同時に、**従来のセキュリティモデルでは想定していなかった新しい攻撃面** を生み出しています。

:::note info
**MCP（Model Context Protocol）とは**
Anthropic が 2024年11月に公開したプロトコル。LLM（大規模言語モデル）が外部ツール・データソース・APIと標準化された方法で連携できるようにする仕組み。
ChatGPT・Claude などの AI が、MCP を介して外部システムを自律的に操作できるようになる。
OpenAI・Google DeepMind をはじめ主要 AI プロバイダーが対応済み。
:::

---

## MCP が生み出す「新しいリスクの構造」

### 従来の API との根本的な違い

従来の API 連携と MCP の違いを理解することが重要です。

```
従来の API 連携：
  人間 → コード（決定論的） → API 呼び出し
  ↑ 何が起きるかは事前に決まっている

MCP 連携：
  人間 → LLM（確率論的）→ 自律的にツールを選択・実行
  ↑ 何が起きるかは LLM の判断次第
```

これが根本的な違いです。

> **MCP は「確定的な操作」ではなく「自律的な判断」を LLM に委ねる。**
> **リクエストが常に検証されるわけではない。**

Sysdig はこれを「**レガシーセキュリティモデルでは対応できない爆発的な被害範囲**」と表現しています。

<!-- スクリーンショット: MCP Server の構成図またはアーキテクチャダイアグラム -->
![MCP Architecture](./images/ss-s03-01-mcp-architecture.png)
*MCP の構成：LLM が MCP Server を介して外部ツールを自律的に操作する*

---

## 実際に起きているリスク：3つの攻撃カテゴリ

### 1. LLMjacking

```
攻撃者が LLM へのアクセス権を不正取得
    ↓
コード生成・ソーシャルエンジニアリングに悪用
    ↓
組織のシステムへの侵入 or 詐欺
```

MCP 経由で LLM が多くのツールと連携するほど、
LLMjacking の影響範囲が拡大します。

### 2. 認証情報漏洩

実際の事例として Sysdig が挙げているデータ：

| 事例 | 内容 |
|---|---|
| **DeepSeek データベース露出** | 設定ミスにより数百万のチャットログと API キーが公開 |
| **LLM トレーニングデータ汚染** | 12,000件以上の API キー・パスワードがトレーニングデータに混入 |

MCP Server は認証情報を扱う頻度が高く、これが外部に漏れるリスクは無視できません。

### 3. プロンプトインジェクション経由のツール操作

```
攻撃者が工作したコンテンツを外部から挿入
    ↓
LLM がそれをプロンプトとして誤認識
    ↓
意図しない MCP ツール実行（ファイル削除・データ送信等）
```

例えば、メールの本文に隠されたインジェクション文字列が、
AI アシスタント経由で Slack への送信や Jira チケット作成を引き起こすシナリオが実在します。

<!-- スクリーンショット: Sysdig の脅威検知画面（MCP 経由の不審なアクション検知アラート）-->
![MCP Security Alert](./images/ss-s03-02-mcp-alert.png)
*MCP 経由の不審なツール操作を Sysdig が検知したアラートの例*

---

## 攻撃が成立しやすい3つの条件

Sysdig が分析する「MCP リスクが爆発する組み合わせ」：

```
① 脆弱性（CVE / 設定ミス）
      +
② 弱い認証（長命なトークン・MFA なし）
      +
③ 広範な認可（過剰な権限を持つ MCP Server）

= レガシーモデルでは対処不能な被害範囲
```

これが従来の WAF・SIEM・EDR では防ぎにくい理由です。
攻撃の「実行主体」が人間ではなく LLM になっているからです。

---

## Sysdig が推奨する4つの対策柱

### 柱① 認証と認証情報管理

```yaml
# 推奨：短命なトークンの使用
token_lifetime: 3600  # 1時間で失効
rotation: enabled
mfa: required

# 監視すべき項目
- トークンの不審な多重使用
- 通常外の時間帯でのアクセス
- 地理的に異常なアクセス元
```

長命なトークンはリスクの元凶です。MCP では特に注意が必要です。

### 柱② 入力検証・プロンプト制御

```
入力サニタイゼーション
    ├─ 許可リスト（allowlist）の活用
    ├─ 拒否リスト（denylist）の管理
    └─ 異常なプロンプトパターンの監視

一部の組織では：
既知の悪意あるクエリを削除するプロキシを
通じて LLM リクエストをルーティングしている
```

### 柱③ 細粒度の認可・コンテキスト分離

MCP は認可に歴史的な課題があります。

```
悪い例：MCP Server に全リソースへのアクセス権
  └─ 一度侵害されると全データが危険

良い例：最小権限原則の徹底
  ├─ 読み取り専用 MCP vs 書き込み可能 MCP を分離
  ├─ スコープを用途ごとに制限
  └─ ロールベースアクセス制御の適用
```

<!-- スクリーンショット: Sysdig Secure の権限設定またはロールベースアクセス管理画面 -->
![Least Privilege Configuration](./images/ss-s03-03-least-privilege.png)
*最小権限原則の実装：MCP Server のスコープを用途ごとに絞る*

### 柱④ 継続的な監視とリアルタイム対応

```
監視すべき項目（Falco ルール例）：
  ├─ 通常外のファイルアクセス（/etc/passwd, .ssh/）
  ├─ 予期しない外部ネットワーク接続
  ├─ 大量データの転送
  └─ 特権昇格の試み

定期的な実施：
  └─ レッドチーミング（MCP 経由の攻撃シミュレーション）
```

<!-- スクリーンショット: Falco による MCP 関連の不審挙動検知アラート一覧 -->
![Falco Monitoring for MCP](./images/ss-s03-04-falco-mcp.png)
*Falco によるリアルタイム監視：MCP 経由の異常挙動を syscall レベルで検知*

---

## これは「技術的問題」ではなく「経営戦略の問題」

Sysdig が強調している点があります。

> **「このリスクを技術的問題ではなく、経営戦略の中核として扱うことを経営陣に求める」**

なぜか。

MCP は **ビジネス部門主導で導入が進む** ケースが多いからです。
「AI アシスタントを便利にする設定」として、セキュリティチームの関与なしに設定されることがあります。

この状況は、かつての **シャドー IT** と同じ構造です。

```
かつて：          今：
クラウドサービスを  MCP Server を
セキュリティチームに セキュリティチームに
無断で利用         無断で設定・連携
```

---

## まとめ

MCP は AI の利便性を飛躍的に高める技術です。
しかし同時に、**従来のセキュリティモデルでは想定していなかった攻撃面** を生み出しています。

| 従来の攻撃 | MCP 時代の攻撃 |
|---|---|
| 人間がコードを実行 | AI が自律的にツールを実行 |
| 意図が明確 | 意図が確率論的 |
| ログで追跡可能 | 実行経路が複雑 |
| 境界が明確 | LLM の判断が境界 |

次の記事からは視点を切り替えて、**軸②「AIで守る」** に入ります。
最初に取り上げるのは、Sysdig が「チャットボットではない」と位置づける **Sysdig Sage™** です。

---

## 参考リンク

- [Sysdig MCP Server: Bridging AI and cloud security insights](https://www.sysdig.com/blog/sysdig-mcp-server-bridging-ai-and-cloud-security-insights)
- [Why MCP server security is critical for AI-driven enterprises](https://www.sysdig.com/blog/why-mcp-server-security-is-critical-for-ai-driven-enterprises)
- [Shifting left with AI and MCP: Sysdig + Amazon Q Developer](https://www.sysdig.com/blog/shifting-left-with-ai-and-mcp-sysdig-amazon-q-developer)
- [AI echolocation of cloud risks using Sysdig and Snyk MCP servers](https://www.sysdig.com/blog/ai-echolocation-of-cloud-risks-using-sysdig-and-snyk-mcp-servers)
