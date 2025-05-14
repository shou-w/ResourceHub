# ZscalerがインストールされたMac環境でのColima Docker開発における証明書問題の解決方法

社内環境でZscalerを使用しているMacOSでColima Dockerを活用した開発を行う際に発生する証明書関連の問題について、具体的な対処方法をまとめました。

## Zscalerが引き起こす証明書問題の概要

Zscalerはクラウドセキュリティサービスとして機能し、SSL/TLS通信を検査するために中間者（MITM）のように動作します。この過程でZscalerは独自のSSL証明書を使用するため、PythonライブラリのインストールやAzure Open AIのAPI呼び出しなど、様々な開発タスクで証明書検証エラーが発生します[3][5][9]。

## MacOSでのZscaler証明書の信頼設定

まず最初に、MacOSシステム全体でZscalerのルート証明書を信頼するように設定する必要があります：

1. Zscalerルート証明書をダウンロードします[14][19]
2. Finderを開き、「アプリケーション > ユーティリティ > キーチェーンアクセス」を選択します
3. 左側のパネルで「システム」を選択します
4. 「ファイル > 項目を読み込む」を選択し、ダウンロードした証明書ファイルをインポートします
5. インポートした「Zscaler Root CA」証明書をダブルクリックします
6. 「信頼」セクションを展開し、「この証明書を使用するとき」オプションを「常に信頼する」に設定します[18][19]
7. 管理者パスワードを入力して設定を確定します

## Python環境での証明書問題の解決

PythonのSSL証明書問題を解決するには、以下の方法があります：

### 1. pip-system-certsを使用する方法

```bash
pip install pip-system-certs
```

このパッケージをインストールすると、pipやrequestsライブラリがシステムの証明書ストアを使用するようになります[7]。

### 2. 環境変数を設定する方法

```bash
pip install certifi
```

そして、以下の環境変数を設定します：

```bash
export SSL_CERT_FILE=$(python -c "import certifi; print(certifi.where())")
export REQUESTS_CA_BUNDLE=$(python -c "import certifi; print(certifi.where())")
```

これらの設定を`.bash_profile`や`.zshrc`に追加すると便利です[8][10]。

### 3. 特定のライブラリの問題に対処する場合

特にhttpxに関連する問題（Azure Open AI SDKなど）については、バージョンを指定してインストールすることで解決できる場合があります：

```bash
pip install -U openai httpx==0.27.2
```

または、コードで明示的に証明書の場所を指定する：

```python
import os
import certifi
os.environ['SSL_CERT_FILE'] = certifi.where()
```

これにより、httpxがシステムの証明書を使用するようになります[10]。

## Colimaでの証明書問題の解決

ColimaはMac上でLinux VMを実行してDockerデーモンをホストしているため、VM内に証明書を追加する必要があります：

### 方法1：起動時に証明書をプロビジョニングする

1. まずキーチェーンアクセスからZscalerルート証明書を`.cer`形式でエクスポートします[15]
2. 証明書を`.crt`形式に変換します：
   ```bash
   openssl x509 -inform DER -in zscaler.cer -out zscaler.crt
   ```
3. 証明書をホームディレクトリに配置します
4. Colimaを停止し、設定を編集します：
   ```bash
   colima stop
   colima start --edit
   ```
5. 設定ファイルで`provision:`セクションを追加/編集します：
   ```yaml
   provision:
   - mode: system
     script: |
       cp /Users/あなたのユーザー名/zscaler.crt /usr/local/share/ca-certificates/
       update-ca-certificates
   ```
6. 保存して終了し、Colimaを起動します[17]

### 方法2：既に起動しているColimaに証明書を追加する

```bash
colima ssh -psandbox -- sudo sh -c "openssl x509 -inform PEM -in /Users/あなたのユーザー名/zscaler.crt > /usr/local/share/ca-certificates/zscaler.crt && update-ca-certificates && cat /var/run/docker.pid | xargs kill"
```

この方法では、Colimaのサンドボックスに接続し、証明書を追加してDocker demonを再起動します[12][15]。

## Dockerfileでの証明書問題の解決

Dockerビルド内でも証明書問題が発生する場合は、Dockerfileに以下を追加します：

```dockerfile
COPY zscaler.crt /usr/local/share/ca-certificates/zscaler.crt
RUN cat /usr/local/share/ca-certificates/zscaler.crt >> /etc/ssl/certs/ca-certificates.crt
```

これにより、ビルド時に証明書が追加されます[17]。

## Azure Open AIのAPI呼び出し時の対応

Azure Open AIのAPI呼び出しで証明書エラーが発生する場合は、以下の方法で対応します：

1. Pythonコードで明示的にSSL証明書の設定を行う：

```python
import os
import certifi
import ssl

# 証明書の場所を設定
os.environ['SSL_CERT_FILE'] = certifi.where()
os.environ['REQUESTS_CA_BUNDLE'] = certifi.where()

# または、SSLコンテキストを明示的に作成
ctx = ssl.create_default_context(cafile=certifi.where())

# OpenAIクライアントの設定
from openai import AzureOpenAI
client = AzureOpenAI(
    azure_endpoint="https://xxx.openai.azure.com/",
    api_key="your-api-key",
    api_version="2024-02-01"
)
```

