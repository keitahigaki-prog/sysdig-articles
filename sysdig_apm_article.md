# Sysdig APM 徹底解説 — SDK型APMとの違いと実践ガイド

## はじめに

近年、クラウドネイティブ環境でのアプリケーション監視は、単なるCPU/メモリ監視では不十分になってきました。マイクロサービス化が進み、サービス間の依存関係が複雑化する中で、**APM（Application Performance Monitoring）** の重要性が増しています。

DatadogやNew Relicなどの**SDK型APM**が主流ですが、Sysdigは全く異なるアプローチを採用しています。本記事では、Sysdig APMの仕組みと、従来のAPMとの違いを実践的な視点で解説します。

### 対象読者

- Kubernetes/コンテナ環境を運用しているSRE・インフラエンジニア
- APMの導入を検討している方
- SDK型APMの導入に課題を感じている方
- eBPFベースの監視技術に興味がある方

### この記事で分かること

- Sysdig APMの技術的な仕組み
- SDK型APMとの具体的な違い
- 実際の導入方法と設定例
- ユースケース別の活用パターン
- 導入時の注意点とベストプラクティス

## 目次

1. [APMの基本とは？](#1-apmの基本とは)
2. [SDK型APMの仕組みと課題](#2-sdk型apmの仕組みと課題)
3. [Sysdig APMは何が違うのか？](#3-sysdig-apmは何が違うのか)
4. [詳細比較：SDK型 vs Sysdig](#4-詳細比較sdk型-vs-sysdig)
5. [実践：Sysdig APMの導入手順](#5-実践sysdig-apmの導入手順)
6. [ユースケース別活用パターン](#6-ユースケース別活用パターン)
7. [導入時の注意点とベストプラクティス](#7-導入時の注意点とベストプラクティス)
8. [まとめ：どちらを選ぶべきか](#8-まとめどちらを選ぶべきか)

---

## 1. APMの基本とは？

APMは「**アプリケーションの応答性能を監視し、遅延やエラー、ボトルネックを可視化する仕組み**」です。

### APMの主要な機能

| 機能 | 説明 |
|------|------|
| **トレーシング** | リクエストの流れをサービス間で追跡 |
| **メトリクス収集** | レスポンスタイム、エラー率、スループット |
| **サービスマップ** | マイクロサービス間の依存関係を可視化 |
| **アラート** | 性能劣化時の通知 |
| **ルート原因分析** | ボトルネックの特定と分析 |

### 従来のモニタリングとの違い

```
従来のインフラ監視:
- CPU使用率: 80%
- メモリ使用率: 65%
→ でも、「なぜ遅いのか？」は分からない

APM:
- /api/users エンドポイント: 平均2.3秒 (通常0.5秒)
- データベースクエリが1.8秒 ← ボトルネック特定！
- 特定のSQLクエリがインデックスを使っていない
→ 具体的な改善策が見える
```

---

## 2. SDK型APMの仕組みと課題

### 2.1 SDK型APMの仕組み

代表的なAPM（Datadog APM、New Relic、AppDynamicsなど）は、**SDK/エージェントをアプリケーションに組み込む方式**が主流です。

#### アーキテクチャ

```
┌─────────────────────────────────────┐
│   Application Code                  │
│                                     │
│   ┌──────────────────────────┐    │
│   │  APM SDK (Instrumentation) │    │
│   │  - Trace API calls         │    │
│   │  - Measure timing          │    │
│   │  - Collect context         │    │
│   └──────────┬───────────────┘    │
└──────────────┼─────────────────────┘
               │ Traces/Metrics
               ▼
       ┌───────────────┐
       │  APM Agent    │
       └───────┬───────┘
               │
               ▼
       ┌───────────────┐
       │  APM Backend  │
       │  (Datadog等)  │
       └───────────────┘
```

#### 実装例（Python + Datadog）

```python
from ddtrace import tracer
from flask import Flask

app = Flask(__name__)

@app.route('/api/users/<user_id>')
def get_user(user_id):
    # 自動的にトレースされる
    with tracer.trace("database.query", service="user-api"):
        user = db.query(f"SELECT * FROM users WHERE id={user_id}")

    with tracer.trace("cache.get", service="user-api"):
        cache.set(f"user:{user_id}", user)

    return user
```

```yaml
# Kubernetes Deployment例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-api
spec:
  template:
    spec:
      containers:
      - name: app
        image: user-api:latest
        env:
        - name: DD_AGENT_HOST
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: DD_TRACE_ENABLED
          value: "true"
        - name: DD_SERVICE
          value: "user-api"
```

### 2.2 SDK型APMの特徴

#### メリット

- **高精度トレーシング**: メソッド単位、SQL単位での計測
- **リッチなコンテキスト**: ユーザーID、トランザクションIDなどビジネスコンテキストの紐付け
- **言語固有の最適化**: 各言語の特性に合わせた計装

#### デメリット（課題）

| 課題 | 詳細 |
|------|------|
| **コード変更が必要** | SDKの追加、設定ファイルの変更 |
| **言語依存** | 対応言語が限定される、新言語への対応待ち |
| **メンテナンス負荷** | SDKバージョン管理、互換性維持 |
| **レガシーアプリ対応困難** | 古いコードベースへの組み込みが難しい |
| **マルチテナント環境の複雑性** | 多数のマイクロサービスへの個別導入 |
| **デプロイ影響** | SDK更新のたびに再デプロイ必要 |

### 2.3 実際の導入時の課題例

```
ケース1: マイクロサービス環境（50サービス）
- 言語: Java, Go, Python, Node.js, Ruby
- 課題: 各言語のSDKを個別に導入・設定
- 工数: 1サービスあたり2-3日 × 50 = 100-150日

ケース2: レガシーアプリケーション
- 言語: Java 8（古いフレームワーク）
- 課題: 最新SDKと互換性なし
- 結果: APM導入断念

ケース3: 動的スケーリング環境
- 環境: Kubernetes + HPA
- 課題: 新規Podへの自動計装が複雑
- 対応: Admission Webhookなどの追加実装が必要
```

---

## 3. Sysdig APMは何が違うのか？

Sysdig APMは**SDK不要**で、**eBPF**（extended Berkeley Packet Filter）を使った全く異なるアプローチを採用しています。

### 3.1 eBPFとは？

eBPFは、Linuxカーネル内でサンドボックス化されたプログラムを安全に実行する技術です。

```
┌──────────────────────────────────────┐
│        User Space                     │
│  ┌──────────┐  ┌──────────┐         │
│  │  App 1   │  │  App 2   │         │
│  └────┬─────┘  └────┬─────┘         │
└───────┼─────────────┼────────────────┘
        │             │
═══════════════════════════════════════
        │             │
┌───────▼─────────────▼────────────────┐
│       Linux Kernel                    │
│                                       │
│  ┌─────────────────────────────┐    │
│  │  eBPF Programs              │    │
│  │  - System calls監視         │    │
│  │  - Network packet監視       │    │
│  │  - Process events監視       │    │
│  └─────────┬───────────────────┘    │
└────────────┼─────────────────────────┘
             │ データ収集
             ▼
     ┌──────────────┐
     │ Sysdig Agent │
     └──────────────┘
```

### 3.2 Sysdig APMの仕組み

#### アーキテクチャ

```
┌─────────────────────────────────────────────┐
│  Kubernetes Cluster                         │
│                                             │
│  ┌────────────┐  ┌────────────┐           │
│  │   Pod A    │  │   Pod B    │           │
│  │  (No SDK)  │  │  (No SDK)  │           │
│  └──────┬─────┘  └──────┬─────┘           │
│         │                │                  │
│    ┌────▼────────────────▼─────┐          │
│    │  Sysdig Agent (DaemonSet) │          │
│    │  - eBPF probes             │          │
│    │  - Kernel events収集        │          │
│    │  - Network traffic解析     │          │
│    └────────────┬───────────────┘          │
└─────────────────┼──────────────────────────┘
                  │
                  ▼
          ┌──────────────┐
          │ Sysdig Cloud │
          │  - APM UI    │
          │  - Analytics │
          └──────────────┘
```

### 3.3 データ収集の仕組み

Sysdigは以下の情報をカーネルレベルで収集します：

```
収集データ:
├─ System Calls (syscalls)
│  ├─ read/write
│  ├─ connect/accept (ネットワーク)
│  └─ execve (プロセス起動)
│
├─ Network Traffic
│  ├─ HTTP/HTTPS requests
│  ├─ gRPC calls
│  ├─ Database queries (PostgreSQL, MySQL, Redis等)
│  └─ Service mesh traffic (Istio, Linkerd)
│
├─ Process Metrics
│  ├─ CPU使用率（プロセス単位）
│  ├─ Memory使用率
│  └─ I/O統計
│
└─ Kubernetes Context
   ├─ Pod名、Namespace
   ├─ Service名
   └─ Labels/Annotations
```

### 3.4 具体例：HTTPリクエストの追跡

SDK型APMの場合：

```python
# アプリコードに計装が必要
@tracer.trace("http.request")
def handle_request():
    start = time.time()
    response = requests.get("http://api.example.com/users")
    duration = time.time() - start
    tracer.set_tag("http.status_code", response.status_code)
    tracer.set_tag("http.duration", duration)
```

Sysdig APMの場合：

```python
# アプリコードは一切変更不要
def handle_request():
    response = requests.get("http://api.example.com/users")
    # Sysdigが自動的に以下を収集:
    # - connect() syscall
    # - send()/recv() syscall
    # - HTTP method, path, status code
    # - Response time
    # - Pod/Service context
```

---

## 4. 詳細比較：SDK型 vs Sysdig

### 4.1 総合比較表

| 観点 | SDK型APM<br>(Datadog, New Relic等) | Sysdig APM<br>(eBPFベース) |
|------|-----------------------------------|---------------------------|
| **導入方法** | SDK/ライブラリの組み込み | エージェントのDaemonSet展開のみ |
| **コード変更** | 必要 | 不要 |
| **言語依存** | あり（対応言語のみ） | なし（言語非依存） |
| **対応フレームワーク** | SDK対応範囲に依存 | ほぼすべて（HTTP/gRPCベース） |
| **トレース粒度** | メソッド/関数レベル | プロセス/通信レベル |
| **内部処理の可視化** | 可能（詳細） | 不可能（外部通信のみ） |
| **SQLクエリ詳細** | クエリ本文まで取得可能 | 接続情報とレイテンシのみ |
| **カスタムメトリクス** | 容易（コードで追加） | 困難（カーネル経由のみ） |
| **オーバーヘッド** | 低〜中（1-5%） | 極低（1%未満） |
| **レガシーアプリ対応** | 困難 | 容易 |
| **マルチテナント** | 個別設定必要 | 一括対応可能 |
| **Kubernetes統合** | 追加設定必要 | ネイティブ対応 |
| **セキュリティ監視統合** | 別製品 | Sysdig Secureと統合 |
| **コスト** | サービス単位課金 | ノード/コンテナ単位課金 |

### 4.2 トレース精度の違い

#### SDK型APM：深いトレース

```
Request Flow (Datadog APM):

GET /api/orders/123                            Total: 245ms
│
├─ [Controller] OrderController.getOrder        5ms
│  └─ [Validation] validateOrderId              2ms
│
├─ [Service] OrderService.fetchOrder          180ms
│  ├─ [DB] SELECT * FROM orders WHERE id=123  150ms
│  │  └─ Query: "SELECT o.*, u.name FROM orders o JOIN users u ON..."
│  │
│  └─ [Cache] Redis.get("order:123")           30ms
│     └─ Command: GET order:123
│
└─ [Serialization] JSON.encode                  60ms
   └─ Custom serializer overhead detected
```

#### Sysdig APM：サービス間トレース

```
Request Flow (Sysdig APM):

GET /api/orders/123                            Total: 245ms
│
├─ [frontend-service] → [order-service]        10ms (network)
│  └─ HTTP POST /orders/123
│
├─ [order-service] → [postgres-db]           150ms
│  └─ PostgreSQL connection (10.0.1.5:5432)
│     - Queries: 3
│     - Bytes: 2.3KB
│
├─ [order-service] → [redis-cache]            30ms
│  └─ Redis connection (10.0.1.8:6379)
│     - Commands: 1
│
└─ [order-service] → [frontend-service]        5ms (network)
   └─ HTTP 200 OK
```

### 4.3 どちらが向いているか？

#### SDK型APMが向いているケース

```
✅ アプリケーション内部の詳細分析が必要
   例: 特定メソッドのパフォーマンス最適化

✅ ビジネスメトリクスとの紐付けが必要
   例: ユーザーID、注文ID、決済情報などのコンテキスト

✅ カスタムトレースポイントの追加
   例: ビジネスロジックの特定箇所を計測

✅ 言語が限定的（Java、Pythonなど主要言語のみ）
```

#### Sysdig APMが向いているケース

```
✅ Kubernetes/コンテナ環境が中心
   例: マイクロサービスアーキテクチャ

✅ 多言語混在環境
   例: Java + Go + Python + Node.js + Rust

✅ レガシーアプリケーションも含む
   例: 変更困難な既存システム

✅ サービス間通信の可視化が主目的
   例: マイクロサービス間のレイテンシ分析

✅ インフラ+セキュリティ+APMの統合運用
   例: SysdigのMonitor + Secureの併用

✅ 迅速な導入が求められる
   例: エージェント展開のみで即時開始
```

---

## 5. 実践：Sysdig APMの導入手順

### 5.1 前提条件

```yaml
環境要件:
- Kubernetes: 1.19以上
- OS: Linux kernel 4.14以上（eBPF対応）
- CPU arch: x86_64, ARM64
- 権限: DaemonSetの展開権限
```

### 5.2 導入手順

#### Step 1: Sysdigアカウントの準備

```bash
# アクセスキーの取得
# Sysdig UI > Settings > Agent Installation から取得
export SYSDIG_ACCESS_KEY="your-access-key-here"
export SYSDIG_REGION="us1"  # または us2, us3, eu1, ap1等
```

#### Step 2: Helmリポジトリの追加

```bash
helm repo add sysdig https://charts.sysdig.com
helm repo update
```

#### Step 3: Values.yamlの作成

```yaml
# sysdig-values.yaml
global:
  sysdig:
    accessKey: "YOUR_ACCESS_KEY_HERE"
    region: "us1"

agent:
  # eBPFを有効化
  ebpf:
    enabled: true

  # APM機能の有効化
  sysdig:
    settings:
      app_checks_enabled: true

      # HTTPトレーシング
      network_topology:
        enabled: true

      # プロセスメトリクス
      process_names:
        enabled: true

      # カスタムアプリ検出
      app_checks:
        - name: http
          enabled: true
          interval: 10
        - name: postgresql
          enabled: true
        - name: redis
          enabled: true
        - name: mysql
          enabled: true
        - name: mongodb
          enabled: true

  # リソース制限
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

  # 収集データのフィルタリング（オプション）
  sysdig:
    settings:
      # 特定のnamespaceのみ監視
      k8s_namespaces:
        - production
        - staging

      # 除外するnamespace
      k8s_namespaces_exclude:
        - kube-system
        - kube-public

# Node Analyzerの有効化（脆弱性スキャン）
nodeAnalyzer:
  enabled: true
  secure:
    vulnerabilityManagement:
      newEngineOnly: true
```

#### Step 4: エージェントのデプロイ

```bash
# Namespaceの作成
kubectl create namespace sysdig-agent

# Helmでインストール
helm install sysdig-agent sysdig/sysdig-deploy \
  --namespace sysdig-agent \
  --values sysdig-values.yaml

# デプロイ確認
kubectl get pods -n sysdig-agent
```

#### Step 5: 動作確認

```bash
# エージェントのログ確認
kubectl logs -n sysdig-agent -l app.kubernetes.io/name=sysdig-agent

# 期待される出力:
# * Sysdig Agent connected successfully
# * eBPF probe loaded
# * Application checks enabled: http, postgresql, redis
```

### 5.3 アプリケーションへの自動検出設定

Sysdigは自動的にアプリケーションを検出しますが、より正確な識別のためにラベルを追加できます：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      labels:
        app: order-service
        version: v1.2.3
        # Sysdig用のラベル
        sysdig.com/app-name: "order-service"
        sysdig.com/tier: "backend"
    spec:
      containers:
      - name: order-service
        image: order-service:1.2.3
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        # 環境変数は不要（SDKと違い）
```

### 5.4 Sysdig UIでの確認

導入後、Sysdig UIで以下を確認できます：

```
1. Service Map
   - マイクロサービス間の依存関係
   - レイテンシのビジュアル表示

2. APM Metrics
   - Response time (p50, p95, p99)
   - Request rate
   - Error rate
   - サービス単位、エンドポイント単位

3. Network Topology
   - Pod間通信
   - 外部API通信
   - データベース接続

4. Golden Signals
   - Latency
   - Traffic
   - Errors
   - Saturation
```

---

## 6. ユースケース別活用パターン

### 6.1 ケース1: マイクロサービスのレイテンシ分析

#### 課題

```
症状: /api/checkout エンドポイントが突然遅くなった
通常: 200ms
現在: 2000ms（10倍遅い）
```

#### Sysdig APMでの分析

```
1. Service Mapで依存関係を確認
   checkout-service → payment-service → fraud-detection-service
                    → inventory-service
                    → notification-service

2. 各サービスのレイテンシを確認
   checkout-service: 50ms (正常)
   payment-service: 100ms (正常)
   fraud-detection-service: 1800ms ← 原因特定！
   inventory-service: 30ms (正常)
   notification-service: 20ms (正常)

3. fraud-detection-serviceの詳細分析
   - 外部API呼び出し: 1750ms
   - 外部API: https://fraud-check.example.com
   - Network latency: 1700ms (通常50ms)

4. Network Topologyで確認
   - fraud-check.example.com へのパケットロス: 30%
   - 原因: 外部サービスの障害

5. 対応
   - Circuit Breakerの追加
   - タイムアウト設定の調整（3秒 → 500ms）
   - フォールバック処理の実装
```

### 6.2 ケース2: データベースボトルネックの特定

#### 課題

```
症状: ユーザーAPIのレスポンスが遅い
原因: 不明（アプリログには何も出ていない）
```

#### Sysdig APMでの分析

```bash
# Sysdig UIまたはCLIで確認
# sysdig-inspect (Sysdig CLIツール) での分析例

# PostgreSQLへの接続を監視
sysdig -c topprocs_net 'fd.sip=10.0.1.5 and fd.sport=5432'

出力:
PROCESS              PID    RX(B/s)  TX(B/s)
user-service        1234    125KB    2KB     ← 読み込みが多い
order-service       5678    50KB     1KB

# 遅いクエリを特定（レイテンシ順）
Sysdig UI > APM > Databases > PostgreSQL
- SELECT * FROM users: 平均 150ms (通常10ms)
- クエリ実行回数: 1000回/秒
- 問題: N+1クエリ問題の可能性

# Kubernetes Podとの紐付け
Pod: user-service-7d9f8b-xyz
Namespace: production
Node: node-03
```

#### 対応

```yaml
# 原因: user-serviceでキャッシュが効いていない
# 対策: Redisキャッシュの追加

apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  template:
    spec:
      containers:
      - name: user-service
        env:
        - name: REDIS_ENABLED
          value: "true"
        - name: REDIS_HOST
          value: "redis-cache.production.svc.cluster.local"

# 改善後のSysdig APMメトリクス:
# - PostgreSQL query latency: 150ms → 10ms
# - Redis cache hit rate: 0% → 85%
# - Overall response time: 200ms → 50ms
```

### 6.3 ケース3: セキュリティインシデントとパフォーマンスの相関分析

Sysdigの強み：**Sysdig Secure**と**Sysdig Monitor**の統合

#### シナリオ

```
Sysdig Secure Alert:
- 異常なプロセス起動を検知
- Pod: frontend-web-5f7d8-abc
- Process: /tmp/cryptominer
- 時刻: 2026-02-06 03:15:22 UTC
```

#### Sysdig APM + Secureでの分析

```
1. Sysdig Secureでの検知
   - Runtime Detection: 不審なバイナリ実行
   - Network Activity: 外部IPへの大量接続
   - Process: /tmp/cryptominer (SHA256: abc123...)

2. Sysdig APM (Monitor) で影響範囲を確認
   - CPU使用率: 通常30% → 95% (03:15以降)
   - Response time: 通常100ms → 500ms (03:15以降)
   - Error rate: 通常0.1% → 5% (03:16以降)

3. Service Mapでの影響確認
   frontend-web → backend-api: レイテンシ増加
   frontend-web → 外部IP(不明): 大量通信 ← 異常

4. Timeline分析
   03:15:22 - 不審なプロセス起動
   03:15:30 - CPU使用率急増
   03:16:00 - エラー率上昇開始
   03:17:00 - 自動スケーリング発動（HPA）
   03:18:00 - 運用チームが気づく（Sysdig Alert）

5. 対応
   - 該当Podの即時削除
   - コンテナイメージの脆弱性スキャン
   - Network Policyで外部通信を制限
```

---

## 7. 導入時の注意点とベストプラクティス

### 7.1 eBPF対応の確認

```bash
# カーネルバージョン確認
uname -r
# 必要: 4.14以上（推奨: 5.8以上）

# eBPF機能の確認
cat /proc/sys/kernel/unprivileged_bpf_disabled
# 0 = 有効, 1 = 無効（rootのみ）

# BTF (BPF Type Format) サポート確認
ls /sys/kernel/btf/vmlinux
# 存在すればBTF対応（CO-RE eBPF可能）
```

### 7.2 権限とセキュリティ

```yaml
# Sysdig AgentのServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sysdig-agent
  namespace: sysdig-agent

---
# 必要な権限
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sysdig-agent
rules:
# Podメタデータの読み取り
- apiGroups: [""]
  resources: ["pods", "services", "namespaces", "nodes"]
  verbs: ["get", "list", "watch"]

# Events読み取り
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]

# DaemonSetとして動作
- apiGroups: ["apps"]
  resources: ["daemonsets", "deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
```

### 7.3 リソース使用量の最適化

```yaml
# 環境に応じたリソース設定

# 小規模環境（<50 Pods/Node）
agent:
  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

# 中規模環境（50-200 Pods/Node）
agent:
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# 大規模環境（>200 Pods/Node）
agent:
  resources:
    requests:
      cpu: 1000m
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 2Gi

  # サンプリングレートの調整
  sysdig:
    settings:
      # デフォルト: すべてのイベントを収集
      # 大規模環境: サンプリング有効化
      sampling_ratio: 100  # 1/100のイベントを収集
```

### 7.4 データ保持とコスト最適化

```yaml
# Sysdig UIでのデータ保持設定

APM Metrics:
  - High-resolution (1秒粒度): 1時間
  - Medium-resolution (10秒粒度): 24時間
  - Low-resolution (1分粒度): 14日間
  - Aggregated (1時間粒度): 90日間

Traces (Distributed Tracing):
  - サンプリング: 1% (デフォルト)
  - 保持期間: 7日間
  - Cost-optimization: サンプリングレート調整可能

Infrastructure Metrics:
  - 保持期間: 90日間
```

### 7.5 ベストプラクティス

#### 1. 段階的な導入

```bash
# Phase 1: 開発環境で試す
helm install sysdig-agent sysdig/sysdig-deploy \
  -n sysdig-agent \
  --set agent.sysdig.settings.k8s_namespaces={development}

# Phase 2: ステージング環境
helm upgrade sysdig-agent sysdig/sysdig-deploy \
  -n sysdig-agent \
  --set agent.sysdig.settings.k8s_namespaces={development,staging}

# Phase 3: 本番環境（段階的）
# まずは重要度の低いサービスから
helm upgrade sysdig-agent sysdig/sysdig-deploy \
  -n sysdig-agent \
  --set agent.sysdig.settings.k8s_namespaces={development,staging,production-tier3}
```

#### 2. アラート設定

```yaml
# 推奨アラート設定例

# レイテンシアラート
- name: "High Response Time - Order Service"
  condition: "avg(http.request.time) > 1000"  # 1秒以上
  scope: "kubernetes.deployment.name = 'order-service'"
  duration: 300  # 5分間継続
  severity: warning

# エラー率アラート
- name: "High Error Rate"
  condition: "sum(http.statusCode) / sum(http.request.count) > 0.05"  # 5%以上
  duration: 180  # 3分間継続
  severity: critical

# データベース接続異常
- name: "Database Connection Issues"
  condition: "avg(net.connection.time) > 500"  # 500ms以上
  scope: "fd.sport = 5432"  # PostgreSQL
  duration: 300
  severity: warning
```

#### 3. タグとラベルの活用

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    # 標準的なKubernetesラベル
    app: payment-service
    version: v2.1.0

    # Sysdig APMでのグルーピング用
    sysdig.com/team: "payment-team"
    sysdig.com/tier: "critical"
    sysdig.com/cost-center: "payments"
spec:
  template:
    metadata:
      labels:
        app: payment-service
        version: v2.1.0
        sysdig.com/team: "payment-team"
    spec:
      containers:
      - name: payment-service
        image: payment-service:v2.1.0
```

#### 4. 他のツールとの併用

```
推奨構成:

Sysdig APM:
  ✅ サービス間通信の可視化
  ✅ インフラメトリクス
  ✅ セキュリティ監視

+

Logging (ELK, Loki等):
  ✅ 詳細なアプリケーションログ
  ✅ エラースタックトレース

+

SDK型APM (必要に応じて):
  ✅ 特定サービスの詳細分析
  ✅ ビジネスメトリクス
```

---

## 8. まとめ：どちらを選ぶべきか

### 8.1 選択フローチャート

```
START
│
├─ Kubernetes/コンテナ環境が中心？
│  YES → Sysdig APM推奨
│  NO  → ↓
│
├─ 多言語混在環境？
│  YES → Sysdig APM推奨
│  NO  → ↓
│
├─ コード変更が困難？（レガシー/規制等）
│  YES → Sysdig APM推奨
│  NO  → ↓
│
├─ アプリ内部の詳細分析が必要？
│  YES → SDK型APM推奨
│  NO  → ↓
│
├─ ビジネスコンテキストの追跡が必要？
│  YES → SDK型APM推奨
│  NO  → Sysdig APM推奨
```

### 8.2 最終的な推奨

#### Sysdig APMを選ぶべきケース

```
✅ Kubernetes環境でマイクロサービスを運用
✅ 迅速な導入が必要（数時間〜1日）
✅ 多言語混在（Java + Go + Python + ...）
✅ セキュリティとパフォーマンスを統合管理したい
✅ インフラチームが主導
```

#### SDK型APMを選ぶべきケース

```
✅ モノリシックアプリケーション中心
✅ アプリ内部の詳細なトレースが必要
✅ ビジネスメトリクスとの紐付けが重要
✅ 特定言語に特化（Javaのみ、等）
✅ 開発チームが主導
```

#### ハイブリッド構成

```
多くの企業では併用が現実的:

全体監視: Sysdig APM
- すべてのマイクロサービス
- インフラメトリクス
- セキュリティ監視

詳細分析: SDK型APM
- 重要な3-5個のコアサービスのみ
- ビジネスクリティカルなトランザクション
- A/Bテスト効果測定
```

### 8.3 今後のトレンド

```
eBPFベース監視の台頭:
- Sysdig以外にも Cilium, Pixie, Parca等
- カーネルレベル監視が標準化へ
- "SDK不要" がスタンダードになる可能性

OpenTelemetry:
- ベンダー非依存のトレーシング標準
- Sysdigも一部対応予定
- SDK型とeBPF型の橋渡しに
```

---

## 参考リンク

### 公式ドキュメント
- [Sysdig APM 公式ドキュメント](https://docs.sysdig.com/en/docs/sysdig-monitor/monitoring-your-apps/apm/)
- [eBPF 公式サイト](https://ebpf.io/)
- [OpenTelemetry](https://opentelemetry.io/)

### 関連記事
- [eBPF入門 - Linuxカーネル監視の革命](https://ebpf.io/what-is-ebpf/)
- [Kubernetes Observability Best Practices](https://kubernetes.io/docs/concepts/cluster-administration/system-metrics/)

### 比較記事
- Datadog APM vs Sysdig APM
- New Relic vs eBPF-based monitoring

---

## おわりに

Sysdig APMは、従来のSDK型APMとは全く異なるアプローチで、特にKubernetes/コンテナ環境において強力な可視化を提供します。

**SDK不要**という特徴は、単なる利便性ではなく、以下の戦略的メリットをもたらします：

1. **迅速な導入** - 数時間で全サービスを可視化
2. **運用負荷の削減** - SDKのバージョン管理やメンテナンス不要
3. **言語非依存** - 新しい言語やフレームワークへの即座の対応
4. **セキュリティとの統合** - パフォーマンス異常とセキュリティ異常の相関分析

一方で、アプリケーション内部の詳細分析やビジネスコンテキストの追跡にはSDK型APMが優れています。

**最適な選択は、あなたの環境と目的によります。** この記事が、その判断の一助となれば幸いです。

---

**タグ候補**: #Sysdig #APM #Kubernetes #eBPF #可観測性 #監視 #マイクロサービス #SRE #Observability #クラウドネイティブ
