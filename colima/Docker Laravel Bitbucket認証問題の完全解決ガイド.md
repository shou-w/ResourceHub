# Docker Laravel Bitbucket認証問題の完全解決ガイド

**Consumer token**エラーの根本原因は、DockerビルドコンテキストでのHTTPS認証の失敗と、Bitbucketの認証方式変更によるものです。SSH鍵設定では解決しない理由とともに、実用的な解決策を以下に示します。

## Consumer tokenエラーの正確な原因

**主要原因**: Bitbucketは2021年9月以降、アカウントパスワードでの認証を段階的に廃止し、**App Password**またはOAuth consumer tokensのみを受け付けるように変更されました。このため、従来のusername/passwordベースの認証が「consumer token」を要求するエラーを生成しています。

**技術的背景**:
- ComposerがBitbucketプライベートリポジトリにアクセスする際、HTTPS経由でのHTTP Basic認証を試行
- Bitbucketが従来のパスワード認証を拒否し、OAuth consumer tokenまたはApp Passwordを要求
- Dockerビルド時の分離された環境では、インタラクティブな認証プロンプトが失敗

## SSH鍵設定で解決しない理由

**Dockerビルドコンテキストの制限**により、コンテナ内で作成したSSH鍵は機能しません：

1. **プロセス分離**: 各RUN命令は独立したコンテナレイヤーで実行され、SSH agentの状態が引き継がれない
2. **ネットワーク分離**: ビルドコンテナはホストのSSH agentソケットにアクセスできない
3. **セキュリティ境界**: Dockerは意図的にビルドプロセスをホストシステムから分離
4. **認証フロー**: Composerは外部リポジトリアクセス時にHTTPS認証を優先的に使用

```dockerfile
# これは失敗する例
FROM php:8.1-fpm
RUN ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
RUN composer install  # SSH agentが利用できないため失敗
```

## Docker build時のBitbucketアクセス正解手法

### 方法1: BuildKit secretsとApp Password（推奨）

**Bitbucket App Password作成**:
1. Bitbucket Settings → Access Management → App passwords
2. 必要な権限を選択: `Repositories: Read`（最小限）
3. 生成されたApp Passwordを安全に保存

**Dockerfileの実装**:
```dockerfile
# syntax=docker/dockerfile:1
FROM composer:2.5 AS deps

# Composerの認証設定をsecret mountで安全に処理
RUN --mount=type=secret,id=composer_auth \
    mkdir -p ~/.composer && \
    cp /run/secrets/composer_auth ~/.composer/auth.json && \
    composer install --no-dev --optimize-autoloader && \
    rm ~/.composer/auth.json
```

**auth.jsonファイル（ローカルに作成）**:
```json
{
  "http-basic": {
    "bitbucket.org": {
      "username": "your_bitbucket_username",
      "password": "your_app_password"
    }
  }
}
```

**ビルド実行**:
```bash
docker build --secret id=composer_auth,src=./auth.json -t your-app .
```

### 方法2: 環境変数によるComposer認証

**Docker Composeでの実装**:
```yaml
services:
  app:
    build:
      context: .
      args:
        - COMPOSER_AUTH=${COMPOSER_AUTH}
    environment:
      - COMPOSER_AUTH
```

**環境変数設定**:
```bash
export COMPOSER_AUTH='{"http-basic":{"bitbucket.org":{"username":"your_username","password":"your_app_password"}}}'
docker-compose up -d --build
```

### 方法3: Multi-stage buildでの認証分離

```dockerfile
# syntax=docker/dockerfile:1
FROM composer:2.5 AS composer
WORKDIR /app
COPY composer.json composer.lock ./

# BuildKit secretで認証情報を安全に処理
RUN --mount=type=secret,id=bitbucket_auth,target=/tmp/auth.json \
    cp /tmp/auth.json ~/.composer/auth.json && \
    composer install --no-dev --optimize-autoloader --no-scripts && \
    rm ~/.composer/auth.json

FROM php:8.1-fmp AS runtime
WORKDIR /app
# 認証情報はコピーされず、vendorディレクトリのみコピー
COPY --from=composer /app/vendor ./vendor
COPY . .
```

## composer.jsonでのプライベートリポジトリ設定

**基本設定**:
```json
{
  "repositories": [
    {
      "type": "vcs",
      "url": "https://bitbucket.org/workspace/package-name.git"
    }
  ],
  "require": {
    "vendor/package-name": "dev-main"
  },
  "config": {
    "secure-http": true,
    "github-protocols": ["https"],
    "bitbucket-oauth": {
      "bitbucket.org": {
        "consumer-key": "your-consumer-key",
        "consumer-secret": "your-consumer-secret"
      }
    }
  }
}
```

**OAuth consumer設定**（大規模プロジェクト向け）:
```json
{
  "config": {
    "bitbucket-oauth": {
      "bitbucket.org": {
        "consumer-key": "YOUR_CONSUMER_KEY",
        "consumer-secret": "YOUR_CONSUMER_SECRET"
      }
    }
  }
}
```

