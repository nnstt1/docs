# PowerDNS

OSS の PowerDNS を自宅ラボ用 DNS サーバとして構築します。

https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_openstack_platform/7/html/dns-as-a-service_guide/install_and_configure_powerdns


## PostgreSQL インストール

```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf module disable postgresql
sudo dnf install -y postgresql13-server

sudo /usr/pgsql-13/bin/postgresql-13-setup initdb
sudo systemctl enable --now postgresql-13

# postgres ユーザのパスワード設定
sudo passwd postgres
su - postgres
postgres=\# ALTER ROLE postgres WITH PASSWORD 'postgres';
postgres=\# \\q
exit

# ログイン方法変更
sudo vim /var/lib/pgsql/13/data/pg_hba.conf
sudo systemctl restart postgresql-13

# postgres ユーザで PowerDNS 用データベース作成
psql -U postgres
postgres=\# CREATE ROLE pdns WITH LOGIN PASSWORD 'pdns';
postgres=\# CREATE DATABASE pdns OWNER 'pdns';
postgres=\# \\q

# PowerDNS 用スキーマ作成
wget https://github.com/PowerDNS/pdns/blob/auth-4.5.1/modules/gpgsqlbackend/schema.pgsql.sql
psql -U pdns -d pdns -a -f schema.pgsql.sql 
```


## PowerDNS インストール

```bash
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install -y pdns pdns-backend-postgresql pdns-tools
```

2021/8/14 時点でインストールされたバージョンは 4.5.1 でした。

```console
=================================================================================================================================
 パッケージ                      アーキテクチャー           バージョン                  リポジトリー                      サイズ
=================================================================================================================================
インストール:
 pdns                            x86_64                    4.5.1-1.el8                 epel                             3.6 M
 pdns-backend-postgresql         x86_64                    4.5.1-1.el8                 epel                             67 k
 pdns-tools                      x86_64                    45.1-1.el8                  epel                             2.7 M
 ```


## PowerDNS 設定

```ini
# /etc/pdns/pdns.conf
api=yes
api-key=7fe343a8-c445-ed83-f815-81a369883984
launch=gpgsql
setgid=pdns
setuid=pdns
webserver=yes
webserver-address=0.0.0.0
webserver-allow-from=0.0.0.0/0,::1
gpgsql-host=/run/postgresql
gpgsql-dbname=pdns
gpgsql-user=pdns
gpgsql-password=pdns
```

## PowerDNS-Admin

PowerDNS 用 WebUI ツール [PowerDNS-Admin](https://github.com/ngoduykhanh/PowerDNS-Admin) をデプロイします。

### コンテナ

コンテナを使って PowerDNS-Admin をデプロイする方法です。

```bash
podman run -d \
    --name pdns_admin \
    --restart always \
    -e SQLALCHEMY_DATABASE_URI='postgresql://powerdnsadmin:powerdnsadmin@192.168.2.3/powerdnsadmindb' \
    -e SECRET_KEY='a-very-secret-key' \
    -v pda-data:/data \
    -p 9191:80 \
    ngoduykhanh/powerdns-admin:latest
```


### 事前準備

コンテナを使わずに PowerDNS の Web UI として PowerDNS-Admin をインストールする方法です。

以下の Wiki を参照します。

https://github.com/ngoduykhanh/PowerDNS-Admin/wiki


```bash
sudo dnf install -y postgresql-libs python3 python3-devel postgresql13-devel git gcc openldap-devel libxml2-devel python3-xmlsec
export PATH=$PATH:/usr/pgsql-13/bin/
pip3 install psycopg2 --user
pip3 install virtualenv --user
```

PostgreSQL に PowerDNS-Admin 用のユーザとデータベースを作成します。

```bash
su - postgres
createuser powerdnsadmin    # postgres ユーザのパスワードを入力
createdb powerdnsadmindb
psql
postgres=\# ALTER USER powerdnsadmin WITH ENCRYPTED PASSWORD 'powerdnsadmin';
postgres=\# GRANT ALL PRIVILEGES ON DATABASE powerdnsadmindb TO powerdnsadmin;
postgres=\# \\q
exit
```

後述の pip で xmlsec をビルドする際に必要な `xmlsec1-devel` はデフォルトのリポジトリでは提供されていません。
そこで、サポートされていない追加のコンテンツが新しい `CodeReady Linux Builder` を有効化します。
`CodeReady Linux Builder` リポジトリには `xmlsec1-devel` が提供されています。

https://access.redhat.com/ja/articles/5304171

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-x86_64-rpms
sudo yum install -y xmlsec1-devel
```

### yarn + Nodejs14

Wiki に記載されている Nodejs は 10 ですが、バージョンが古いため 14 をインストールします。

```bash
curl -sL https://rpm.nodesource.com/setup_14.x | sudo bash -
sudo curl -sL https://dl.yarnpkg.com/rpm/yarn.repo -o /etc/yum.repos.d/yarn.repo
sudo dnf install -y yarn
```

### checkout

```bash
sudo git clone https://github.com/ngoduykhanh/PowerDNS-Admin.git /opt/web/powerdns-admin
sudo chmod -R 777 /opt/web/powerdns-admin
cd /opt/web/powerdns-admin
virtualenv -p python3 flask
. ./flask/bin/activate
(flask) pip install python-dotenv

# requirements.txt から mysqlclient を削除しておく
(flask) pip install --upgrade pip setuptools wheel
(flask) pip install -r requirements.txt
```

### Running

```bash
(flask) export FLASK_APP=powerdnsadmin/__init__.py
(flask) flask db upgrade
```

`flask db upgrade` でエラーが発生したが、このタイミングでコンテナを使う方法があると判明し、中断しました。