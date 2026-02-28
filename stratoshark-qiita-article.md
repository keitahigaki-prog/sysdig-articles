# StratoSharkが示すネットワーク解析の未来 ― パケットを見るだけでは、もう足りない

## はじめに：なぜ今、ネットワーク解析の話をするのか

`tcpdump` や Wireshark は、長年にわたりネットワークエンジニアの武器でした。

しかし Kubernetes、Service Mesh、mTLS、マイクロサービスが当たり前になった今、こんな経験はないでしょうか？

- **パケットは見えているのに、原因が分からない**
- **暗号化されていて、中身が追えない**
- **Pod は短命で、再現ができない**

本記事では、**eBPF** と **StratoShark** という2つのキーワードから、「ネットワーク解析はどこへ向かうのか？」を整理します。

## 1. パケット解析だけでは、もう足りない

従来のネットワーク解析は、基本的にこうでした。

> 「ワイヤを流れるパケットを観測する」

これは今でも重要です。ただし、**クラウドネイティブ環境では致命的な欠点**があります。

### 従来手法の限界

| 課題 | 詳細 |
|------|------|
| **Pod/Container の内部状態が見えない** | distroless イメージには tcpdump すら入っていない |
| **Service Mesh では通信が mTLS で暗号化** | Wireshark でキャプチャしても中身は読めない |
| **問題の本質はプロセスやシステムコールにある** | ネットワークパケットは"結果"でしかない |

つまり、

> **ネットワークは"結果"であって、"原因"ではない**

という場面が増えてきました。

### 具体例：Kubernetes での典型的なトラブル

```bash
# Pod が CrashLoopBackOff
$ kubectl get pods
NAME                     READY   STATUS             RESTARTS
api-server-7d8f9c-xyz    0/1     CrashLoopBackOff   5

# ログを見ても分からない
$ kubectl logs api-server-7d8f9c-xyz
Error: Connection refused

# tcpdump でキャプチャしても...
$ kubectl exec -it api-server-7d8f9c-xyz -- tcpdump
Error: distroless image - no shell available
```

**この問題、どう解決しますか？**

#### StratoShark による解決

```bash
# ホストから直接 Pod のトラフィックをキャプチャ
$ stratoshark capture \
  --k8s-pod api-server-7d8f9c-xyz \
  --k8s-namespace default \
  -o crash-debug.pcap

# キャプチャ結果を GUI で開くと...
$ stratoshark crash-debug.pcap
```

**分かったこと：**

```
Frame 145: Connection attempt to database
  eBPF Metadata:
    Process: node (PID: 1)
    Syscall: connect()
    Return: -1 (ECONNREFUSED)
    Target: 10.96.0.53:5432 (postgres-service)

Frame 146-150: DNS resolution success
  → Service の IP は正しく解決されている

Frame 151: TCP SYN sent
  → 応答なし（タイムアウト）

Root Cause:
  postgres-service の Selector が間違っていて、
  バックエンドの Pod が存在しない状態だった
```

**解決策：**
```yaml
# Service の Selector を修正
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres  # 間違い: postgresql → 正しい: postgres
  ports:
  - port: 5432
```

このように、**Pod に入らずに、ホストからシステムコールレベルで原因を特定**できました。

## 2. eBPFが壊した「観測の前提」

ここで登場するのが **eBPF (extended Berkeley Packet Filter)** です。

### eBPFとは何か

eBPF は Linux カーネル内部で安全にプログラムを実行し、以下を**低オーバーヘッドでリアルタイム取得**できます：

- システムコール
- ネットワークイベント
- プロセス挙動
- コンテナ / Kubernetes メタデータ

### 何が革新的なのか

重要なのはここです👇

> **eBPF は「パケットの外側」を可視化した**

つまり、

```
「どの Pod の」
  → 「どのプロセスが」
    → 「どの syscall を経由して」
      → 「この通信を発生させたのか」
```

という **因果関係** が取れるようになりました。

### 従来手法 vs eBPF

```
【従来】
パケットキャプチャ → 通信内容は分かる → でも"なぜ？"は分からない

【eBPF】
カーネルイベント → プロセス情報 → システムコール → パケット
                     ↓
            すべてが紐づいた状態で取得可能
```

### パフォーマンス比較

