---
title: "SysdigのInventory / Graph SearchでConfiguration値を検索できない理由と完全攻略ガイド"
type: "tech"
topics: ["sysdig", "aws", "cspm", "cloudsecurity", "security"]
published: false
---

## はじめに

クラウドセキュリティ管理のツールとしてSysdigを使っていると、こんな壁にぶつかることがあります。

> 「EBS VolumeのうちMagneticタイプのものを一覧で出したい」
> → InventoryでもGraph Searchでもうまく絞り込めない…

UIには値が表示されているのに、検索やフィルタに引っかからない。このもどかしさ、経験した方も多いのではないでしょうか。

この記事では：
- **なぜ検索できないのか**（仕組みから）
- **どう回避するか**（実務的な手段ごとに）
- **根本的にどう設計すべきか**（継続運用に向けて）

を体系的にまとめます。

---

## 対象読者

- SysdigのInventory / Graph Searchを使い始めた方
- CSPMツールとしてSysdigを導入したが、フィルタ・検索に限界を感じている方
- 「UIで見えているのに検索できない」問題で詰まっている方

---

## 今回のユースケース：EBS Magnetic の抽出

### AWSにおけるEBS Volumeタイプ

AWSのEBS Volumeには複数のタイプがあります。

| Volume Type | 識別子 | 備考 |
|-------------|--------|------|
| General Purpose SSD | gp2 / gp3 | 現在の標準 |
| Provisioned IOPS SSD | io1 / io2 | 高I/O向け |
| Throughput Optimized HDD | st1 | ログ・ビッグデータ向け |
| Cold HDD | sc1 | アクセス頻度低 |
| **Magnetic** | **standard** | **旧世代・非推奨** |

今回の目的は `volumeType = "standard"` のもの、つまり**旧世代のMagnetic**を洗い出すことです。

コスト最適化やセキュリティポスチャ改善の観点で、こういった旧世代リソースの棚卸しは非常に重要です。

---

## 仕組みから理解する：なぜ検索できないのか

まずは「なぜ動かないのか」を根本から理解しましょう。ここを理解すると、何ができて何ができないかの判断が一気に速くなります。

### Sysdig Inventoryのデータ構造

SysdigのInventoryは内部的に**Graphモデル**で動いています。
CloudResourceには大きく2種類のデータが存在します。

```
CloudResource（例：EBS Volume）
 │
 ├─ [Indexed Field]（検索できる）
 │   ├─ type         = "AWS_EBS_VOLUME"
 │   ├─ id           = "vol-xxxxxxxxx"
 │   ├─ name         = "my-volume"
 │   ├─ accountId    = "123456789012"
 │   └─ region       = "ap-northeast-1"
 │
 └─ [Configuration]（検索できないことがある）
     ├─ volumeType   = "standard"
     ├─ size         = 100
     ├─ encrypted    = false
     └─ iops         = null
```

この**2層構造**が問題の核心です。

### なぜConfigurationはIndexされないのか

Configurationは、AWSやGCP、Azureなどのクラウドプロバイダーが返すAPIレスポンスをほぼそのままJSONとして保持したものです。

全フィールドをIndexしようとすると、以下の問題が起きます：

- **データ量の爆発**：AWSだけでも数百種類のリソースタイプ × 数十フィールド
- **クエリパフォーマンスの崩壊**：インデックスが多いほど書き込みが重くなる
- **スキーマ管理の複雑化**：クラウドプロバイダーのAPIは頻繁に変わる

そのためSysdigは「**よく使うフィールドだけ**」をIndexし、それ以外はJSONとして保持する設計になっています。

### 「UIで見える」≠「検索できる」

ここが**最も重要なポイント**です。

| 概念 | 説明 |
|------|------|
| 表示可能（Displayable） | JSONから取り出して画面に描画できる。Inventoryの詳細ビューで見えている値 |
| 検索可能（Indexed） | Graph / フィルタクエリで絞り込める。Indexed Fieldとして登録されている値 |

この2つは**完全に別物**です。

> 「画面に出ているのになぜ検索できないの？」

この疑問が出たとき、それは「Displayableだけど未Indexed」なフィールドを触っている状態です。

---

## Graph Searchで試す

仕組みを理解した上で、まずGraph Searchで試みることから始めましょう。

### 基本クエリ

```
MATCH CloudResource
WHERE CloudResource.type = "AWS_EBS_VOLUME"
  AND CloudResource.configuration.volumeType = "standard"
RETURN CloudResource
```

### Flattenされている場合

環境によっては `configuration.` なしで直接アクセスできるケースがあります。

```
MATCH CloudResource
WHERE CloudResource.type = "AWS_EBS_VOLUME"
  AND CloudResource.volumeType = "standard"
RETURN CloudResource
```

### 関連リソースも含めて取得する場合

どのEC2インスタンスにアタッチされているかも一緒に確認したい場合：

```
MATCH CloudResource-[*1]-RelatedResource
WHERE CloudResource.type = "AWS_EBS_VOLUME"
  AND CloudResource.configuration.volumeType = "standard"
RETURN CloudResource, RelatedResource
```

### 環境によって結果が変わる理由

Graph Searchのクエリが通るかどうかは、以下の状態によります：

| 状態 | クエリ結果 |
|------|-----------|
| `volumeType` がIndexedフィールド | ヒットする |
| `configuration.volumeType` としてIndexed | `configuration.volumeType` でヒット |
| JSONとしてのみ保持 | どちらもヒットしない |

