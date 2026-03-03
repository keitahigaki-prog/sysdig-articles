# OWASP「Secure MCP Server Development」完全解説 ― AI Agent時代の実行基盤セキュリティ設計

## はじめに

2026年2月、OWASPはGenAI Security Projectの一環として **「A Practical Guide for Secure MCP Server Development v1.0」** を公開した。

このガイドが示す最大のメッセージは一つだ。

**AIアプリケーションは、従来のWebセキュリティでは守れない。**

MCP（Model Context Protocol）は単なるAPIラッパーではない。LLMとツール・外部システムをつなぐ「実行基盤」であり、その三層構造が従来にない攻撃面を生んでいる。

本記事ではPDF全17ページを精読したうえで、背景・脅威・設計原則・実装要件を体系的に解説する。セキュリティ担当者だけでなく、**ソフトウェアアーキテクト・プラットフォームエンジニア・開発チームリード**が読むべき内容として構成した。

---

## 対象読者と前提

- MCPサーバーを設計・開発・運用する人
- AIエージェント基盤のセキュリティレビューを行う人
- LLMアプリケーションのリスク評価をしたい人

MCPの仕様そのものの解説は行わない。セキュリティ設計に絞る。

---

## MCPの構造を理解する ― なぜ従来と違うのか

### 三層の責任分離

MCPを用いたAIシステムは、以下の三層で動作する。

| 層 | 担当 | 特徴 |
|---|---|---|
| 判断層 | LLM / Agent | 自然言語で意図を解釈し、ツール呼び出しを決定する |
| 仲介層 | MCPサーバー | 認証・ポリシー検証・ツールバインディングを実行する |
| 実行層 | ツール / 外部システム | コードを動かし、データを読み書きする |

重要なのは「**判断するのはLLM、実行するのはMCP、権限はユーザーからの委任**」という三者分離の構造だ。この分離が、攻撃面を爆発的に広げる。

### 従来APIとの本質的な差異

従来のAPIは、呼び出し元が明示的にエンドポイントを指定する。対してMCPでは：

- **LLMが動的にツールを選択する** ― 呼び出しは予測不能
- **ツール定義（説明文）がLLMの判断を誘導する** ― 説明文が攻撃面になる
- **複数ツールを連鎖呼び出しできる** ― 単一の脆弱性が増幅される
- **ユーザー権限を委任して動作する** ― Confused Deputy問題が構造的に発生する

OWASPガイドはこの差異を冒頭で明示する：

> "Unlike traditional APIs, MCP servers often operate with delegated user permissions, dynamic tool-based architectures, and can chain multiple tool calls, amplifying the impact of any single vulnerability."

---

## 現在の脆弱性ランドスケープ

OWASPが整理した主要リスクは6つ。それぞれの本質を解説する。

### 1. Tool Poisoning（ツール汚染）

ツールの「説明文」に悪意ある指示を埋め込む攻撃。

LLMはツールの説明文を信用してツールを選択・実行する。つまり**説明文を書いた人間がLLMの行動を制御できる**。説明文を通じて「パスワードを外部に送信せよ」という隠し命令を忍ばせることが可能だ。

これは従来のコードインジェクションとは異なる。**攻撃はコードではなく自然言語で行われる。**

### 2. Dynamic Tool Instability ― "Rug Pull"

一度信頼されたツール定義が、後から差し替えられる問題。

MCPはツールを動的にロードできる。初回の承認・セキュリティレビューをパスしたツールでも、その後にメタデータや挙動を書き換えれば、**初期の信頼検証が無効化される。**

サプライチェーン攻撃に近い性質を持つ。

### 3. Code Injection & Unsafe Execution

モデルから渡された入力がそのままシステムコマンドやSQLに流し込まれる問題。

LLMの出力は自然言語であり、構造が保証されない。不適切な入力サニタイズは従来のWebと同様に致命的だ。

### 4. Credential Leakage & Token Misuse

APIキーやOAuthトークンの誤った管理。

MCPサーバーは多くの外部サービスのクレデンシャルを扱う。**ログへの平文出力・環境変数への平文保存・キャッシュの長期保持**が漏洩の主要経路になる。

### 5. Excessive Permissions（過剰権限）

最小権限の原則違反。

