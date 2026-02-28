# Platform Engineeringとは
## ― 開発者を"本来の仕事"に集中させるための設計思想

---

## はじめに ― なぜ今、Platformなのか

「Kubernetesの設定、CI/CDの構築、IAM設計、セキュリティ対策、監視設定…」

クラウドネイティブの世界では、開発者はコードを書く前に大量の"準備作業"を強いられます。

- クラスタのnamespace設計
- RBACの細かな調整
- Terraformのモジュール理解
- イメージスキャンの例外処理
- セキュリティレビュー対応
- ログ転送の設定
- 脆弱性の優先度判断

その結果どうなるか？

| 症状 | 影響 |
|---|---|
| 開発速度が落ちる | リリースサイクルの長期化 |
| セキュリティが形骸化する | インシデントリスクの増大 |
| チームごとに設計が分裂する | 再利用不可・サポート困難 |
| 監査対応が地獄になる | コンプライアンスコストの爆発 |

これは「個人の努力」で解決する問題ではありません。

**構造の問題です。**

それを解決するアプローチが **Platform Engineering** です。

---

## Platform Engineeringの定義（構造で理解する）

> 複雑なクラウド基盤を抽象化し、
> 開発者向けの"内部プロダクト"として提供する設計思想

ここで重要なのは：

- Platformは単なるインフラではない
- Platformは**社内向けプロダクト**
- PlatformにはUXがある
- Platformにはロードマップがある
- Platformには利用者（開発者）がいる

つまり：

> **Platformチーム = プロダクトチーム**

という構造です。

---

## 抽象化レイヤー

```
[ Developer ]
      ↓
[ Internal Developer Platform (IDP) ]
      ↓
[ Cloud / Kubernetes / Security / CI/CD ]
```

開発者は下層を意識しません。

IDP（Internal Developer Platform）が：

- 設定を標準化し
- セキュリティを組み込み
- 再現性を担保する

> 抽象化は、自由を奪うことではありません。
> **認知負荷を下げることです。**

---

## Before / After ― 構造的な変化

### Before（非構造化状態）

- チームごとにTerraformの書き方が違う
- RBAC設計が属人化
- CI/CDがプロジェクト単位でバラバラ
- セキュリティ例外が口頭承認
- 監査ログの保存期間が不統一

**結果：**

- 組織がスケールしない
- レビュー負荷が増大
- セキュリティ事故が発生
- オンボーディングに時間がかかる

### After（Platform導入後）

Platformが提供するもの：

| 機能 | 効果 |
|---|---|
| 標準化テンプレート | 再現性 |
| 自動ガードレール | 安全性 |
| 統合セキュリティ | 一貫性 |
| GitOpsベースの配布 | 監査容易性 |
| セルフサービス申請 | 速度・スケール耐性 |

---

## DevOpsとの違い（より深く）

DevOpsは**文化**です。

- サイロを壊す
- 協力を促す
- 早く届ける

しかし、文化だけでは構造は変わりません。

| | DevOps | Platform Engineering |
|---|---|---|
| 性質 | 文化・思想 | 実装モデル |
| 主眼 | 開発と運用の協力 | Platformをプロダクトとして提供 |
| 体制 | チーム連携 | Platformチームが明確に存在 |

> DevOpsが「思想」なら、
> **Platform Engineeringは「実装」。**

---

## セキュリティを組み込む（後付けにしない）

多くの組織での現実：

```
アプリ開発 → デプロイ → セキュリティチェック（最後）
```

**Platformでは、セキュリティは最初から組み込まれる。**

| 領域 | 実装例 |
|---|---|
| コンテナ | イメージの自動スキャン |
| ランタイム | 異常の自動検知 |
| 脆弱性管理 | 優先度付き可視化 |
| ポリシー | 違反の自動ブロック |

---

## 具体例：FalcoをPlatform化する

ランタイムセキュリティの [Falco](https://falco.org/) を例にします。

```
Falco Rule
    ↓ Git管理（変更の透明性確保）
PRレビュー
    ↓ 構文・論理ミスを自動検出
CI検証
    ↓ 全クラスタへ自動展開
GitOps配布（ArgoCD等）
    ↓ 申請→承認→記録
例外申請もコード化
```

セキュリティは"摩擦"ではなくなります。**自然に流れる。**

---

## Security組み込み型Platformの構造

```
[ Developer ]
      ↓
[ IDP ]
      ↓
[ Security Layer (built-in) ]
      ↓
[ Infrastructure ]
```

> セキュリティは横から差し込むものではない。
> **下層に常設される。**

---

## CNAPPとの接続

[Sysdig](https://sysdig.com/) のようなCNAPP（Cloud-Native Application Protection Platform）は以下を統合します：

- Runtime Protection
- Vulnerability Management
- CSPM（Cloud Security Posture Management）
- CIEM（Cloud Infrastructure Entitlement Management）
- Container Scanning

Platform Engineeringの視点では：

> **CNAPPはGuardrailエンジンです。**

IDPの下で：

- 標準ポリシーを適用
- 脆弱性を優先度付きで可視化
- リスクをコードに還元

**Security becomes default.**

---

## Golden Path（組織設計の核心）

> 組織として最善の方法を"デフォルト"にすること

```
❌ 自由にやっていいよ
  → 設定がバラバラになる
  → セキュリティポリシーが崩壊する
  → 障害対応が属人化する

⭕ このテンプレートを使って
  → 安全・高速・再現性あり
  → レビューコストが下がる
  → 新メンバーもすぐ動ける
```

> 自由を奪うのではない。
> **最善手を簡単にする。**

---

## Platform Engineeringの本質

Platform Engineeringが実現するもの：

| 領域 | 効果 |
|---|---|
| 認知負荷の削減 | 開発者がコアに集中できる |
| セキュリティの民主化 | 全員が"自然に"安全な選択をする |
| 抽象化による加速 | 複雑さを隠蔽してスピードを出す |
| 再現性の提供 | どのチームでも同じ品質 |
| 組織スケールの設計 | 人が増えても品質が落ちない |
| レビューコスト削減 | 標準化で差分レビューが軽くなる |
| 監査対応の自動化 | ログ・証跡が自動で揃う |
| 新人オンボーディング短縮 | Golden Pathを辿るだけで動く |
| 障害対応の標準化 | Runbookが統一される |
| 技術負債の構造的圧縮 | 散在する設定を一元管理 |

---

## まとめ

優れたPlatformは：

- 開発者体験（DX）を改善し
- セキュリティを自然に組み込み
- スピードと統制を両立させる

そして最終到達点はここです。

---

> 「セキュリティを止める存在」から
> **「セキュリティが加速装置になる世界」へ**

---

Platformは単なる技術論ではありません。

それは、

**組織設計の戦略そのものです。**

---

*参考リンク*

- [CNCF Platforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/)
- [Team Topologies](https://teamtopologies.com/)
- [Falco](https://falco.org/)
- [Sysdig CNAPP](https://sysdig.com/)