## 認証情報の適切な受け渡し方法

### セキュリティレベル別の実装

**レベル1: 最高セキュリティ（本番環境推奨）**
```bash
# BuildKit secretsを使用
docker build --secret id=composer_auth,src=./auth.json .
```

**レベル2: 良好なセキュリティ（開発環境）**
```bash
# 環境変数を使用（値は事前に設定）
docker build --build-arg COMPOSER_AUTH="$COMPOSER_AUTH" .
```

**レベル3: 基本セキュリティ（ローカル開発のみ）**
```bash
# ファイルマウント（最終イメージには残らない）
docker run -v ~/.composer/auth.json:/tmp/auth.json:ro composer:install
```

## OAuth App PasswordsとApp Passwordsの使用方法

### App Passwordの推奨設定手順

**1. Bitbucket App Password作成**:
```bash
# 必要な権限スコープ:
# - Repositories: Read (必須)
# - Account: Read (推奨)
# - Pull requests: Read (Webhookを使用する場合)
```

**2. Composerでの設定**:
```bash
# グローバル設定
composer config --global http-basic.bitbucket.org username app_password

# プロジェクト固有設定  
composer config http-basic.bitbucket.org username app_password
```

**3. Docker環境での利用**:
```bash
# セキュアな環境変数として設定
export BITBUCKET_USER="your_username"
export BITBUCKET_APP_PASSWORD="generated_app_password"
export COMPOSER_AUTH="{\"http-basic\":{\"bitbucket.org\":{\"username\":\"$BITBUCKET_USER\",\"password\":\"$BITBUCKET_APP_PASSWORD\"}}}"
```

## colima特有の注意点

### Docker Context管理
```bash
# colimaコンテキストの確認
docker context ls
docker context use colima

# 環境変数の設定（必要に応じて）
export DOCKER_HOST="unix://$HOME/.colima/default/docker.sock"
```

### ボリュームマウントの制限
```yaml
# ~/.colima/default/colima.yaml
mounts:
  - location: ~/development
    writable: true
  - location: /path/to/project
    writable: true
```

### SSH Agent転送（補完的解決策）
```bash
# SSH agent転送を有効にしてcolima起動
colima start --ssh-agent

# コンテナでSSH agent使用（runtime時のみ有効）
docker run -v $SSH_AUTH_SOCK:/ssh-agent -e SSH_AUTH_SOCK=/ssh-agent ubuntu
```

## 設定ファイルの完全な実装例

**auth.json**（グローバル設定: ~/.composer/auth.json）:
```json
{
  "http-basic": {
    "bitbucket.org": {
      "username": "your_bitbucket_username",
      "password": "your_app_password"
    }
  }
}
```

**.netrcファイル**（補完的手法）:
```
machine bitbucket.org
login your_username
password your_app_password
```

**docker-compose.yml**（完全版）:
```yaml
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      secrets:
        - composer_auth
    volumes:
      - .:/var/www/html
    environment:
      - COMPOSER_CACHE_DIR=/tmp/composer-cache

secrets:
  composer_auth:
    file: ./auth.json
```

## 具体的な解決手順

**即座に実行可能な解決策**:

```bash
# 1. Bitbucket App Password作成（Web UIで実行）
# 2. 認証ファイル作成
cat > auth.json << EOF
{
  "http-basic": {
    "bitbucket.org": {
      "username": "YOUR_BITBUCKET_USERNAME",
      "password": "YOUR_APP_PASSWORD"
    }
  }
}
EOF

# 3. Dockerfileを修正
cat > Dockerfile << 'EOF'
# syntax=docker/dockerfile:1
FROM php:8.1-fpm

# Composerインストール
COPY --from=composer:2.5 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html
COPY composer.json composer.lock ./

# 認証情報を安全に処理してComposer install
RUN --mount=type=secret,id=composer_auth,target=/tmp/auth.json \
    mkdir -p ~/.composer && \
    cp /tmp/auth.json ~/.composer/auth.json && \
    composer install --no-dev --optimize-autoloader && \
    rm ~/.composer/auth.json

COPY . .
EOF

# 4. ビルド実行
docker build --secret id=composer_auth,src=./auth.json -t laravel-app .

# 5. 認証ファイルを削除（セキュリティのため）
rm auth.json
```

**検証コマンド**:
```bash
# 認証設定の確認
composer config --global --list | grep bitbucket

# プライベートリポジトリアクセステスト
composer show --available | grep your-vendor

# Docker buildの詳細ログ確認
docker build --secret id=composer_auth,src=./auth.json -t test --progress=plain .
```

## トラブルシューティング

**よくあるエラーと解決策**:

```bash
# Consumer tokenエラーが継続する場合
composer clear-cache
docker system prune -f

# Bitbucket接続テスト
curl -u "username:app_password" https://api.bitbucket.org/2.0/user

# SSH接続テスト（補完的確認）
ssh -T git@bitbucket.org
```

**この実装により、colimaでのDocker環境においても、Bitbucketプライベートリポジトリへの安全で確実なアクセスが実現できます。**