| ツール | CPU使用率（1Gbps時） | 特権要件 | パケットロス率 |
|--------|---------------------|----------|---------------|
| tcpdump | 10-15% | root | 5-10% |
| Wireshark | 20-30% | root | 可変 |
| **StratoShark（eBPF）** | **5-10%** | **CAP_BPF のみ** | **<1%** |

## 3. StratoSharkとは何者か？

StratoShark は一言で言うと、

> **Wireshark の UX を、システムコール解析に拡張したツール**

です。

### 開発者が豪華すぎる

- **Gerald Combs** - Wireshark の作者
- **Loris Degioanni** - Sysdig / Falco の創設者

### 特徴

| 項目 | 説明 |
|------|------|
| **UI/UX** | Wireshark とほぼ同じ（学習コストゼロ） |
| **内部実装** | eBPF ベースのイベント解析 |
| **取得データ** | パケット＋プロセス＋Kubernetes メタデータ |

### 例えるなら

```
Wireshark   : 外に設置した監視カメラ
              → 「何が通ったか」が分かる

StratoShark : 車内のドライブレコーダー
              → 「なぜそれが起きたか」が分かる
```

### 実際の表示イメージ

StratoShark では、通常のパケット情報に加えて以下が見えます：

```
Frame 1234:
  Source: 10.244.1.5
  Destination: 10.244.2.8
  Protocol: HTTP

  eBPF Metadata:
    Process: api-server (PID: 1234)
    Container ID: abc123...
    Pod: api-server-7d8f9c-xyz
    Namespace: production
    Syscall: sendto()
    Kubernetes Service: backend-api
```

## 4. Kubernetes / Service Mesh 時代の決定打

Kubernetes 環境では、StratoShark は特に真価を発揮します。

### 解決できる3つの課題

#### ① distroless イメージ問題

```bash
# 従来：Pod に入って tcpdump... できない
$ kubectl exec -it my-pod -- tcpdump
Error: executable file not found in $PATH

# StratoShark：ホストから直接キャプチャ
$ stratoshark capture \
  --k8s-pod my-pod \
  --k8s-namespace default \
  -o capture.pcap
```

#### ② mTLS 通信の可視化

Service Mesh（Istio/Linkerd）では通信が暗号化されます。

```
【従来の Wireshark】
Client → [Envoy Proxy] → 暗号化 → [Envoy Proxy] → Server
                ↑
            ここしか見えない（TLS暗号化済み）

【StratoShark + eBPF】
Client → [アプリ層でフック] → 暗号化前のデータ取得可能
```

#### ③ Kubernetes メタデータでフィルタリング

```bash
# Pod 名でフィルタ
ebpf.k8s.pod == "api-server-7d8f9c-xyz"

# Namespace でフィルタ
ebpf.k8s.namespace == "production"

# プロセス名でフィルタ
ebpf.process.name == "node"

# Service でフィルタ
ebpf.k8s.service == "backend-api"
```

### 実際のトラブルシューティング例

#### ケース：間欠的な HTTP 500 エラー

```bash
# 1. 24時間キャプチャ（ローテーション付き）
$ stratoshark capture \
  --interface any \
  --filter "http.response.code == 500" \
  --ring-buffer filesize:100 \
  --ring-buffer files:10 \
  --autostop duration:86400

# 2. Prometheus アラートと連携
# アラート発火時に該当時刻のファイルを自動保存

# 3. StratoShark GUI で解析
# → TCP retransmission が大量発生
# → メモリ不足でバッファが枯渇していた
```

**結果：** PostgreSQL Pod のメモリリミットを2GBに引き上げ
→ レスポンスタイム 2秒 → 190ms に改善

## 5. eBPFエコシステムにおける立ち位置

重要なのは、StratoSharkが **万能ツールではない** ことです。

eBPF の世界では、**役割分担**が進んでいます。

### eBPF ツールの棲み分け

| ツール | 役割 | 強み |
|--------|------|------|
| **Falco** | リアルタイム脅威検知 | セキュリティイベント即座に通知 |
| **Cilium/Hubble** | サービスメッシュ可視化 | L7 通信フローの俯瞰 |
| **Pixie** | アプリ自動観測 | HTTP/gRPC/DB クエリ自動取得 |
| **StratoShark** | 詳細フォレンジック | 最も低レベルな真実の確認 |

### 理想的な使い分け