まず両方のクエリを試して、どちらかでヒットするか確認してみてください。

---

## それでもダメな場合：現実的な回避策

### 回避策A：CSV Export → 外部フィルタ（最も確実）

**手順：**

1. InventoryでEBS Volumeを表示（type: AWS_EBS_VOLUME でフィルタ）
2. 右上のExportボタンからCSVダウンロード
3. ExcelまたはPythonで `volumeType = standard` の行を抽出

**Pythonで処理する場合：**

```python
import pandas as pd

df = pd.read_csv("sysdig_inventory_export.csv")

# volumeType列が存在する場合
magnetic = df[df["volumeType"] == "standard"]
print(f"Magnetic EBS Volume数: {len(magnetic)}")
print(magnetic[["name", "region", "accountId", "size"]])
```

**メリット：** 確実、追加設定不要
**デメリット：** 手動作業、リアルタイムでない

---

### 回避策B：Sysdig API を使う

SysdigはREST APIを提供しているため、プログラムから直接データを取得できます。

```python
import requests

SYSDIG_URL = "https://app.sysdigcloud.com"
API_TOKEN = "your-api-token"

headers = {
    "Authorization": f"Bearer {API_TOKEN}",
    "Content-Type": "application/json"
}

payload = {
    "filter": "type = 'AWS_EBS_VOLUME'",
    "limit": 1000
}

response = requests.post(
    f"{SYSDIG_URL}/api/inventory/v1/resources/query",
    headers=headers,
    json=payload
)

resources = response.json()["data"]

# configurationの中でvolumeType = standardのものを抽出
magnetic = [
    r for r in resources
    if r.get("configuration", {}).get("volumeType") == "standard"
]

print(f"Magnetic EBS Volumes: {len(magnetic)}")
for vol in magnetic:
    print(f"  - {vol['name']} ({vol['region']})")
```

**メリット：** 自動化できる、定期実行可能
**デメリット：** API仕様は環境・バージョンによって異なる場合がある

---

### 回避策C：Insights / Custom Dashboard

Sysdig InsightsはGraph Searchより柔軟なクエリが使える場合があります。

**手順：**
1. Insights → New Dashboard
2. Custom Queryでリソース条件を指定
3. Configuration値へのアクセスが通るか確認

Graph Searchと同じクエリ構文を使いますが、Insightsのほうが深いフィールドへのアクセスを許容しているケースがあります。

---

### 回避策D：タグベース運用（根本解決・継続運用向け）

上記はどれも「現状を後追いで調べる」アプローチです。継続的な管理には**設計を変える**のが最も安定した解決策になります。

**AWS側でタグを付与：**

```bash
# AWS CLIでMagnetic EBSにタグ付与
aws ec2 describe-volumes \
  --filters "Name=volume-type,Values=standard" \
  --query "Volumes[].VolumeId" \
  --output text | tr '\t' '\n' | while read vol_id; do
    aws ec2 create-tags \
      --resources "$vol_id" \
      --tags Key=StorageType,Value=magnetic
done
```

**Sysdig側でタグ検索：**

```
MATCH CloudResource
WHERE CloudResource.type = "AWS_EBS_VOLUME"
  AND CloudResource.tags["StorageType"] = "magnetic"
RETURN CloudResource
```

タグはIndexed Fieldとして扱われるため、**確実に検索できます**。

**メリット：** Sysdig上で安定して検索できる、アラートにも活用できる
**デメリット：** AWS側のタグ管理コストが発生する

---

## 手段の選び方まとめ

| 目的 | 推奨手段 | 理由 |
|------|----------|------|
| 今すぐ一回だけ調べたい | CSV Export | 確実・手軽 |
| 定期的に自動チェックしたい | API連携 | 自動化できる |
| Sysdig上でダッシュボード化したい | Insights or タグ設計 | 可視化・継続監視に向く |
| 根本的に検索しやすくしたい | タグ設計 | 最も安定 |

---

## 本質的な理解：Sysdigをどう捉えるか

最後に、この問題を通じて得られる重要な視点をまとめます。

> **SysdigのInventoryは「全部検索できるDB」ではない**
> **「検索用に最適化されたGraphビュー」である**

これはSysdigの限界ではなく、設計上のトレードオフです。

クラウド全体のリソースをリアルタイムに把握しながら、高速なクエリを実現するためには、全フィールドをIndexするわけにはいきません。

この前提を持った上でツールを使うと：

- **「なぜ検索できないのか」** で止まらなくなる
- **「どう設計すれば検索できるか」** という発想になる
- タグ設計やAPI活用を自然に取り入れられるようになる

---

## おわりに

「UIで見えているのに検索できない」はSysdigのInventory/Graph Searchを使っていると誰しもぶつかる壁です。

ポイントを3つにまとめると：

1. **DisplayableとIndexedは別物** → UIで見えても検索できるとは限らない
2. **Configuration値は検索しづらい** → 設計上の理由で全フィールドがIndexされるわけではない
3. **回避策は目的によって選ぶ** → 単発ならExport、継続運用ならタグ設計

この記事がSysdigをより使いこなすための一助になれば幸いです。

---

## 参考リンク

- [Sysdig Documentation](https://docs.sysdig.com/)
- [AWS EBS Volume Types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)
