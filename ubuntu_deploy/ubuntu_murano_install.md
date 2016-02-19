# Murano 安裝與設定
本章節會說明與操作如何安裝```Application Catalog```服務到 OpenStack Controller 節點上，並設置相關參數與設定。若對於 Murano 不瞭解的人，可以參考[Murano 應用程式目錄套件](http://murano.readthedocs.org/en/stable-kilo/install/manual.html)

### 安裝前準備
我們需要在 Database 底下建立儲存 Murano 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Murano 資料庫與使用者：
```sql
CREATE DATABASE murano;
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'localhost'  IDENTIFIED BY ' MURANO_DBPASS';
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'%'  IDENTIFIED BY 'MURANO_DBPASS';

```
> 這邊若```MURANO_DBPASS```要更改的話，可以更改。

接著要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Magnum User
openstack user create --password MURANO_PASS --email murano@example.com murano

# 建立 Magnum Role
openstack role add --project service --user murano admin

# 建立 Magnum service
openstack service create --name murano  --description "Application Catalog" application_catalog

# 建立 Magnum cloudfoundry broker service
openstack service create --name murano-cfapi  --description "Murano CloudFoundry Service Broker" service-broker

# 建立 Magnum URL
openstack endpoint create --publicurl http://163.17.136.246:8082 \
--internalurl http://10.0.0.11:8082 \
--adminurl http://10.0.0.11:8082 \
--region RegionOne application_catalog

# 建立 Magnum URL
openstack endpoint create --publicurl http://163.17.136.246:8083 \
--internalurl http://10.0.0.11:8083 \
--adminurl http://10.0.0.11:8083 \
--region RegionOne service-broker
```
> 這邊若```MURANO_PASS```要更改的話，可以更改。

建立 murano 使用者與相關檔案放置用目錄：
```sh
for SERVICE in murano
do
useradd --home-dir "/var/lib/$SERVICE" \
    --create-home \
    --system \
    --shell /bin/false \
    $SERVICE

#Create essential dirs

mkdir -p /var/log/$SERVICE
mkdir -p /etc/$SERVICE

#Set ownership of the dirs

chown -R $SERVICE:$SERVICE /var/log/$SERVICE
chown -R $SERVICE:$SERVICE /var/lib/$SERVICE
chown $SERVICE:$SERVICE /etc/$SERVICE
done
```


完成建立後，安裝 Python 相關套件：
```sh
sudo apt-get install python-pip python-dev \
libmysqlclient-dev libpq-dev \
libxml2-dev libxslt1-dev \
libffi-dev
```
並安裝虛擬環境管理套件```tox```：
```sh
sudo pip install tox
```

### 安裝 Murano 套件
這邊採用 git 的 repository 安裝：
```sh
 git clone git://git.openstack.org/openstack/murano
```
> 若使用 .deb 安裝的話，可以執行以下指令：
> 
```
sudo apt-get install murano-api murano-engine python-muranoclient
```

設置 murano 的設定檔案，可以透過```tox```產生板模檔案：
```sh
tox -e genconfig
```
建立設定檔目錄```/etc/murano```，並將設定檔複製到目錄：
```sh
sudo mkdir -p /etc/murano/
sudo cp etc/murano/murano-paste.ini /etc/murano/
sudo cp etc/murano/policy.json /etc/murano/
sudo cp etc/murano/murano.conf.sample /etc/murano/murano.conf
```
設定 murano 配置檔```/etc/murano/murano.conf```，在```[DEFAULT]```加入以下：
```sh
[DEFAULT]
notification_driver = messagingv2
use_syslog = False
debug = True
```
在```[database]```加入資料庫連接：
```sh
[database]
connection = mysql+pymysql://murano:MURANO_DBPASS@10.0.0.11/murano
```
在```[engine]```加入以下：
```sh
[engine]
enable_model_policy_enforcer = False
```
在```[keystone]```加入驗證 URL：
```sh
[keystone]
auth_url = http://10.0.0.11:5000/v2.0
```
在```[keystone_authtoken]```加入完整 Keystone 驗證：
```sh
[keystone_authtoken]
admin_password = MURANO_PASS
admin_user = murano
admin_tenant_name = service
cafile = 
auth_protocol = http
auth_port = 35357
auth_host = 10.0.0.11
auth_uri = http://10.0.0.11:5000/v2.0
```
在```[murano]```加入 bind ip 與 port：
```sh
[murano]
url = http://127.0.0.1:8082
```
在```[networking]```加入是否要自動建立 Router：
```sh
[networking]
create_router = true
external_network = "4a34fcea-1921-4bc5-a6ab-9af49f853357"
```
在```[oslo_messaging_rabbit]```加入 oslo  AMQP：
```sh
[oslo_messaging_rabbit]
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
rabbit_hosts = 10.0.0.11
```
在```[rabbitmq]```加入 AMQP：
```sh
[rabbitmq]
password = RABBIT_PASS
login = openstack
host = 10.0.0.11
```

完成後，透過以下指令初始化與更新資料庫：
```sh
murano-db-manage --config-file /etc/murano/murano.conf upgrade
```

啟動服務 ```murano-api```：
```sh
murano-api --config-file /etc/murano/murano.conf
```

上傳核心套件檔案：
```sh
murano-manage --config-file /etc/murano/murano.conf import-package ./meta/io.murano
```

啟動服務 ```murano-engine```：
```sh
murano-engine --config-file /etc/murano/murano.conf
```
> 若要在背景模式，可以建立以下兩個 upstart 檔案：

#### murano-api
```sh
cat > /etc/init/murano-api.conf << EOF
description "OpenStack Murano API"
author "kyle Bai <kyle.b@inwinStack.com>"

start on runlevel [2345]
stop on runlevel [!2345]

exec start-stop-daemon --start --chuid murano --exec /usr/local/bin/murano-api -- --config-file=/etc/murano/murano.conf
EOF
```
#### murano-engine
```sh
cat > /etc/init/murano-engine.conf << EOF
description "OpenStack Murano Engine"
author "kyle Bai <kyle.b@inwinStack.com>"

start on runlevel [2345]
stop on runlevel [!2345]

exec start-stop-daemon --start --chuid murano --exec /usr/local/bin/murano-engine -- --config-file=/etc/murano/murano.conf
EOF
```
完成後，設定開機啟動：
```sh
sudo update-rc.d murano-api defaults
sudo update-rc.d murano-engine defaults
```

啟動服務：
```sh
sudo start murano-api
sudo start murano-engine
```







