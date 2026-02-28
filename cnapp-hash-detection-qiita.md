# セキュリティ製品は"設計思想"で使え
〜Hash検知が漏れた事象から考えるCNAPP運用のベストプラクティス〜

## はじめに

セキュリティ製品を導入したとき、よく聞く言葉があります。

> 「とりあえず有効化しました」
>
> 「エラーは出ていません」
>
> 「検知は上がっています」

でも、これは安心材料ではありません。

セキュリティ製品は **"動いていること"がゴールではない**。
**設計思想に沿って使えているかどうか** がすべてです。

この記事では、実際に起きたマルウェア検知の事象をもとに、

- Hash検知の限界
- データ品質問題の本質
- なぜベストプラクティスが重要なのか

を技術的に整理します。

---

## 前提：SysdigのCNAPPとマルウェア検知の全体構成

Sysdig Secureは **CNAPP（Cloud Native Application Protection Platform）** として、
ビルド・レジストリ・ランタイムをまたぐ統合セキュリティプラットフォームです。

マルウェア検知は主に **ランタイムレイヤー** で機能しており、以下の3つの検知エンジンで構成されています。

| 検知方式 | 仕組み | 特徴 |
|---|---|---|
| **Hash検知** | ファイルのSHA-256をハッシュDBと照合 | 既知マルウェアの高速検知。DBに存在しないものは検知不可 |
| **YARA検知** | ファイル内容のパターンマッチング | 亜種・バリアントに対応。カスタムルール追加可能 |
| **Runtime挙動検知** | Falcoによるsyscallレベルの動的検知 | ゼロデイ・LotL攻撃に対応。ハッシュ非依存 |

<!-- スクリーンショット: Sysdig Secure の Malware Detection ポリシー設定画面全体 -->
![Sysdig Secure - Malware Detection Policy](./images/ss-01-malware-policy-overview.png)
*Sysdig Secure の Malware Detection ポリシー全体像（Use Hashes / Use YARA / Behavioral の各スイッチ）*

---

## 実例：Hash検知のみ有効にしたら検知されなかった

ある検証で、Malware Detectionポリシーを以下の構成にしました。

| 設定項目 | 状態 |
|---|---|
| Malware Control | **有効** |
| Use Hashes | **有効** |
| Use YARA | 無効 |

<!-- スクリーンショット: Use Hashes のみ ON、Use YARA が OFF の状態のポリシー設定画面 -->
![Malware Policy - Hash Only](./images/ss-02-policy-hash-only.png)
*Use Hashes のみ有効、Use YARA は無効の状態*

テストとして **EICAR（`elf_eicar`）** を実行。

:::note info
**EICAR テストファイルとは**
EICAR（European Institute for Computer Antivirus Research）が策定した、セキュリティ製品の動作確認用の標準的なテストファイルです。無害ですが、ほぼすべてのセキュリティ製品がマルウェアとして認識する約束になっています。
`elf_eicar` は Linux ELF バイナリ版であり、コンテナ環境での検知テストに用います。
:::

**結果：検知されない**

デバッグログには、以下が出力されていました。

```
Hash 'e1faac29ddf23aca2e8065309e76d252e6...' not matching in DB
```

<!-- スクリーンショット: デバッグログの出力（not matching in DB の行が見える状態） -->
![Debug Log - Hash not matching](./images/ss-03-debug-log-hash-not-found.png)
*デバッグログ。エージェントは正常動作しているが、ハッシュDBにエントリが存在しない*

---

## 何が起きていたのか？

整理すると、以下はすべて **正常** でした。

```
✅ エージェント起動
✅ Runtimeエンジン稼働
✅ イベント取得（syscall レベル）
✅ ポリシー有効化
```

にもかかわらず、

> **バックエンドのハッシュDBに想定ハッシュが存在していなかった**

という状態でした。

これは：

- エンジン障害ではない
- 改ざんでもない
- 配布破損でもない

**検知データの内容品質問題** です。

