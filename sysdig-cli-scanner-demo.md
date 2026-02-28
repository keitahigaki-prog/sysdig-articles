# Sysdig CLI Scanner Demo 手順書

## 概要
Sysdig CLI Scannerは、コンテナイメージやIaC（Infrastructure as Code）の脆弱性スキャンを行うためのコマンドラインツールです。

## 前提条件
- Docker がインストールされていること
- インターネット接続があること（DBダウンロード用）
- macOS / Linux / Windows 環境

## デモ手順

### 1. Sysdig CLI Scanner のインストール

#### 作業ディレクトリの作成
```bash
# 専用ディレクトリを作成
mkdir -p ~/sysdig-scanner
cd ~/sysdig-scanner
```

#### macOS (Intel/AMD64) の場合:
```bash
# 最新バージョンの取得
LATEST_VERSION=$(curl -L -s https://download.sysdig.com/scanning/sysdig-cli-scanner/latest_version.txt)

# ダウンロード
curl -LO "https://download.sysdig.com/scanning/bin/sysdig-cli-scanner/${LATEST_VERSION}/darwin/amd64/sysdig-cli-scanner"

# 実行権限の付与
chmod +x ./sysdig-cli-scanner

# バージョン確認
./sysdig-cli-scanner --version
```

#### macOS (Apple Silicon/ARM64) の場合:
```bash
# 最新バージョンの取得
LATEST_VERSION=$(curl -L -s https://download.sysdig.com/scanning/sysdig-cli-scanner/latest_version.txt)

# ダウンロード
curl -LO "https://download.sysdig.com/scanning/bin/sysdig-cli-scanner/${LATEST_VERSION}/darwin/arm64/sysdig-cli-scanner"

# 実行権限の付与
chmod +x ./sysdig-cli-scanner

# バージョン確認
./sysdig-cli-scanner --version
```

#### Linux (AMD64) の場合:
```bash
# 最新バージョンの取得
LATEST_VERSION=$(curl -L -s https://download.sysdig.com/scanning/sysdig-cli-scanner/latest_version.txt)

# ダウンロード
curl -LO "https://download.sysdig.com/scanning/bin/sysdig-cli-scanner/${LATEST_VERSION}/linux/amd64/sysdig-cli-scanner"

# 実行権限の付与
chmod +x ./sysdig-cli-scanner

# バージョン確認
./sysdig-cli-scanner --version
```

### 2. 基本的な使い方の確認

```bash
# ヘルプ表示（~/sysdig-scanner ディレクトリ内で実行）
./sysdig-cli-scanner --help
```

### 3. APIトークンの設定

Sysdig Secure から取得したAPIトークンを環境変数に設定します。

```bash
export SECURE_API_TOKEN="your-api-token-here"
```

### 4. イメージスキャンの実行

#### 4-1. 軽量イメージのスキャン (alpine)

```bash
# Alpine Linux の最新イメージをスキャン
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com alpine:latest
```

初回実行時は脆弱性データベースのダウンロードが行われます（約215MB、数分かかる場合があります）。

#### 4-2. より複雑なイメージのスキャン (nginx)

```bash
# Nginx 最新イメージをスキャン
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com nginx:latest
```

#### 4-3. 古いバージョンのイメージスキャン（脆弱性が多い例）

```bash
# 古いバージョンのイメージをスキャン（脆弱性が検出される可能性が高い）
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com nginx:1.20
```

### 5. スキャン結果の出力オプション

#### 5-1. JSON形式で結果を出力

```bash
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --output=json-file=scan-result.json nginx:latest
```

#### 5-2. 詳細な脆弱性リストを表示

```bash
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --full-vulns-table nginx:latest
```

#### 5-3. レイヤー別の詳細情報を表示

```bash
cd ~/sysdig-scanner
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --separate-by-layer nginx:latest
```

### 6. スキャン結果の確認

スキャン実行後、以下の情報が表示されます：

- **パッケージ数**: スキャンされたパッケージの総数
- **脆弱性数**: 検出された脆弱性の数（Severity別）
  - Critical: 重大
  - High: 高
  - Medium: 中
  - Low: 低
- **修正可能な脆弱性**: アップデートで修正可能な脆弱性の数
- **ポリシー評価結果**: Pass/Fail

### 7. ログとキャッシュの管理

```bash
cd ~/sysdig-scanner

# キャッシュをクリアしてスキャン
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --clearcache alpine:latest

# ログレベルを変更（debug, info, warn, error）
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --loglevel=debug alpine:latest

# ログファイルを指定
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com --logfile=./scan.log alpine:latest
```

### 8. ローカルのDockerイメージをスキャン

```bash
cd ~/sysdig-scanner

# ローカルでビルドしたイメージをスキャン
docker build -t myapp:latest .
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com myapp:latest
```

## Exit コード

- **0**: スキャン評価 "pass"
- **1**: スキャン評価 "fail"
- **2**: 無効なパラメータ
- **3**: 内部エラー

## ディレクトリ構成

```
~/sysdig-scanner/
├── sysdig-cli-scanner         # スキャナー実行ファイル
├── main.db/                    # 脆弱性データベース（自動生成）
├── scan-logs                   # スキャンログ（自動生成）
└── *.json                      # スキャン結果（JSON出力時）
```

## トラブルシューティング

### データベースのエラー
```bash
cd ~/sysdig-scanner

# データベースを再ダウンロード
rm -rf main.db
./sysdig-cli-scanner --apiurl=https://us2.app.sysdig.com alpine:latest
```

### キャッシュのクリア
```bash
cd ~/sysdig-scanner

# キャッシュディレクトリをクリア
./sysdig-cli-scanner --clearcache --apiurl=https://us2.app.sysdig.com alpine:latest
```

## 参考情報

- Sysdig CLI Scanner 公式ドキュメント: https://docs.sysdig.com/en/docs/sysdig-secure/vulnerabilities/scanning/
- ダウンロードページ: https://download.sysdig.com/scanning/sysdig-cli-scanner/

## デモのポイント

1. **インストールの簡単さ**: シングルバイナリで依存関係なし
2. **スタンドアロンモード**: バックエンド接続不要でローカル実行可能
3. **包括的なスキャン**: OS パッケージ、言語ライブラリの脆弱性検出
4. **詳細なレポート**: Severity別の脆弱性、修正可能性の情報
5. **CI/CD 統合**: Exit コードによる自動化対応
