# Lightsail + Django の構築手順
LightsailはAWSにおけるEC2+RDS+ElasticIP+S3+Route53などの一連のサービスをセットにして
使いやすくしたもの。
AWSにおけるDjangoの構成イメージ 
https://docs.google.com/presentation/d/1csa31FCjwntZH6QVCV5a7PAXEOiBqVy-AQYtZhd3eqI/edit#slide=id.gd66254ea13_0_0

## 必要な前提知識
Pythonを使った開発、Djangoのプロジェクト作成や開発サーバーを使った開発
コンソール操作、Git操作はできることが、以降の作業を行う上での最低限の前提となります。
また、IT知識は、ITパスポート取得レベル程度を想定しているため、一般的なIT用語説明は省略しています。

## 用語説明
インスタンス：サーバー(EC2に相当)
データベース: RDS(AWSにおけるリレーショナル・データベース)に相当
nginx： Webサーバー。フロントエンドのやり取りを行う
gunicorn: アプリケーションサーバー(Djangoサーバー)。バックエンドのやり取りを行う（開発サーバーの本番用）
静的IP:AWSは既定では動的なIPが付与されており、再起動等でIPが変更されるため、Webサーバーの場合は、静的IPを付与するのが一般的。

## AWS料金
インスタンス：5ドル～(実用的なインスタンス)
DB:15ドル～
→　計20ドル/月で実用的な環境が構築可能

## 手順
### インスタンスの作成
インスタンスタブから新規作成を実施。
プラットフォーム:Linux/Unix
設計図の選択:OSのみ→AmazonLinux2
インスタンスプラインの選択:好きなものを選択（5ドルを推奨）
任意の名前をつけてインスタンスを作成

### データベースの作成
データベースタブからデータベースを作成。
MysQLを選択。
好きな料金プランを選択。
任意の名称をつけてデータベースを作成。

### データベースの公開設定
エンドポイントとポートの欄を参照し<BR>
パブリックアクセスが無効になっているので、有効化する。<BR>
エンドポイントは、Djangoに設定する必要があるので控えておく。<BR>
エンドポイントの例： *******.us-east-1.rds.amazonaws.com
 
### Djangoのデータベース接続設定
settings.pyのデータベース接続設定を変更する。
```
        'NAME': <DB名>,　#既定ではdbmaster
        'USER': <DBユーザー名>,  #既定ではdbmasteruser
        'PASSWORD': <DBパスワード>,  #既定ではランダムに生成されたパスワード
        'HOST': <エンドポイント>, # 例：******.us-east-1.rds.amazonaws.com
```

### ネットワーキングの設定
作成されたインスタンスを選択、ネットワーキングタブ
パブリックIP→静的IPの作成からIPを設定。
IPv4ファイアーウォール→ルールを追加からアプリケーション：HTTPSを選択して作成。
接続タブからSSHを使用して接続をクリックしLinuxに接続できることを確認する。
※SSHクライアントを使用したい場合は、以下からデフォルトキーをダウンロードして使用する。
https://lightsail.aws.amazon.com/ls/webapp/account/keys

### Linuxの設定
LinuxにSSH接続して以下の作業を実施する。
#### パッケージ更新
`sudo yum update -y`

#### タイムゾーン変更
`sudo timedatectl set-timezone Asia/Tokyo`

#### 日本語のローケルに変更
`sudo localectl set-locale LANG=ja_JP.UTF-8`

#### 再起動で時刻を反映(後で行っても良い)
`sudo reboot`

#### 操作用のアカウント作成
デフォルトのEC2を使用するとセキュリティ上よろしくないため、任意のユーザーアカウントを作成する。<BR>
以降では、アカウント名を「web_admin」とした場合の例を記述しますので、それ以外の名称にしたい場合は<BR>
コマンドのweb_adminの箇所を全て置き換えてください。<BR>
※ただし、ec2-userをログイン不可にすると、LightsailのSSH画面から接続できなくなるため<BR>
LightsailのSSH画面から接続したい場合（デスクトップからSSHで接続するのが難しい場合など）は<BR>
既定のec2-userのまま作業を進めてください。<BR>
その場合は「操作用のアカウント作成」の作業は全てスキップしてください<BR>
(その場合は以降はweb_adminの箇所をec2-userに読み替えてください)