原因として構造的に考えられるのは以下のいずれかです。

| 原因候補 | 概要 |
|---|---|
| DB登録漏れ | 脅威インテリジェンスフィードへの登録が未実施 |
| データ同期不備 | フィードは存在するがSysdigのDBへの同期が未完了 |
| ローテーション処理ミス | DB更新時の処理バグで該当エントリが欠落 |
| テナント反映遅延 | マルチテナント構成でのロールアウト遅延 |
| Canary配布段階差 | 段階的配布の途中で特定テナントに未反映 |

いずれにせよ、これは **SaaSバックエンドの品質管理プロセス** に属する問題です。

---

## Hash検知の仕組みを正しく理解する

Hash型マルウェア検知のロジックは構造上シンプルです。

```python
# Hash型検知の擬似コード
file_hash = sha256(target_file)

if file_hash in malicious_hash_db:
    alert(severity="HIGH", hash=file_hash)
else:
    # DBに存在しなければ何も起きない
    pass
```

つまり、

> **DBに存在するものしか検知できない**

という性質を根本的に持ちます。

これは製品の欠陥ではなく、**シグネチャ型検知の本質的制約** です。

### シグネチャ型 vs 挙動型の比較

| 比較軸 | Hash検知 | YARA検知 | Runtime挙動検知（Falco） |
|---|---|---|---|
| 既知マルウェア検知 | ◎ 高速 | ○ | ○ |
| 亜種・変形への対応 | ✗ 不可 | ○ パターン次第 | ◎ 挙動は変わらない |
| ゼロデイ攻撃 | ✗ 不可 | △ | ◎ |
| LotL（環境寄生型）攻撃 | ✗ 不可 | △ | ◎ |
| 処理コスト | 低 | 中（ファイルサイズ依存） | 中（syscall監視） |
| DB/フィード依存 | **強**（これが今回の問題） | 中 | 弱 |

---

## YARA検知：パターンマッチングで亜種をカバーする

YARAはファイルの **内容パターン** でマルウェアを識別します。
同じ攻撃ツールの亜種でも、共通する文字列・バイト列・構造を持つことが多く、
Hash検知が無力なシナリオでも有効です。

```yara
rule CryptoMiner_XMRig {
    meta:
        description = "XMRig 系マイナーを検出"
        severity    = "HIGH"

    strings:
        $s1 = "stratum+tcp://" ascii wide
        $s2 = "xmrig"          ascii nocase
        // Monero ウォレットアドレスパターン
        $re1 = /[0-9a-zA-Z]{95}=/

    condition:
        2 of ($s*) or $re1
}
```

Sysdigでは **管理されたYARAルールセット**（Sysdig Threat Research Teamが管理）に加え、
カスタムルールのアップロードにも対応しています。

<!-- スクリーンショット: Sysdig Secure の YARA ルール設定画面（カスタムルールアップロード箇所など） -->
![Sysdig Secure - YARA Rules](./images/ss-04-yara-rules.png)
*Sysdig Secure の YARA ルール管理画面*

---

## Falcoによるランタイム挙動検知：ハッシュに頼らない検知

Falcoは **Linux syscall を監視** し、異常な挙動をルールベースで検出します。
実行ファイルの中身やハッシュとは完全に独立しており、
「その行為自体が怪しい」ことを検知します。

```yaml
# Falco ルール例：コンテナ内でシェルが起動された
- rule: Terminal shell in container
  desc: コンテナ内でシェルが起動された（侵入の可能性）
  condition: >
    spawned_process and container and
    shell_procs and proc.tty != 0
  output: >
    コンテナ内でシェル起動 (user=%user.name container=%container.name
    image=%container.image.repository shell=%proc.name parent=%proc.pname)
  priority: WARNING
  tags: [container, shell, T1059]
```

Falcoが検知できる代表的な攻撃パターン：