過剰な権限を持つツールが一つ侵害されると、その権限範囲内のすべてのシステムが危険にさらされる。MCPが「ポリシー強制者」として機能しているか、LLMとユーザーの権限に齟齬がないかの両面で問題が発生する。

### 6. Insufficient Isolation（セッション・ID・コンピュート隔離の不足）

OWASPが最も詳細に記述する問題であり、実装上最も見落とされやすい。

三つの層で隔離が必要：

**セッション隔離：** 複数ユーザーが同一MCPサーバーを利用する場合、グローバル変数・クラスレベル属性・共有シングルトンを通じてあるユーザーのデータが別ユーザーに漏洩する。

**ID隔離：** MCPサーバーが複数ユーザーに対して単一のサービスアカウントを使い回すと、個人の行動を監査で追跡できなくなる。

**コンピュート隔離：** 共有プロセス環境でコードを実行すると、悪意あるテナントが他テナントのプロセスメモリにアクセスするか、リソースを独占できる。

---

## 1. Secure MCP Architecture ― アーキテクチャ設計原則

### ローカル接続とリモート接続の使い分け

MCPは2種類の通信チャネルを持つ。

**ローカルMCPサーバー（STDIOまたはUnixソケット）：**

- ネットワークソケットより安全なSTDIOまたはUnixソケットを優先
- HTTPを使わざるを得ない場合は`127.0.0.1`のみにバインド
- Originヘッダーを明示的に検証する
- 外部ネットワークアクセスを持たない隔離されたサブプロセス・コンテナで実行

**リモートMCPサーバー（Streamable HTTP）：**

- TLS 1.2以上を全通信に強制
- JSON-RPCメッセージをMCPスキーマに対して厳密にバリデート
- 不正形式・未知フィールドを含むデータは即座に拒否

### クライアント認証

静的な関係（既知のクライアントのみ）では：
- Allowlist・ハードコード接続・相互TLS（mTLS）を使用

動的な環境では：
- OAuth 2.1またはOIDCでクライアントIDを動的に検証

**「ネットワーク上の位置」だけを信頼してはならない。** IDを検証する。

### ユーザー・セッション分離

ガイドの核心的な要件の一つ。実装上の具体的な指針が記されている：

```
禁止：グローバル変数・クラスレベル属性・共有シングルトンへのユーザーデータ格納
必須：セッションごとのオブジェクトインスタンス化
      または session_id namespaceを持つRedisのようなセッションキー型ステートストア
```

### Strict Lifecycle Management（厳格なライフサイクル管理）

これはガイドが明示する具体的な実装要件で、見落とされがちだ。

> "When an MCP session terminates, disconnects, or times out, all associated file handles, temporary storage, in-memory contexts, and cached tokens are **immediately flushed and destroyed** to prevent residual data exposure."

セッション終了は「片付け」ではなく**セキュリティ要件**だ。決定論的なクリーンアップルーティンを実装し、残存データによる情報漏洩を防ぐ。

### Per-Session Resource Quotas

リソース消費はセッションIDまたはユーザーIDに紐付けて制限する：

| リソース種別 | 制限対象 |
|---|---|
| メモリ | セッションあたりの上限を設定 |
| CPU | バジェット制御 |
| ファイルシステム | 読み書きサイズ・件数の制限 |
| API呼び出し | レートリミット |

---

## 2. Safe Tool Design ― ツール設計の安全性

### Cryptographic Tool Manifests（暗号署名付きマニフェスト）

**すべてのツールは署名付きマニフェストを持たなければならない。**

マニフェストに含めるべき情報：

- ツールの説明文
- 入出力スキーマ
- バージョン番号
- 必要な権限リスト

**ロード時にシグネチャとハッシュを検証する。** 検証を通過しないツールは実行しない。これがRug Pull攻撃への根本的な対策になる。

### Strict Onboarding & Approval（厳格なオンボーディング）

新規ツールや既存ツールの更新には以下のすべてを通過させる：

| チェック種別 | 内容 |
|---|---|
| SAST | 静的コードスキャン |
| Dynamic Testing | 動的テスト |
| SCA | ソフトウェアコンポジション分析（依存関係脆弱性） |
| Manual Security Review | 人間によるセキュリティレビュー |

### Validate Descriptions vs. Behavior（説明文と実挙動の差異検出）

