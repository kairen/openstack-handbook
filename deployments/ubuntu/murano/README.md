# Murano 安裝與設定
本章節會說明與操作如何安裝應用程式目錄服務到 Controller 節點上，並設置相關參數與設定。若對於 Murano 不瞭解的人，可以參考[Murano 應用程式目錄服務](../../../conceptions/murano/README.md)

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Murano 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Murano 資料庫：
```sql
CREATE DATABASE murano;
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'localhost'  IDENTIFIED BY ' MURANO_DBPASS';
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'%'  IDENTIFIED BY 'MURANO_DBPASS';
```
> 這邊```MURANO_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Murano 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Murano User
$ openstack user create --domain default --password MURANO_PASS --email murano@example.com murano

# 建立 Murano Role
$ openstack role add --project service --user murano admin

# 建立 Murano service
$ openstack service create --name murano \
--description "Application Catalog" application_catalog

# 建立 Murano public endpoints
$ openstack endpoint create --region RegionOne \
application_catalog public http://10.0.0.11:8082

# 建立 Murano internal endpoints
$ openstack endpoint create --region RegionOne \
application_catalog internal http://10.0.0.11:8082

# 建立 Murano admin endpoints
$ openstack endpoint create --region RegionOne \
application_catalog admin http://10.0.0.11:8082
```
> 這邊```MURANO_PASS```可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install murano-api murano-engine python-muranoclient
```

安裝完成後，編輯 ```/etc/murano/murano.conf``` 設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
home_region = RegionOne
use_syslog = False
debug = True
```

接下來，在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://murano:MURANO_DBPASS@10.0.0.11/murano
```
> 這邊```MURANO_DBPASS```可以隨需求修改。

在```[oslo_messaging_rabbit]```部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下內容：
```
[keystone_authtoken]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_name = default
project_name = service
user_domain_name = default
admin_tenant_name = default
auth_type = password
admin_user = murano
admin_password = MURANO_PASS
username = murano
password = MURANO_PASS
```
> 這邊```MURANO_PASS```可以隨需求修改。

在```[engine]```部分加入以下內容：
```
[engine]
enable_model_policy_enforcer = False
```

在```[murano]```部分加入以下內容：
```
[murano]
url = http://127.0.0.1:8082
```

在```[networking]```部分加入以下內容：
```
[networking]
create_router = true
external_network = EXTERNAL_NETWORK
```
> 這邊```EXTERNAL_NETWORK```請取代成環境的 Provider 網路。

在```[oslo_messaging_notifications]```部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messagingv2
```

在```[rabbitmq]```部分加入以下內容：
```
[rabbitmq]
host = 10.0.0.11
login = openstack
password = RABBIT_PASS
```

在```[keystone]```部分加入以下內容：
```
[keystone]
auth_url = http://10.0.0.11:5000
```

完成所有設定後，即可同步資料庫來建立 Murano 資料表：
```
$ sudo murano-db-manage upgrade
```

重新啟動所有 Murano 服務：
```sh
sudo service murano-api restart
sudo service murano-engine restart
```

# 驗證服務
首先回到 ```Controller``` 節點並接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

下載將使用的 Murano meta core 套件，並上傳核心套件：
```sh
$ git clone https://github.com/openstack/murano.git
$ cd murano
$ sudo murano-manage import-package ./meta/io.murano
```

透過 Murano client 來查看服務列表，如以下方式：
```sh
$ murano package-list
```
