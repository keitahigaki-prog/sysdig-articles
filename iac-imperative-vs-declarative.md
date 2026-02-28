# 「こうしろ」vs「こうあるべき」IaC思想の本質と実務での使い分け

## はじめに

IaC（Infrastructure as Code）を学ぶとき、必ず出てくるのが **Imperative（命令型）** と **Declarative（宣言型）** という概念です。

これは単なる用語の違いではなく、**IaCの思想・設計・運用すべてに関わる本質的な考え方**です。

本記事では、この2つのアプローチを「実務視点」で徹底的に整理し、Terraform・Kubernetes・Ansibleの使い分けから、GitOpsの思想まで一気通貫で解説します。

---

## 一言でいうと

| 観点 | Imperative（命令型） | Declarative（宣言型） |
|------|---------------------|---------------------|
| 思考 | **手順を書く** | **状態を書く** |
| 書く内容 | 「何を・どの順で実行するか」 | 「最終的にどうあるべきか」 |
| 実行結果 | 手続き通りに処理 | 現在との差分を自動調整 |
| 代表例 | shell, AWS CLI, Ansible（task寄り） | Terraform, Kubernetes, CloudFormation |

---

## Imperative（命令型）IaC

### 考え方

**人がオペレーターになって、手順を正確に書く**

### 例（AWS CLI）

```bash
aws ec2 run-instances \
  --image-id ami-xxxx \
  --instance-type t3.micro

aws ec2 create-tags \
  --resources i-xxxx \
  --tags Key=Name,Value=web
```

👉 **1️⃣ EC2を作る → 2️⃣ その後タグを付ける**

**順番が重要**

### 特徴

**メリット**
- ✅ 分かりやすい
- ✅ 既存手作業の自動化に向く

**デメリット**
- ❌ 冪等性を自分で担保する必要あり
- ❌ 途中失敗時のリカバリが面倒
- ❌ 現在の状態を常に意識する必要あり

### 向いているケース

- 一時的な操作
- デバッグ
- 単発の変更
- 既存手順のスクリプト化

---

## Declarative（宣言型）IaC

### 考え方

**「こうなっていてほしい状態」だけを書く**

### 例（Terraform）

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxxx"
  instance_type = "t3.micro"

  tags = {
    Name = "web"
  }
}
```

👉 **「EC2がこの状態で存在すること」を宣言**

- 手順はツールが判断
- 差分は `plan` → `apply` で制御

### Kubernetes例

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
```

👉 **「Podは3つあるべき」**

- 落ちたら**自動で戻す**

### 特徴

**メリット**
- ✅ 冪等性が高い
- ✅ 差分管理ができる
- ✅ 再現性・レビュー・Git管理に最適
- ✅ スケール・復旧が自動

**デメリット**
- ❌ 学習コストあり
- ❌ 内部挙動がブラックボックスになりやすい

### 向いているケース

- 本番インフラ
- 長期運用
- チーム開発
- マルチ環境（dev / stg / prod）

---

## IaCで重要なのは「状態管理」

Declarative IaCの本質はこれ👇

> **Desired State（理想）と Actual State（現実）の差分を、ツールが埋める**

| 状況 | 結果 |
|------|------|
| リソースが無い | **作る** |
| 設定がズレている | **直す** |
| 余計なものがある | **消す** |

- **Imperativeでは、これを人間が頭でやる**
- **Declarativeでは、これをツールがやる**

---

## 現場目線の結論（重要）

### ❌ よくある失敗

- Terraform + 手動CLI
- Kubernetes + kubectl 手作業

👉 **宣言型IaCの前提が崩れる**

### ✅ ベストプラクティス

- **状態はDeclarativeに寄せる**
- **Imperativeは「補助」「一時対応」に限定**

---

## まとめ（覚え方）

- **Imperative** 👉「こうしろ → 次にこうしろ」
- **Declarative** 👉「こうなっていてほしい」

> IaC・Kubernetes・GitOps を理解するなら
> **Declarative = 正義** と思ってOKです。

---

# 深掘り編

ここからは、より実践的なテーマを掘り下げます。

---

## ① Terraform × Imperative / Declarative の「境界線」

### 結論（先に）

**Terraformは「宣言型」だが、完全な宣言型ではない**

Terraformは

- **Desired State（理想状態）を書く → 差分を解決**

という点で **Declarative** ですが、
**実行タイミング・依存関係・外部操作** に **Imperative** が入り込みます。

---

### Terraformが Declarative な部分（コア）

```hcl
resource "aws_instance" "web" {
  instance_type = "t3.micro"
}
```

- 「EC2がこの状態で存在すること」を宣言
- 作る / 直す / 消す は Terraform が判断
- 冪等性・再現性あり
- Git管理に強い

👉 **ここがTerraformの正義**

---

### Terraformに混入する Imperative 要素（境界）

#### 1️⃣ `depends_on`

```hcl
depends_on = [aws_iam_role.example]
```

👉 **本来ツールが判断すべき順序を人が明示的に命令している = Imperative的**