**ツールの説明文に書かれていない動作をツールが実行しようとした場合、フラグを立てる。**

例：説明文に「ファイルを読む」と書かれているのに、実行時にネットワーク通信を試みる場合は異常として検出する。

ツールピニング（特定バージョンへの固定）とLLMスキャン（説明文の内容を別LLMで審査）を組み合わせる。

### Tool Structure Validation（ツール構造バリデーション）

**モデルに公開するフィールドは最小限にする。**

内部メタデータ・センシティブフィールドはモデルのコンテキスト外に置く。LLMが「知る必要がない」情報はLLMに渡さない。

---

## 3. Data Validation & Resource Management ― データ検証とリソース管理

### すべてのデータを「信頼しない」

LLMからの入力も、ツールからの出力も、外部システムからのレスポンスも、すべて**untrusted（信頼しないもの）**として扱う。

### JSON Schemaによる厳密なバリデーション

```
ツールへの入力（モデルから）  → JSONスキーマで検証・スキーマ不一致は即時拒否
ツールからの出力（モデルへ）  → JSONスキーマで検証・サイズ制限を適用
```

スキーマに合致しないリクエストは処理しない。エラーメッセージも安全な形式にとどめる（詳細は後述）。

### Sanitization & Encoding（サニタイズとエンコーディング）

古典的なコードインジェクション攻撃はMCP環境でも有効だ：

- XSS（ツールが返したHTMLがUIにレンダリングされる場合）
- SQLインジェクション（ツールがDBクエリを実行する場合）
- RCE（ツールがシェルコマンドを実行する場合）

モデルへの入力・ツールへの出力の両方向でエスケープシーケンスを除去する。

### Resource Usage Limits（リソース使用制限）

- ツール呼び出し回数のレートリミット（セッションあたり）
- データフェッチ量の上限
- タイムアウト設定（暴走プロセスの防止）
- 隔離されたメモリ・コンピュートバジェット

---

## 4. Prompt Injection Controls ― プロンプトインジェクション対策

### なぜ従来の入力検証では防げないのか

Prompt Injectionは**SQLインジェクションでもXSSでもない。**

攻撃は自然言語で行われる。「ignore previous instructions」「あなたはいまから別のアシスタントです」のような文字列には、従来のサニタイズが機能しない。**意味的な解析が必要であり、それは構造的には非常に難しい。**

OWASPガイドが示す対策は、サニタイズ中心ではなく**設計で封じ込める**アプローチだ。

### Structured Tool Invocation（構造化ツール呼び出し）

**モデルが自由形式のテキストコマンドを生成するより、JSON構造のツール呼び出しを優先する。**

モデルの意図をスキーマで定義された形式に通すことで、攻撃者がコンテキストに忍ばせた任意の命令が直接実行されるリスクを低減する。

### Human-in-the-Loop（HITL）

**高リスクアクションには人間の承認ステップを挿入する。**

HITL対象の典型例：

- データの削除
- 金銭の送金
- システムレベルの設定変更

MCP Elicitations（MCP仕様における確認要求機能）を使い、MCPクライアント側で明示的な確認をユーザーに求める。

### LLM-as-a-Judge（審査LLM）

これはガイドが提示する中でも特に重要な設計パターンだ。

> "For high-risk actions, run a dedicated approval check in a **distinct context LLM session**, using a policy prompt that defines which tool calls and parameters are allowed and which are blocked."

**高リスクなツール呼び出しに対して、別のLLMセッションが「このツール呼び出しは許可されるか」を審査する。**

通常のLLMセッションとは独立したコンテキストで実行されるため、そのセッションに対するPrompt Injectionの影響を受けない。HITLが不可能な高速・自動化シナリオでの現実解として機能する。

### One Task, One Session（タスクとセッションの1対1対応）

> "Reset MCP sessions when an agent switches contexts or tasks. This 'context compartmentalization' prevents hidden instructions from persisting in a long conversation history."

エージェントがタスクを切り替えるたびにMCPセッションをリセットする。長い会話履歴に蓄積した隠し命令が次のタスクに引き継がれるのを防ぐ。副次効果として「コンテキスト劣化」（長すぎる会話によるLLM品質低下）も抑制する。

---

## 5. Authentication & Authorization ― 認証・認可

