Zscalerを使用している環境でcolima/Dockerを設定するには、主に証明書関連の問題を解決する必要があります。最新情報を探してみましょう。

次にPythonライブラリとAzure OpenAI API関連の情報を探します：

以上の検索結果を元に、Zscaler環境下でMac + colima + Docker + Python + Azure OpenAI APIを使用するための証明書設定手順をまとめます。

# Zscaler環境下でのMac開発環境設定手順

## 1. Zscaler証明書のエクスポートと設定（colima/Docker用）

1. まず、Keychain Accessを使用して「Zscaler Root CA」証明書をSystemキーチェーンから.cer形式でエクスポートします。

2. ターミナルでcerファイルをcrt形式に変換します：
```bash
openssl x509 -inform DER -in zscaler.cer -out zscaler.crt
```

3. ホームディレクトリに必要なディレクトリを作成：
```bash
mkdir -p ~/.docker/certs.d/
```

4. 変換した証明書ファイルを配置：
```bash
mv zscaler.crt ~/.docker/certs.d/
```

5. colimaを再起動します：
```bash
# Homebrewでインストールした場合
brew services restart colima

# 通常の場合
colima stop
colima start
```

## 2. Docker設定ファイルの作成

Docker設定ファイルを作成して、レジストリへの安全でない接続を許可することでも問題を軽減できます。以下の設定を`~/.colima/default/colima.yaml`に追加してみてください：

```yaml
docker:
  insecure-registries:
    - registry-1.docker.io
```

## 3. Python環境の証明書設定

Pythonプログラムで証明書エラーが発生する場合は、以下の環境変数を設定します：

1. Macのシステム証明書をファイルにエクスポート：
```bash
(security find-certificate -a -p ls /System/Library/Keychains/SystemRootCertificates.keychain && security find-certificate -a -p ls /Library/Keychains/System.keychain) > $HOME/.mac-ca-roots
```

2. 必要な環境変数を設定（.bashrcや.zshrcに追加）：
```bash
# Zscaler証明書のパスを設定
export CERT_PATH=$HOME/.docker/certs.d/zscaler.crt
export CERT_DIR=$HOME/.docker/certs.d/
export SSL_CERT_FILE=${CERT_PATH}
export SSL_CERT_DIR=${CERT_DIR}
export REQUESTS_CA_BUNDLE="$HOME/.mac-ca-roots"
export NODE_EXTRA_CA_CERTS=${CERT_PATH}
```

3. Python 3.xをインストールしている場合、Install Certificates.commandを実行します：
```bash
# Pythonがインストールされているディレクトリに応じて変更
/Applications/Python\ 3.x/Install\ Certificates.command
```

## 4. Azure OpenAI API用の設定

Azure OpenAI APIの呼び出しで証明書検証エラーが発生する場合、以下の方法で対処できます：

1. Python OpenAIライブラリを使用する場合、HTTPXクライアントで証明書検証を設定する方法：

```python
import httpx
import openai
from openai import AzureOpenAI

# 証明書ファイルのパスを指定
ssl_cert_file = "~/.docker/certs.d/zscaler.crt"

# HTTPXクライアントを作成
client = httpx.Client(verify=ssl_cert_file)

# Azure OpenAIクライアントを設定
azure_client = AzureOpenAI(
    api_key="YOUR_API_KEY",  
    azure_endpoint="https://YOUR_RESOURCE_NAME.openai.azure.com",
    api_version="2023-05-15",
    http_client=client
)
```

## 5. Dockerコンテナ内での証明書設定

Dockerコンテナ内でもZscaler証明書を信頼させる必要がある場合は、Dockerfileに以下を追加します：

```dockerfile
# Zscaler証明書をコンテナにコピー
COPY ~/.docker/certs.d/zscaler.crt /usr/local/share/ca-certificates/zscaler-root-ca.crt

# 証明書を更新
RUN apt-get update && \
    apt-get install -y ca-certificates && \
    update-ca-certificates
```

## 追加のトラブルシューティング

* 問題が解決しない場合は、証明書チェーン全体を確認してみてください。
* `curl`コマンドでもSSL検証エラーが出る場合は、システム全体の問題かもしれません。
* Docker Desktop for Macではシステムのキーチェーンを自動的に認識するはずですが、colimaは手動設定が必要な場合があります。

これらの設定を行うことで、Zscaler環境下でのPythonライブラリのインストールやAzure OpenAI APIの呼び出しなどの問題を解決できるはずです。問題が解決しない場合は、IT部門に相談することをお勧めします。