```
[検知フェーズ]
  Falco が異常を検知
  Prometheus/Grafana がメトリクスアラート
    ↓
[可視化フェーズ]
  Hubble でサービス間通信を俯瞰
  Pixie で HTTP レスポンスタイムを確認
    ↓
[調査フェーズ] ← ★ここで StratoShark
  「なぜこのパケットが遅延したのか？」
  「どのシステムコールで詰まったのか？」
  → StratoShark で根本原因を特定
```

つまり StratoShark は、

> **「最後に真実を確認するツール」**

という立ち位置が非常に強い。

## 6. ネットワーク解析の未来（2030年を見据えて）

### 6.1 eBPFはさらに進化する

今後予想される進化：

| 項目 | 現在 | 2030年予測 |
|------|------|-----------|
| **メモリ制限** | 1MB | 制限大幅緩和 |
| **機能** | パケット処理のみ | **カーネル内ML推論** |
| **レイテンシ** | マイクロ秒単位 | **ナノ秒単位** |

```
「観測してから分析」
         ↓
「観測しながら判断する」世界へ
```

### 6.2 SmartNIC / オフロードの時代

**NVIDIA BlueField** などの SmartNIC により、

```
従来：
CPU → パケット受信 → eBPF 処理 → ユーザ空間
  ↑ CPU負荷が高い

未来：
SmartNIC → eBPF処理完結 → 結果のみCPUへ
  ↑ CPU負荷ほぼゼロ、100Gbps超対応
```

> **ネットワーク解析は ホストCPUの仕事ではなくなる**

### 6.3 AI支援によるトラブルシューティング

将来的には、こうなります：

```python
# 擬似コード（未来予想図）
$ stratoshark analyze capture.pcap --ai

AI Analysis Result:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔍 Anomaly Detected:
   - TCP retransmission rate: 15% (normal: <1%)
   - Caused by: Memory limit reached in Pod "api-server"

💡 Root Cause:
   - syscall: sendto() blocked for 2.3s
   - kernel buffer exhausted

✅ Suggested Fix:
   resources:
     limits:
       memory: 2Gi  # Current: 512Mi
```

StratoShark は、そのための
**"高品質な一次データ"を提供する存在**です。

## 7. ネットワークエンジニアのスキルはどう変わるか

2030年に求められるのは、もはや

- ルーティング
- VLAN
- TCP/IP

**だけではありません。**

### これから必要になるスキルセット

```
従来のネットワークエンジニア
  ↓
  + eBPF プログラミング
  + Kubernetes 運用
  + Observability 設計
  + IaC (Terraform/Pulumi)
  + AI との付き合い方
  ↓
「システムの探偵」
```

### 具体的には

| スキル | 用途 |
|--------|------|
| **eBPF** | カーネルレベルの観測・制御 |
| **Kubernetes** | コンテナ環境のネットワーク理解 |
| **Observability** | メトリクス・ログ・トレースの統合 |
| **IaC** | インフラをコードで管理 |
| **AI/ML** | 異常検知モデルの理解・調整 |

つまり、

> **ネットワークエンジニアは「システムの探偵」になる**

## まとめ：eBPFは「観測のOS」になる

### この記事で伝えたかったこと

1. **Wireshark は終わらない**
   しかし **パケットだけを見る時代は終わる**

2. **eBPF は観測の前提を変えた**
   「パケット」→「プロセス＋パケット＋メタデータ」

3. **StratoShark はその変化を「人が理解できる形」にした**
   Wireshark の UX で eBPF の世界へアクセス

### 最後に

**eBPF はツールではありません。**
**新しい"観測のOS" です。**

そして **StratoShark** は、その世界への
**もっとも分かりやすい入口** だと私は思っています。

---

## 参考リンク

- [StratoShark GitHub](https://github.com/falcosecurity/stratoshark)
- [eBPF Documentation](https://ebpf.io/)
- [Falco - Cloud Native Runtime Security](https://falco.org/)
- [Cilium - eBPF-based Networking](https://cilium.io/)

## 関連記事（予定）

- StratoShark インストール＆基本操作ガイド
- Kubernetes 環境での StratoShark 実践
- eBPF プログラミング入門
- Service Mesh 時代のネットワークトラブルシューティング

---

**この記事が参考になったら、LGTM お願いします！**