### OAuth 2.1 / OIDCの強制

すべてのリモートMCPサーバーでOAuth 2.1 / OIDCを必須とする。

各リクエストで以下を検証する：

| 項目 | 内容 |
|---|---|
| `iss` | トークン発行者 |
| `aud` | 対象オーディエンス |
| `exp` | 有効期限 |
| Signature | 署名の正当性 |

### Token Delegation（トークン委任）と Token Passthrough 禁止

これはガイドの中核原則の一つ。**Token Passthrough（トークン直接転送）は明示的に禁止されている。**

**禁止パターン（Token Passthrough）：**

クライアントから受け取ったトークンをそのままダウンストリームAPIに転送する。

**この問題の本質：**
- 監査証跡が途切れる（MCPサーバーを経由した操作が追跡できなくなる）
- ポリシーをバイパスできる
- Confused Deputy攻撃が成立する

> "This vulnerability allows the MCP server to be tricked into misusing a user's privileges to execute unauthorized actions, effectively bypassing the server's intended security constraints."

**正しいパターン（Token Delegation）：**

RFC 8693（OAuth 2.0 Token Exchange）に準拠したフローを使用。

処理の流れ：
1. ユーザーのトークンをMCPサーバーが受け取る
2. MCPサーバーがスコープ制限・有効期限短縮した**サーバー専用トークン**に交換する
3. そのサーバー専用トークンをダウンストリームAPIに渡す

これにより：
- MCPサーバーのIDが常に識別可能
- 権限スコープが最小限に制御される
- 監査ログで全操作を追跡できる

### Short-Lived, Scoped Tokens（短命・スコープ制限トークン）

- アクセストークンの有効期限は**数分単位**で設定
- 各呼び出しの前にトークンのシグネチャ・オーディエンス・有効期限を**再検証**
- スコープは実行するツール・リソースに必要な最小限に絞る

### Sessions as State, Not Identity（セッションIDは認証に使わない）

セッションIDだけで認証・認可を行ってはならない。

```
すべてのセッション / ストリーム / キューエントリを
検証済みのユーザーIDとクライアントID（OAuth）に紐付ける
センシティブ操作の前に認可を再チェックする
トークンを攻撃的にローテーション・失効させる
```

### Centralize Policy Enforcement（集中ポリシー強制）

認証・認可・同意・ツールフィルタリング・監査ログを一元化されたポリシー/ゲートウェイレイヤーで処理する。

エージェント・サーバー・上流サービスにまたがって、ポリシーが一貫して適用される状態を保つ。

---

## 6. Secure Deployment & Updates ― セキュアデプロイ

### Secrets Storage and Management（秘密情報管理）

| 禁止事項 | 理由 |
|---|---|
| 環境変数への秘密情報格納 | コンテナ起動時のデバッグログやkubectl describeで露出する |
| ログへの平文出力 | ログ集約システムが漏洩経路になる |
| コード内ハードコード | VCSに残る |
| LLMへの秘密情報の直接アクセス許可 | コンテキストを通じて漏洩する |

**Vault（HashiCorp VaultやAWS Secrets Manager等）を使い、秘密管理機能はLLMからアクセスできない透明なミドルウェアで実行する。**

### Containerize and Harden（コンテナ化とハードニング）

- **非rootユーザーで実行する**
- 不要なコンテナパッケージ・依存関係・Linuxケーパビリティを削除
- 最小構成のベースイメージを使用

### Network Segmentation（ネットワーク分離）

MCPサーバーは制限されたネットワークセグメントに配置する。

ファイアウォールルールまたはKubernetes NetworkPoliciesを使用し、**明示的に必要なインバウンド・アウトバウンドトラフィック以外はすべてブロックする。**

### Supply Chain Controls（サプライチェーン管理）

| 対策 | 内容 |
|---|---|
| バージョンピニング | すべての依存関係をビルド時に特定バージョンに固定 |
| 署名済みコンテナイメージ | イメージの正当性を暗号的に保証 |
| CVEモニタリング | 新たな脆弱性を継続的に検知 |
| AIBOM（AI Bill of Materials） | 全ビルドの来歴を記録しタンパリングを検出 |