```
sudo useradd web_admin
sudo cp -arp /home/ec2-user/.ssh /home/web_admin
sudo chown -R web_admin /home/web_admin/.ssh
sudo visudo -f /etc/sudoers.d/90-cloud-init-users
```
viが起動するので、 ec2-user を# でコメントアウト<BR>
web_admin ALL=(ALL) NOPASSWD:ALL を追加<BR>
※viは、i入力で編集モードになり、:wq入力で上書き保存する<BR>

SSHからログオフして再度web_adminでログイン<BR>
ec2-userを削除
```
sudo userdel -r ec2-user
```
 
#### gitのインストール
```
sudo yum install git
```

#### リポジトリのクローン
別途作成したDjangoのリポジトリをcloneする。
git clone <url>

#### プロジェクトフォルダに移動
cd <フォルダ>

#### venvを作成してactivate
```
python3 -m venv venv 
. venv/bin/activate
```

#### Mysqlを使用するための前提モジュールをインストール
```
sudo yum install python-devel mysql-devel
sudo yum -y install gcc
sudo yum install python3-devel
```

#### python関連パッケージインストール
```
pip install -r requirements.txt
```

#### cronのインストール(スケジューラ機能)
```
sudo yum install -y yum-cron
sudo amazon-linux-extras install epel
```

#### cronの起動設定
```
sudo systemctl start yum-cron
sudo systemctl enable yum-cron
```

#### Selenium使用の場合(サーバー上でSeleniumを使用しない場合は不要)
Chromeをインストールする
```
curl https://intoli.com/install-google-chrome.sh | bash
sudo yum -y install GConf2
```
バージョン確認して表示されれば成功
```
google-chrome-stable -version
```

#### gunicornのインストール
```
pip install gunicorn
```

#### nginxインストール
```
sudo amazon-linux-extras install nginx1
```
#### nginx自動起動設定
```
sudo systemctl enable nginx.service
```
#### nginxの静的ファイルの設定
```
sudo mkdir -p /usr/share/nginx/html/static
sudo mkdir -p /usr/share/nginx/html/media

sudo chown -R web_admin /usr/share/nginx/html/static
sudo chown -R web_admin /usr/share/nginx/html/media

(Djangoプロジェクトルートにて)
python manage.py collectstatic
```

#### nginx.confの編集
以下を実行してファイルを開く
```
sudo vi /etc/nginx/nginx.conf
```

以下を変更
httpセクション内
```
　server_name：静的IPアドレスを入力
```

以下を追記
httpセクション内
```
 　location /static {
       alias /usr/share/nginx/html/static;
   }

   location /media {
       alias /usr/share/nginx/html/media;
   }

   location / {
       proxy_set_header Host $http_host;
       proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       proxy_set_header X-Forwarded-Proto $scheme;

       proxy_pass http://127.0.0.1:8000;
   }
  ```

#### nginxのリロード
```
sudo nginx -s reload
```

#### gunicron起動
```
gunicorn --bind 127.0.0.1:8000 <プロジェクト名.wsgi> -D
```

#### Webブラウザからアクセスを確認
http://<静的IPアドレス> でアクセスできることを確認。
できない場合は、Djangoのsettings.pyファイルのALLOWS_HOSTに
静的IPアドレスが記載されているか確認。

#### 独自ドメイン・SSL化
SSL化のために事前にドメインを取得して、DNSゾーンの設定にてドメインとインスタンスの紐付けを実施し
名前解決ができるようにしておくこと。
```
git clone https://github.com/certbot/certbot
sudo yum -y install python-virtualenv
sudo wget -r --no-parent -A 'epel-release-*.rpm'  http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/
sudo rpm -Uvh dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-*.rpm
sudo yum install certbot python2-certbot-nginx
sudo certbot --nginx -d　<ドメイン名>
```

※SSL化については、この後にnginx.confファイルの編集が必要だが、煩雑なため別途記載する。