| 検知シナリオ | Falcoルール例 |
|---|---|
| コンテナ内でのシェル起動 | `spawned_process and shell_procs` |
| 機密ファイルの読み取り | `/etc/shadow`, `.ssh/` アクセス |
| バイナリディレクトリへの書き込み | `/usr/bin`, `/bin` への実行時書き込み（Drift Detection） |
| C2サーバーへの不審な外部接続 | マイニングプールポート（3333, 4444）への接続 |
| 権限昇格 | `setuid`, `ptrace` 悪用 |

:::note info
**Drift Detection（ドリフト検知）とは**
コンテナの起動時イメージに存在しなかったバイナリが実行時に現れた場合に検知する機能です。
侵入後のバイナリ設置をほぼ確実に捕捉でき、ハッシュDB非依存でシグナル品質が高いため、
ベースラインポリシーとして最初に有効化することが推奨されています。
:::

<!-- スクリーンショット: Falco の検知アラート画面（イベント一覧など） -->
![Falco Detection Alert](./images/ss-05-falco-alert.png)
*Falcoによるランタイム検知アラートの例*

---

## ここでよく出る誤解

> 「署名を付ければ防げるのでは？」

これは **次元が違います**。

### 署名（デジタル署名）と内容品質は別問題

署名が保証するのは：

- **配布物が改ざんされていないこと**（＝配布後に誰かが書き換えていない）

今回の問題は：

- **DB自体の中身が最初から不完全だった**（＝登録されていないエントリは検知できない）

```
正しい署名付きの不完全なDB
  └─ ハッシュは一致（改ざんなし）
  └─ しかし中身は不完全（eicarのハッシュが未登録）
  └─ 結果：検知されない
```

仮に不完全なDBに正しい署名が付いても、ハッシュは一致します。
**内容の妥当性は保証されません。**

---

## 検知モデルの全体像：Defense in Depth

CNAPP / Runtime製品は **多層防御（Defense in Depth）** の思想で設計されています。

```
┌──────────────────────────────────────────┐
│            CNAPP 検知モデル              │
├──────────────────────────────────────────┤
│  Layer 1: Hash検知                       │
│    └─ 既知マルウェアのシグネチャ照合      │
│    └─ 高速・低コスト                     │
│    └─ DB品質に依存（今回の問題箇所）     │
├──────────────────────────────────────────┤
│  Layer 2: YARA検知                       │
│    └─ パターンマッチング                  │
│    └─ 亜種・バリアントに対応             │
│    └─ カスタムルール拡張可               │
├──────────────────────────────────────────┤
│  Layer 3: Runtime挙動検知（Falco）       │
│    └─ syscallレベルの動的監視            │
│    └─ ゼロデイ・LotL攻撃に対応          │
│    └─ ハッシュDB非依存                  │
├──────────────────────────────────────────┤
│  Layer 4: ポリシー相関・SIEM連携         │
│    └─ 上記を統合した判断・自動対応       │
└──────────────────────────────────────────┘
```

**Hashのみ有効化する構成は、単一レイヤー依存** となり、設計思想から逸脱します。

<!-- スクリーンショット: Sysdig Secure の推奨ポリシー（Hash + YARA + Runtime すべて有効）の設定画面 -->
![Recommended Policy - All Enabled](./images/ss-06-policy-all-enabled.png)
*推奨構成：Hash + YARA + Runtime すべて有効*

---

## ベストプラクティスの意味

ベストプラクティスは、以下を前提に設計されています。

- スケール前提（数万台規模）
- マルチテナント前提
- インシデント対応前提
- **過去事故の再発防止前提**

Hash + YARA + Runtime併用は "推奨" ではなく、

> **想定事故を前提とした設計**

です。

---

## スケール視点で見るともっと怖い

小規模環境では気づきにくい。しかし **20,000台規模** になると：

| 問題 | 内容 |
|---|---|
| DBローテーションのタイミング差 | 全ノードへの反映が完了するまでの間、一部ノードで検知漏れ |
| キャッシュ不整合 | ローカルキャッシュと最新DBの乖離 |
| 部分反映 | Canary配布途中で特定テナントだけ旧DBのまま |