AIBOMはSBOM（Software Bill of Materials）のAI版。モデルウェイト・プロンプト・ツール定義の来歴証明まで対象になりうる概念で、現時点では業界での実装仕様が発展途上だ。

### CI/CD Security Gates（CI/CDセキュリティゲート）

セキュリティチェックをCI/CDパイプラインのゲートとして組み込む。

失敗条件の例：
- 新しいコードが既知の脆弱性を導入する
- 未承認の依存関係を使用している
- ポリシーチェックに失敗する（OPAなどのPolicy-as-Codeツールを使用）

### Safe Error Handling（安全なエラーハンドリング）

> "Do not return stack traces, tokens, filesystem paths, or tool internals in responses returned to the model/client."

エラー応答はモデルとクライアントに返す。**スタックトレース・トークン・ファイルシステムパス・ツール内部情報は含めない。**

エラーメッセージ自体が、攻撃者の偵察情報（サーバー構成・ファイル配置・使用ライブラリ等）になる。

---

## 7. Governance ― ガバナンス

### Cryptographic Integrity（暗号的整合性）

すべてのツール・依存関係・レジストリマニフェストに対して：

- 暗号署名を適用
- バージョンピニングを実施

改ざんを防ぎ、整合性を保証する。

### Peer Review and Oversight（ピアレビュー）

**新規ツールや主要なコード変更は、セキュリティフォーカスのピアレビューなしに本番環境に投入しない。** これをポリシーとして確立する。

### Audit Logs and Trails（監査ログ）

記録すべき操作の例：

| カテゴリ | 内容 |
|---|---|
| ツール呼び出し | 呼び出したツール名とパラメータ |
| リソースアクセス | アクセスしたリソースの種別・ID |
| 認証・認可イベント | ログイン・トークン発行・権限昇格 |
| 設定変更 | 変更前後の値を両方記録 |

フィールドレベルのAllowlistとRedaction/Hashingを使い、センシティブデータをそのままログに残さない。ログは**セキュアかつイミュータブル（改ざん不能）** な形で保管し、フォレンジック分析に使える状態にする。

### Non-Human Identity（NHI）Governance

> "Treat all automated agents, backend processes or MCP server systems as first-class identities with unique credentials and tightly scoped permissions."

**AIエージェント・バックエンドプロセス・MCPサーバーを「非人間アイデンティティ（NHI）」として人間と同じガバナンス対象にする。**

各NHIに一意のクレデンシャルを発行し、権限スコープを最小限に制限する。NHIのデータアクセスとツール使用を継続的に監査する。

これは思想転換を意味する。「AIが動かしている処理」という曖昧なカテゴリで権限を管理するのではなく、**AIシステム自体をIDとして明示的に管理する。**

---

## 8. Tools & Continuous Validation ― ツールと継続的バリデーション

### Automated Code Scanning（自動コードスキャン）

CI/CDパイプラインに組み込むスキャン：

| ツール種別 | 目的 |
|---|---|
| SAST（MCP専用ルール付き） | ソースコードの静的解析 |
| Invariant MCP-Scan | MCP固有の脆弱性検出 |
| SCA（npm audit / pip audit / OSV-Scanner） | 依存関係の既知脆弱性検出 |

リスク許容度を超えるスキャン結果が出た場合はビルドを失敗させる。

### Runtime Protections（ランタイム保護）

静的検査では捕捉できない実行時の異常を検知・強制する：

| 技術 | 用途 |
|---|---|
| Docker seccompプロファイル | 許可するシステムコールを制限 |
| AppArmor | ファイル・ネットワークアクセスの強制 |
| context-protector | コンテキスト汚染の検出 |
| mcp-watch | MCPの実行時挙動モニタリング |

これが**Runtime-native Security**の実体だ。コードを書く時点では予測できない実行時の逸脱（想定外のネットワーク通信・syscall・メモリアクセス）は、ランタイムレベルで監視・制限しなければ防げない。

### Continuous Monitoring（継続的モニタリング）

MCPサーバーの監査ログを**SIEM（Security Information and Event Management）** システムに連携し、以下のような異常パターンにリアルタイムアラートを設定する：

- バリデーション失敗の急増
- 高頻度なツール呼び出し
- ツールによる異常なファイルアクセス

### Supply Chain Vigilance（サプライチェーン監視）