2. httpxのバージョンを0.27.2など、証明書問題が発生しないバージョンに変更する[10]：

```bash
pip install -U openai httpx==0.27.2
```

## 結論

Zscalerが導入されたMac環境でのColima Docker開発では、証明書関連の設定が複数の層で必要になります。システムレベル、Python環境、Colima VM内、そしてDockerコンテナ内など、それぞれの層で適切に証明書を設定することで、開発作業をスムーズに進めることができます。

上記の手順を実施することで、PythonライブラリのインストールやAzure Open AIのAPI呼び出しなど、証明書検証に関連する多くの問題を解決できるはずです。問題が解決しない場合は、具体的なエラーメッセージを確認し、それに応じた追加対応が必要かもしれません。

Citations:
[1] https://help.zscaler.com/ja/zia/adding-custom-certificate-application-specific-trust-store
[2] https://preoios.cag.gov.in/downloads/zcc_Installation_win_andriod_ios_mac.pdf
[3] https://github.com/encode/httpx/discussions/3027
[4] https://github.com/abiosoft/colima/issues/131
[5] https://stackoverflow.com/questions/49324802/pip-always-fails-ssl-verification
[6] https://www.python-httpx.org/advanced/ssl/
[7] https://pypi.org/project/pip-system-certs/
[8] https://stackoverflow.com/questions/40684543/how-to-make-python-use-ca-certificates-from-mac-os-truststore
[9] https://learn.microsoft.com/en-us/answers/questions/2154760/cant-connect-to-azure-openai-service-via-private-e
[10] https://github.com/openai/openai-python/issues/2113
[11] https://www.reddit.com/r/Zscaler/comments/12p5l02/deploying_zscaler_root_certificate_on_mac/
[12] https://gist.github.com/a8143384049b171d4e64c5aeb6da4793
[13] https://learn.microsoft.com/en-sg/answers/questions/1696149/ssl-certification-error-on-app-deployed-on-app-ser
[14] https://www.asheschools.org/Page/6384
[15] https://darren.oh.name/node/81
[16] https://help.zscaler.com/zia/adding-custom-certificate-application-specific-trust-store
[17] https://hackmd.io/@redeyes2015/SybxzB40o
[18] https://ppm.smu.ca/TDClient/34/Portal/KB/PrintArticle?ID=920
[19] https://zscaler.az-ap.com/article/53/install-zscaler-root-certificate-in-macos
[20] https://help.zscaler.com/ja/client-connector/customizing-zscaler-client-connector-install-options-macos
[21] https://ichiri.biz/tech/pip-could-not-find-a-suitable-tls-ca-certificate-bundle-zscaler/
[22] https://populationgenomics.readthedocs.io/en/latest/zscaler.html
[23] https://core-docs.s3.us-east-1.amazonaws.com/documents/asset/uploaded_file/4711/ECS/4189122/Mac_OSX_Instructions.pdf
[24] https://qiita.com/j-dai/items/6a6aa010b0d59d295928
[25] https://nc01910393.schoolwires.net/cms/lib/NC01910393/Centricity/Domain/314/Flow_-_Mac.pdf
[26] https://qiita.com/ylin/items/417b3bdd70bcc05d88e8
[27] https://ichiri.biz/tech/python-ssl-certificate_verify_failed/
[28] https://neko-kawaii.blog.jp/archives/59748812.html
[29] https://www.reddit.com/r/Zscaler/comments/18wjxng/download_zscaler_root_ca_certificate/?tl=ja
[30] https://sites.google.com/a/stokes.k12.nc.us/z-scaler/zscaler-certificate-installation-instructions
[31] https://qiita.com/satoushina/items/56831655a141ec80917d
[32] https://community.f5.com/kb/technicalarticles/creating-importing-and-assigning-a-ca-certificate-bundle/286539
[33] https://github.com/abiosoft/colima/issues/256
[34] https://jiangsc.me/2023/06/24/self-signed-cert-issues-and-solutions/
[35] https://podman-desktop.io/docs/podman/adding-certificates-to-a-podman-machine
[36] https://www.thedroptimes.com/46928/resolving-ddev-tls-issues-with-colima-and-zscaler-macos
[37] https://passingcuriosity.com/2023/colima-docker-on-mac/
[38] https://yaron.github.io/2021-09-02-rancher-desktop2/
[39] https://dev.to/netoht/colima-in-macos-44g
[40] https://discourse.roots.io/t/localhost-ssl-cert-with-lima/28027
[41] https://qiita.com/zacky1972/items/8888d36aa45ab3123bcb
[42] https://github.com/lima-vm/alpine-lima/issues/54
[43] https://knowledge.broadcom.com/external/article/319418/how-to-inject-custom-ca-certificates-to.html
[44] https://qiita.com/pm00/items/9983bac9f188e73cda20
[45] https://github.com/Azure/azure-cli/issues/20921
[46] https://github.com/aminggs/import-zscaler-cert
[47] https://learn.microsoft.com/en-us/answers/questions/1390462/error-communicating-with-openai
[48] https://community.openai.com/t/ssl-error-when-using-azureopenai/1120447
[49] https://community.databricks.com/t5/data-engineering/importing-ca-certificate-into-a-databricks-cluster/td-p/7304
[50] https://docs.conda.io/projects/conda/en/stable/user-guide/configuration/non-standard-certs.html
[51] https://stackoverflow.com/questions/76262208/openai-azure-ssl-error-certification-verification-error
[52] https://learn.microsoft.com/en-us/answers/questions/1143478/az-login-fails-with-certificate-issue
[53] https://pip.pypa.io/en/stable/topics/https-certificates/
[54] https://apple.stackexchange.com/questions/76115/where-is-ruby-looking-for-ssl-cert-file
[55] https://community.openai.com/t/ssl-certificate-verify-failed-certificate-verify-failed-self-signed-certificate-in-certificate-chain/705003
[56] https://pypi.org/project/httpx/
[57] https://stackoverflow.com/questions/6340924/rails-and-ssl-cert-file-os-x
[58] https://stackoverflow.com/questions/75726455/disable-ssl-verification-on-a-third-party-module-using-httpx
[59] https://help.zscaler.com/ja/zia/adding-custom-certificate-application-specific-trust-store
[60] https://stupiddog.jp/note/archives/1266
[61] https://www.python-httpx.org/environment_variables/
[62] https://stackoverflow.com/questions/77442172/ssl-certificate-verify-failed-certificate-verify-failed-unable-to-get-local-is
[63] https://qiita.com/msi/items/9cb90271836386dafce3
[64] https://brightdata.jp/blog/%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7%E5%85%A8%E8%88%AC/httpx-with-proxies
[65] https://gist.github.com/hprobotic/0dc912f69483c3bdf578d4315249820a
[66] https://blog.symdon.info/posts/1611225717/
[67] https://cad-kenkyujo.com/python-install/
[68] https://www.dsp.co.jp/tocreator/engineer/tips-engineer/python_installation
[69] https://github.com/microsoft/promptflow/issues/2618
[70] https://learn.microsoft.com/en-us/answers/questions/1643711/how-to-add-ssl-certificate-to-azure-private-endpoi
[71] https://community.openai.com/t/ssl-certificate-verify-failed/32442
[72] https://licensecounter.jp/engineer-voice/blog/articles/20240604_azure_openai_service-private_endpoint.html
[73] https://github.com/Azure-Samples/azure-search-openai-demo/issues/2032
[74] https://zenn.dev/kazokmr/scraps/3121ca91f10e4c
[75] https://judepereira.com/blog/colima-cloudflare-zero-trust-on-apple-silicon/
[76] https://forum.ansible.com/t/storing-ca-root-certificate-on-mac-for-self-signed-certificate-fails/10566
[77] https://stackoverflow.com/questions/71073306/cannot-pull-image-from-aws-ecr-repository-using-docker-with-virtualbox-or-colima
[78] https://help.zscaler.com/zia/adding-custom-certificate-application-specific-trust-store
[79] https://github.com/pytorch/pytorch/issues/71545
[80] https://docs.docker.com/guides/zscaler/
[81] https://www.reddit.com/r/Zscaler/comments/12p5l02/deploying_zscaler_root_certificate_on_mac/
[82] https://learn.microsoft.com/en-us/answers/questions/1696149/ssl-certification-error-on-app-deployed-on-app-ser
[83] https://github.com/encode/httpx/issues/302
[84] https://www.python-httpx.org
[85] https://pypi.org/project/httpx-pkcs12/
[86] https://docs.openalgo.in/getting-started/mac-os-installation/install-certifi
[87] https://stackoverflow.com/questions/79216060/how-to-use-a-pcks12-file-with-httpx
[88] https://www.python.jp/install/macos/install.html
[89] https://qiita.com/kaizen_nagoya/items/29ac0a3c5554b0e9182a
[90] https://qiita.com/Lovely_030_Dong/items/cbf7ea5e403f695e66a2
[91] https://kredo.jp/media/mac-python-installation/
[92] https://discuss.python.org/t/mac-os-sequola-version-15-1-1-24b91-and-python-3-12-7-qnd-3-13-ssl-certificate-issues/72625
[93] https://starter-guide.od.pythonic-exam.com/ja/latest/python-setup/install-py39-mac.html
[94] https://matduggan.com/til-python-3-6-and-up-is-broken-on-mac/
[95] https://github.com/openai/openai-python/issues/1183
[96] https://qiita.com/nqdior/items/f4dfc7bd22b34c559cd3
[97] https://community.openai.com/t/ssl-certificate-verify-failed/32442?page=2
[98] https://community.fabric.microsoft.com/t5/Data-Engineering/SSL-Verification-failed-when-trying-to-use-OpenAI-library/td-p/4396386

---
Perplexity の Eliot より: pplx.ai/share