設計思想を無視すると、**スケールで破綻する**。

---

## インシデント対応時に何が起きるか？

設計通りに入れていないと：

| 問題 | 影響 |
|---|---|
| 相関不能 | Hash / Runtime 双方のログがないため証跡が繋がらない |
| 証跡不足 | 「何が起きていたか」の再構成ができない |
| 原因特定困難 | 単一レイヤーが漏れた場合に他レイヤーでカバーできない |
| 責任分界曖昧 | ベンダー側の問題か運用側の設定問題か切り分け困難 |

「検知していました」では守れません。

<!-- スクリーンショット: Sysdig Secure のイベント相関画面 or インシデントタイムライン -->
![Incident Timeline](./images/ss-07-incident-timeline.png)
*多層検知が有効な場合のインシデントタイムライン例（相関が可能な状態）*

---

## ではどうすればよいか？

### ✅ 1. 単一検知方式に依存しない

Hash + YARA + Runtime を **全て有効** にする。

### ✅ 2. まずDrift Detectionを有効化する

ノイズが少なく、シグナル品質が高い。侵入後のバイナリ設置をほぼ確実に捕捉できる。

```yaml
# Drift Detection はシンプルかつ強力
# コンテナイメージに存在しなかったバイナリの実行を検知
```

<!-- スクリーンショット: Drift Detection の有効化設定 -->
![Drift Detection Setting](./images/ss-08-drift-detection.png)
*Drift Detection の有効化設定画面*

### ✅ 3. 検知モデルを理解する

静的検知とRuntime検知は別物。それぞれの役割を把握した上で設定する。

### ✅ 4. EICARテストで検知を確認する

```bash
# コンテナ内でELF EICARを実行して検知を確認
docker run --rm -it your-image /bin/sh -c "wget -O /tmp/elf_eicar <elf_eicar_url> && chmod +x /tmp/elf_eicar && /tmp/elf_eicar"
```

<!-- スクリーンショット: EICAR 実行後に Sysdig が検知アラートを発生させた画面 -->
![EICAR Detection Alert](./images/ss-09-eicar-detected.png)
*EICAR 実行後の検知アラート（正常な設定ならここに表示される）*

### ✅ 5. ベストプラクティスを"理由ごと"理解する

なぜ推奨されているのかを理解した上で運用する。

---

## 参考リンク

| リソース | URL |
|---|---|
| Sysdig Secure ドキュメント | https://docs.sysdig.com/en/docs/sysdig-secure/ |
| Sysdig Malware Detection | https://docs.sysdig.com/en/docs/sysdig-secure/policies/threat-detect-policies/manage-rules/malware-detection/ |
| Falco 公式ドキュメント | https://falco.org/docs/ |
| Falco ルールリポジトリ（GitHub） | https://github.com/falcosecurity/rules |
| YARA 公式ドキュメント | https://yara.readthedocs.io/en/stable/ |
| YARA GitHub | https://github.com/VirusTotal/yara |
| EICAR 公式 | https://www.eicar.org/ |
| MITRE ATT&CK for Containers | https://attack.mitre.org/matrices/enterprise/containers/ |
| Sysdig Threat Research Blog | https://sysdig.com/blog/threat-research/ |
| Neo23x0 YARA ルール集（高品質） | https://github.com/Neo23x0/signature-base |

---

## 結論

セキュリティ製品は裏切りません。

裏切るのは：

- 「とりあえず有効化」
- 思い込み
- 設計思想の無視

ベストプラクティスは縛りではありません。

> **未来の自分を守るための設計図です。**

---

## おわりに

「動いている」は合格ではない。

**「設計思想に沿っている」が合格です。**

今回のHash検知漏れは、

- 製品が壊れた話ではなく
- **設計と品質の話**

でした。

セキュリティ製品は **思想で使うもの** です。