- OpenSSF Scorecardでプロジェクトのセキュリティポスチャを評価
- OSV（Open Source Vulnerabilities）等の脆弱性データベースで依存関係の新規問題を継続監視

---

## MCP Security Minimum Bar ― 最低基準チェックリスト

OWASPが「これを満たさないならMCPサーバーを公開するな」と位置づける最低要件。

### 1. Strong Identity, Auth & Policy Enforcement

- [ ] すべてのリモートMCPサーバーでOAuth 2.1 / OIDCを使用
- [ ] トークンは短命・スコープ制限・全呼び出しで再検証
- [ ] Token Passthroughなし・ポリシー強制は一元化

### 2. Strict Isolation & Lifecycle Control

- [ ] ユーザー・セッション・実行コンテキストが完全隔離
- [ ] ユーザーデータの共有ステートなし
- [ ] セッションは決定論的クリーンアップと強制リソースクォータを持つ

### 3. Trusted, Controlled Tooling

- [ ] ツールは暗号署名・バージョンピニング・正式承認済み
- [ ] ツールの説明文は実行時挙動と照合して検証
- [ ] モデルに公開するツールフィールドは最小限

### 4. Schema-Driven Validation Everywhere

- [ ] すべてのMCPメッセージ・ツール入力・出力がスキーマ検証済み
- [ ] 入出力のサニタイズ・サイズ制限・Untrusted扱いを徹底
- [ ] 構造化（JSON）ツール呼び出しを必須とする

### 5. Hardened Deployment & Continuous Oversight

- [ ] サーバーはコンテナ化・非rootユーザー・ネットワーク制限
- [ ] シークレットはVaultに保管・LLMには公開しない
- [ ] CI/CDセキュリティゲート・監査ログ・継続的モニタリングは必須

---

## ガイドが触れていない課題

原文PDFを精読すると、いくつかの領域がまだカバーされていない。

### MCPサーバーの発見・信頼確立プロセス

ツールのロード時署名検証は定義されているが、「**そもそもどのMCPサーバーを接続するか**」という信頼確立のフロー（レジストリ、マニフェスト配布の仕組み）は未定義だ。

### One Task, One Session と状態継続性の矛盾

ステートフルなマルチステップエージェント（長期タスクの途中状態を保持しながら進む）では、セッションリセット原則と状態継続性が根本的に衝突する。このトレードオフへの回答はガイドにない。

### マルチエージェント構成（Agent-to-Agent）

エージェントAがエージェントBをMCP経由で呼び出す構成での**信頼連鎖の定義**が未定義。現実のプロダクション環境ではこのパターンが急増しているが、v1.0では扱われていない。

---

## まとめ ― AIセキュリティはRuntimeへ移動した

OWASPがMCP専用ガイドを出した意味を、もう一度整理する。

MCPは「APIのラッパー」ではなく、**AIの実行基盤**だ。実行基盤のセキュリティは：

- 認証だけでは守れない
- 静的コード検査だけでは守れない
- WAFだけでは守れない

必要なのは：

| 領域 | 対策の軸 |
|---|---|
| アーキテクチャ | セッション完全隔離・Token Delegation・Lifecycle Management |
| ツール設計 | 署名付きマニフェスト・説明文と挙動の照合・最小公開フィールド |
| 認証・認可 | OAuth 2.1 / OIDC・Token Passthrough禁止・NHIガバナンス |
| Prompt Injection | 構造化呼び出し・HITL・LLM-as-a-Judge・One Task One Session |
| デプロイ | 非rootコンテナ・Vault・AIBOM・CI/CDゲート |
| Runtime | seccomp / AppArmor・mcp-watch・SIEM連携 |

チェックリスト（p.14）はスタート地点に過ぎない。しかし**「ここを満たさないならMCPサーバーを公開するな」** という基準を業界が持てたことの意義は大きい。

---

## 参考資料

- [OWASP GenAI Security Project](https://genai.owasp.org)
- A Practical Guide for Secure MCP Server Development v1.0（February 2026）
- [RFC 8693 ― OAuth 2.0 Token Exchange](https://www.rfc-editor.org/rfc/rfc8693)
- [OpenSSF Scorecard](https://securityscorecards.dev)
- [OSV ― Open Source Vulnerabilities](https://osv.dev)
