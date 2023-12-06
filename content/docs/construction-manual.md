+++
title = '週7英単語 サーバ構築手順書'
weight = 30
draft = false
+++
# 週7英単語 サーバ構築手順書

ここでは、[週7英単語](https://shu7-eitango.com)のサーバの構築手順を説明いたします。

一部の具体的な設定値は手順書用にすり替えていますが、手続きについては実際に行ったものと変わりません。

## 1. 前提条件

使用している技術を表にまとめると次のとおりです。

| 分類 | 技術スタック |
| ----------------- | ------------------------------ |
| Frontend          | Bootstrap                      |
| Backend           | Nginx, uWSGI, Flask, Python    |
| Infrastructure    | Amazon EC2, Cloudflare         |
| Database          | MySQL, Redis, Grafana Loki     |
| Monitoring        | Grafana                        |
| Environment setup | Docker Compose                 |
| CI/CD             | GitHub Actions                 |
| Design            | PowerPoint                     |
| etc.              | Promtail, Certbot, geoipupdate |

VPSとしてAmazon EC2を利用しています。  
AWSの構成は非常にシンプルで、以下の図のようになっています。

[![AWS構成図](https://raw.githubusercontent.com/kyasuda516/shu7-eitango/main/docs/img/awsconfig.svg)](https://github.com/kyasuda516/shu7-eitango/blob/main/docs/img/awsconfig.svg)

このEC2インスタンスのなかでDocker Composeプロジェクトを起動しています。Composeプロジェクトの構成は下図のとおりです。

[![Composeプロジェクトの構成図](https://raw.githubusercontent.com/kyasuda516/shu7-eitango/main/docs/img/prjconf.svg)](https://github.com/kyasuda516/shu7-eitango/blob/main/docs/img/prjconf.svg)

## 2. 構築手順
### 2.1 システムの準備
#### 2.1.1 VPSの用意

AWSのVPCコンソールから、VPCを以下の設定で作成します（以下における具体的な設定値は手順書用で実際と異なるものがあります）。
- 作成するリソース: VPCなど
- 名前タグの自動生成: 自動生成ON、`sh7e`
- IPv4 CIDRブロック: `172.16.0.128/26`
- IPv6 CIDRブロック: IPv6 CIDRブロックなし
- テナンシー: デフォルト
- AZの数: 1
- AZのカスタマイズ: 第1AZ: ap-northeast-1a
- パブリックサブネットの数: 1
- プライベートサブネットの数: 1
- サブネットCIDRブロックをカスタマイズ:
    - ap-northeast-1aのパブリックサブネットCIDRブロック: `172.16.0.128/28`
    - ap-northeast-1aのプライベートサブネットCIDRブロック: `172.16.0.160/28`
- NATゲートウェイ: なし
- VPCエンドポイント: なし
- DNSオプション: DNSホスト名を有効化ON、DNS解決を有効化ON

同じくVPCコンソールから、セキュリティグループを以下の設定で作成します。
- 基本的な詳細:
  - セキュリティグループ名: `sh7e-compose-sgr`
  - 説明: for the server running Docker Compose in sh7e
  - VPC: `sh7e-vpc`
- インバウンドルール:
  |  | タイプ | ソース |
  | ---- | ---- | ---- |
  | (1) | SSH | Anywhere-IPv4 |
  | (2) | HTTP | Anywhere-IPv4 |
  | (3) | HTTP | Anywhere-IPv6 |
  | (4) | HTTPS | Anywhere-IPv4 |
  | (5) | HTTPS | Anywhere-IPv6 |
- アウトバウンドルール:
  |  | タイプ | 送信先 |
  | ---- | ---- | ---- |
  | (1) | すべてのトラフィック | Anywhere-IPv4 |
  | (2) | すべてのトラフィック | Anywhere-IPv6 |
- タグ:
  | キー | 値 |
  | ---- | ---- |
  | Name | `sh7e-compose-sgr` |

続いてEC2コンソールから、EC2インスタンスを以下の設定で作成します。
- 名前: `sh7e-compose-server`
- Application and OS Images
  - AMI: Ubuntu Server 20.04 LTS (HVM), SSD Volume Type
  - アーキテクチャ: 64ビット (x86)
- インスタンスタイプ: t3.small
- キーペア: `sh7e-ec2` (というキーペアを作成し、ローカルにダウンロードしておく)
- ネットワーク設定:
  - VPC: `sh7e-vpc`
  - サブネット: (パブリックなサブネットを選択)
  - パブリックIPの自動割り当て: 無効化
  - ファイアウォール: 既存のセキュリティーグループを選択、`sh7e-compose-sgr`
- ストレージを設定: 1x 30GiB gp3

同じくEC2コンソールから、以下の設定でElastic IPアドレスをアカウントに割り当てます。
- Elastic IP アドレスの設定:
  - ネットワークボーダーグループ: ap-northeast-1
  - パブリックIPv4アドレスプール: AmazonのIPv4アドレスプール
- タグ:
  | キー | 値 |
  | ---- | ---- |
  | Name | `sh7e-compose-eip` |

そして、このElastic IPアドレスを以下の設定でインスタンスに関連付けします。
- リソースタイプ: インスタンス
- インスタンス: `sh7e-compose-server`

#### 2.1.2 ドメインの取得

ドメインレジストラとしてCloudflareレジストラを利用します。

[Cloudflareによるドメインの登録と管理](https://www.cloudflare.com/ja-jp/products/registrar/)より、`shu7-eitango.com`というドメインを取得します。

#### 2.1.3 CDNの設定

CDNとしてCloudflare CDNを利用します。

[CloudflareのCDN紹介ページ](https://www.cloudflare.com/ja-jp/application-services/products/cdn/)よりCloudflareダッシュボードのホームに飛び、Webサイトとして`shu7-eitango.com`を追加します。なお、料金プランにはFreeを選択します。

それから、追加した`shu7-eitango.com`を管理する画面に行き、DNSレコードについて次のように設定します。ただし、Aタイプレコードのコンテンツ（画像で伏せている部分）には、先ほどEC2インスタンスに関連付けしたElastic IPのIPv4アドレスを設定してください。

![Cloudflare DNSレコード設定](/portfolio/img/cloudflare_dns_records.png)

### 2.2 セキュリティ設定

（次の操作は**ブラウザ上のEC2コンソールから**サーバに接続したうえで行います）  
以下のコマンドを実行して、メインで扱うユーザを新規作成します（ここではユーザ名を`yamada`としています）。
```bash
$ NEWUSER=yamada
$ sudo useradd -m $NEWUSER
$ sudo usermod -aG sudo $NEWUSER
$ sudo passwd $NEWUSER  # このあと好きなパスワードを入力
$ sudo chsh -s /bin/bash $NEWUSER
$ sudo -u $NEWUSER mkdir /home/$NEWUSER/.ssh/
$ sudo -u $NEWUSER sh -c "sudo cat /home/${USER}/.ssh/authorized_keys >> /home/${NEWUSER}/.ssh/authorized_keys"
$ exit
```
<!-- 
NEWUSER=yamada && sudo useradd -m $NEWUSER && sudo usermod -aG sudo $NEWUSER && sudo passwd $NEWUSER
sudo chsh -s /bin/bash $NEWUSER && sudo -u $NEWUSER mkdir /home/$NEWUSER/.ssh/ && sudo -u $NEWUSER sh -c "sudo cat /home/${USER}/.ssh/authorized_keys >> /home/${NEWUSER}/.ssh/authorized_keys"
-->
（ここまでブラウザで）

次のテキストをローカルの`~/.ssh/config`ファイルに追記します（コメントに従って値を適宜変更し、コメントは削除してください）。
```conf
Host shu7-eitango
  HostName X.X.X.X  # サーバのパブリックIPに
  User yamada  # 先ほど新規作成したユーザ名に
  IdentityFile /path/to/key  # ダウンロードしたSSH鍵(sh7e-ec2)のファイルパスに
```

次のコマンドで、先ほどのユーザとして**ローカルのシェルから**SSH接続します。
```shell
$ ssh shu7-eitango
```

以降は、SSH接続した状態で行います。

### 2.3 ソフトウェアのインストールと設定

#### 2.3.1 GitHubリポジトリをクローン

次のコマンドにより、GitHubリポジトリをクローンし、クローン先のディレクトリに移動します。
```bash
$ cd $HOME
$ git clone https://github.com/kyasuda516/shu7-eitango.git
$ cd shu7-eitango
```

#### 2.3.2 make, jqをインストール

次のコマンドにより、make、jqをインストールします。
```bash
$ sudo apt update
$ sudo apt install make jq
```

#### 2.3.3 Docker Composeをインストール

次のコマンドで、スクリプト[`/scripts/setup/install_docker_compose.sh`](https://github.com/kyasuda516/shu7-eitango/blob/main/scripts/setup/install_docker_compose.sh)を利用して最新のDocker Composeをインストールします。
```bash
$ /bin/bash ./scripts/setup/install_docker_compose.sh
$ docker compose version  # インストール成功を確認
```
<!-- 
cd $HOME && git clone https://github.com/kyasuda516/shu7-eitango.git && cd shu7-eitango && sudo apt update && sudo apt install -y make jq && /bin/bash ./scripts/setup/install_docker_compose.sh
-->

※このスクリプトは以下のDocker公式ドキュメントのページを参考に作っています：
  - [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
  - [Install the Compose plugin](https://docs.docker.com/compose/install/linux/#install-the-plugin-manually)

#### 2.3.4 cronジョブの設定

`$ sudo vi /etc/crontab`のようなコマンドによって、`/etc/crontab`に次のテキストを追記してください（`yamada`の部分は冒頭で新規作成したユーザ名にしてください）。
```cron
2  0    * * *   yamada  /bin/bash ~/shu7-eitango/scripts/cron/rotate_nginx_logs.sh
30 3 1,15 * *   yamada  /bin/bash ~/shu7-eitango/scripts/cron/renew_cert.sh
2 14 * * 3,6    yamada  /bin/bash ~/shu7-eitango/scripts/cron/update_geoip.sh
50 5    * * *   yamada  /bin/bash ~/shu7-eitango/scripts/cron/retrieve_prons.sh
05 6    * * *   yamada  curl -L https://shu7-eitango.com/bunch > /dev/null 2>&1
```

#### 2.3.5 Grafanaによるメール送信先の設定

GrafanaのAlertingによるメールの送信先を環境変数として設定していきます。

まず、`$ vi ~/.bashrc`のようなコマンドによって、`~/.bashrc`ファイルに次のテキストを追記します（メールアドレスの部分は当該の送信先アドレスにしてください）。
```shell
export GRAFANA_ALERT_MAIL_TO=yamada@example.com
```
それから、次のコマンドを実行することで、`~/.bashrc`ファイルをリロードして環境変数を反映させます。
```bash
$ source ~/.bashrc
```

#### 2.3.6 その他のセットアップ

シークレットファイルの書き換えに先立ち、以下の説明に従って、必要となる各種キーを発行しておいてください。

<details>
<summary>GeoLite2のライセンスキーの発行</summary>

本アプリケーションでは、IP GeolocationデータベースとしてMaxMind社によるGeoLite2を利用しています。GeoLite2は同社によるGeoIP2データベースの廉価版といえるもので、精度は劣るものの無料で使えます。

このGeoLite2を利用するためのライセンスキーを発行していきます。

1. あらかじめ[MaxMindのホームページ](https://www.MaxMind.com)にサインインしておきます。

2. [ライセンスキーの管理ページ](https://www.MaxMind.com/en/accounts/current/license-key/)にて「Generate new license key」ボタンを押し、ライセンスキーを発行してください。

3. 発行したらそのキー文字列を控えておいてください（これが後述の`geoip_key.secret`の内容となります）。

4. また、同ページにて「Account ID: ～」で示されているアカウントIDも控えておいてください（これが後述の`geoip_id.secret`の内容となります）。

</details>

<details>
<summary>WordsAPIのAPIキーの発行</summary>

本アプリケーションでは英単語の発音記号を取得するためにRapidAPIのひとつであるWordsAPIを利用しています。

ここでは、RapidAPIを利用するためのAPIキーを取得し、WordsAPIの無料プランにサブスクライブしていきます。

1. あらかじめ[RapidAPI](https://rapidapi.com/hub)にサインインしておきます。

2. RapidAPIのページ上で以下の画像にある①～④の順にクリックすることでAPIキーの文字列をコピーしてください。

    ![RapidAPIダッシュボード](/portfolio/img/rapidapi_devdashboard.png)

3. コピーできたら、それを控えておいてください（後述の`wordsapi_key.secret`の内容になります）。

4. サインイン状態を保持したまま、[WordsAPIの価格説明ページ](https://rapidapi.com/dpventures/api/wordsapi/pricing)に移動します。

5. 表中のBasicプラン（制限付きの無料プラン）の「Subscribe」ボタンを押し、画面上の指示に沿ってサブスクライブの手続きを完了してください（クレジットカードの登録を求められることがありますが、その場合は登録をお願いします）。
</details>

<details>
<summary>Gmailアカウントのアプリパスワードの発行</summary>


本アプリケーションでは、Grafanaからの通知としてメールを送信する際にGmailのSMTPサーバを利用しています。そのために、Gmailアカウントのアプリパスワードが必要となります。

[こちらのGoogleアカウントヘルプ](https://support.google.com/accounts/answer/185833?hl=ja)に沿ってGoogleアカウントのアプリパスワードを発行してください。発行したらそのパスワード文字列を控えておいてください（これが後述の`grafana_mail_key.secret`の内容となります）。
</details>

各種キーの発行を終えたら、次のコマンドを実行して、シークレットファイルを作成してください。
```bash
$ /bin/bash ./scripts/setup/export_secrets.sh
```
次の表に示すファイル群が`./secrets/`に作成されているので、中身を適切な内容に書き換えてください。

| ファイル名 | 内容の説明 | 内容の例 |
| ---- | ---- | ---- |
| `cache_db_limits.secret`| `cache`サービスにおけるAPI使用制限監視用のデータベース(のID) | `0` |
| `cache_db_page.secret`| `cache`サービスにおけるページキャッシュ用のデータベース(のID) | `1` |
| `cache_db_pron.secret`| `cache`サービスにおける発音記号データのキャッシュ用のデータベース(のID) | `2` |
| `cache_password.secret`| `cache`サービスのredisパスワード |  |
| `db_database.secret`| `db`サービスにおけるデータベース名 | `dummydb1` |
| `db_password.secret`| `db`サービスにおけるMySQLユーザのパスワード |  |
| `db_root_password.secret`| `db`サービスにおけるMySQLルートユーザのパスワード |  |
| `db_user.secret`| `db`サービスにおけるMySQLユーザ名 |  |
| `geoip_id.secret`| [MaxMind](https://www.MaxMind.com/en/home)のアカウントID（[先ほど](#geolite2)取得） |  |
| `geoip_key.secret`| [MaxMind](https://www.MaxMind.com/en/home)で発行したライセンスキー（[先ほど](#geolite2)取得） |  |
| `grafana_admin_password.secret`| `grafana`サービスにおけるadminユーザのパスワード |  |
| `grafana_admin_user.secret`| `grafana`サービスにおけるadminユーザ名 |  |
| `grafana_mail_address.secret`| `grafana`サービスでSMTPサーバに用いるGmailのアドレス |  |
| `grafana_mail_key.secret`| 上記Gmailアカウントのアプリパスワード（[先ほど](#gmail)取得） |  |
| `wordsapi_key.secret`| [WordsAPI](https://rapidapi.com/dpventures/api/wordsapi/pricing)のAPIキー（[先ほど](#wordsapi)取得） |  |

それから次のコマンドにより、その他の必要なセットアップを完了してください。
```bash
$ /bin/bash ./scripts/setup/create_volumes.sh  # ボリュームを作成する
$ /bin/bash ./scripts/setup/get_geoip_db.sh    # GeoLite2データベースを取得します。
$ /bin/bash ./scripts/setup/get_cert.sh        # SSL証明書を取得します。
```

### 2.4 テストと検証

次のコマンドを実行して、テストを行ってください。
```bash
$ make test.stg
```

テストが済んだら、パフォーマンスを確認するために[GrafanaのDashborad 1](https://shu7-eitango.com/admin/grafana/d/dashboard1/dashboard-1)を閲覧してください。

## 3. 運用
### 3.1 運用の開始

システムの運用を開始するには、次のコマンドを実行してください。
```bash
$ make build.prod
```

### 3.2 システムの停止

システムを停止するには、次のコマンドを実行してください。
```bash
$ make clean
```

### 3.3 アップデート

システムをアップデートするには、次のコマンドを実行してください。
```bash
$ git pull
$ make build.prod
```