#### 2️⃣ `null_resource` + `local-exec`

```hcl
provisioner "local-exec" {
  command = "bash setup.sh"
}
```

👉
- Terraformで「手順」を実行している
- 再実行・失敗時の保証が弱い
- Stateとズレやすい
- **= IaCとしては危険ゾーン**

#### 3️⃣ `terraform apply` という行為自体

```bash
terraform apply
```

- **いつ適用するか**
- **誰が実行するか**

は完全に **Imperative**

👉 Terraformは
**「宣言型の設計」＋「命令型の実行モデル」**

---

### Terraform 境界の黄金ルール

#### ✅ Terraformにやらせていいこと

- リソース定義
- 依存関係の自動解決
- ライフサイクル管理

#### ❌ Terraformにやらせるべきでないこと

- OS設定
- アプリ初期化
- シェル実行
- 一回きりの手順

👉 **「Terraformは infra まで」が一番事故らない**

---

## ② Ansible はどっち問題（これは重要）

### 結論（はっきり言う）

**Ansibleは「Imperative寄り」だが「Declarative風」に書ける**

---

### Imperative な Ansible（多い）

```yaml
- name: install nginx
  apt:
    name: nginx
    state: present
```

一見 Declarative だが…

```yaml
- name: start nginx
  service:
    name: nginx
    state: started
```

👉
- タスクは**上から順に実行**
- 手順が壊れると止まる
- **実行順が意味を持つ**
- **= 命令型**

---

### Declarativeっぽく見える理由

- `state: present`
- `state: started`

👉 **「状態」を書いているように見える**

でも実際は

> **その状態にするための命令を逐次実行**

---

### 真の Declarative と Ansible の違い

| 観点 | Ansible | Terraform / Kubernetes |
|------|---------|------------------------|
| 実行モデル | タスク逐次 | 状態差分 |
| 再実行 | 壊れやすい | 安定 |
| Drift修復 | 手動 | 自動 |
| Self-healing | なし | あり |

👉
- **Ansibleは Configuration Automation**
- **Terraform/K8sは State Reconciliation**

---

### 実務での正しい使い分け

#### ✅ 正解パターン

```
Terraform → VM / LB / Network
Ansible   → OS / Package / Config
```

#### ❌ ダメパターン

- Terraform + local-exec でOS設定
- Ansible でインフラ構築

---

### 一言で言うと

> **Ansible = Imperativeの王**
>
> **Declarativeに「見せかけられる」だけ**

---

## ③ GitOps と Declarative の関係（ここが思想の完成形）

### GitOpsとは？

> **Gitに書かれた宣言状態を、システムが自動で現実に反映し続ける運用モデル**

---

### 従来（Imperative運用）

```
人 → kubectl apply
人 → terraform apply
```

- 属人化
- 再現性低い
- 「誰がやった？」

---

### GitOps（Declarative完成形）

```
Git（正） → Controller → 現実
```

- **人は Git しか触らない**
- **差分は自動適用**
- **常に Desired State に戻る**

---

### Kubernetes × GitOps の例

```yaml
replicas: 3
```

Podが1つ死んだら？

👉
1. Controllerが検知
2. 自動で3に戻す
3. **= Self-healing**

---

### GitOpsの三原則

1. **Single Source of Truth = Git**
2. **宣言状態のみを管理**
3. **差分は自動修復**

👉 **これは Declarative思想の極北**

---

### Terraform × GitOps

- **Terraform単体：宣言型 + 手動apply**
- **Terraform + GitOps：宣言型 + 自動apply + Drift修復**

👉 **初めて本物のDeclarative運用になる**

---

## 最終まとめ（思想マップ）

```
Imperative
  └─ Shell
  └─ AWS CLI
  └─ Ansible

Declarative（半分）
  └─ Terraform

Declarative（完成）
  └─ Kubernetes
  └─ GitOps
```

---

## 覚えておくべき一文

> **IaCのゴールは「書かなくていいことを増やす」こと**

- **Imperative：人が考える**
- **Declarative：ツールが考える**
- **GitOps：勝手に戻る**

---

## おわりに

IaCにおけるImperativeとDeclarativeの違いは、単なる実装方法の違いではなく、**「システムとどう向き合うか」という思想の違い**です。

現代のクラウドネイティブな運用では、Declarativeなアプローチが主流ですが、完全にImperativeを排除することは現実的ではありません。

重要なのは、**それぞれの特性を理解し、適切に使い分けること**です。

- **インフラの土台 → Terraform（Declarative）**
- **OS・ミドルウェア設定 → Ansible（Imperative的だが冪等性を意識）**
- **アプリケーション運用 → Kubernetes + GitOps（完全Declarative）**

この使い分けができれば、安全で再現性の高いインフラ運用が実現できます。

---

## 参考

- [Terraform公式ドキュメント](https://www.terraform.io/)
- [Kubernetes公式ドキュメント](https://kubernetes.io/)
- [Ansible公式ドキュメント](https://docs.ansible.com/)
- [GitOps - What is GitOps?](https://www.gitops.tech/)
