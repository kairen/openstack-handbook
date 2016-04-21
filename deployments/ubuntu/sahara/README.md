# Sahara 安裝與設定
本章節會說明與操作如何安裝資料處理服務到 Controller 節點上，並設置相關參數與設定。若對於 Sahara 不瞭解的人，可以參考[Sahara 資料處理服務](../../../conceptions/sahara/README.md)

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)
    - [建立 Sahara 服務映像檔](#建立-sahara-服務映像檔)
    - [建立 Sahara 叢集](#建立-sahara-叢集)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Sahara 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Sahara 資料庫：
```sql
CREATE DATABASE sahara;
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'localhost'  IDENTIFIED BY ' SAHARA_DBPASS';
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'%'  IDENTIFIED BY 'SAHARA_DBPASS';
```
> 這邊```SAHARA_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Sahara 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Sahara User
$ openstack user create --domain default \
--password SAHARA_PASS --email sahara@example.com sahara

# 建立 Sahara Role
$ openstack role add --project service --user sahara admin

# 建立 Sahara Service
$ openstack service create --name sahara \
--description "Data processing service" data_processing

# 建立 Sahara public endpoints
$ openstack endpoint create --region RegionOne \
data_processing public http://10.0.0.11:8386/v1.1/%\(tenant_id\)s

# 建立 Sahara internal endpoints
$ openstack endpoint create --region RegionOne \
data_processing internal http://10.0.0.11:8386/v1.1/%\(tenant_id\)s

# 建立 Sahara admin endpoints
$ openstack endpoint create --region RegionOne \
data_processing admin http://10.0.0.11:8386/v1.1/%\(tenant_id\)s
```
> 這邊```SAHARA_PASS```可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install sahara-api sahara-common sahara-engine python-sahara
```

安裝完成後，編輯```/etc/sahara/sahara.conf```，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
use_syslog = False
infrastructure_engine = heat
use_neutron = true
use_floating_ip = true
plugins = vanilla,hdp,cdh,spark,fake
debug = True
verbose = True
notification_driver = messaging
enable_notifications = true
rpc_backend = rabbit
```

接下來，在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql://sahara:SAHARA_DBPASS@controller/sahara
```
> 這邊```SAHARA_DBPASS```可以隨需求修改。

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
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = sahara
password = SAHARA_PASS
```
> 這邊```SAHARA_PASS```可以隨需求修改。

在```[oslo_policy]```部分加入以下內容：
```
[oslo_policy]
policy_file = /etc/sahara/policy.json
```

完成所有設定後，即可同步資料庫來建立 Magnum 資料表：
```sh
$ sudo sahara-db-manage upgrade head
```

重新啟動所有 Sahara 服務：
```sh
sudo service sahara-api restart
sudo service sahara-engine restart
```

# 驗證服務
首先回到 ```Controller``` 節點並接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

透過 Sahara client 來查看服務列表，如以下方式：
```sh
$ sahara cluster-list
+------+----+--------+------------+
| name | id | status | node_count |
+------+----+--------+------------+
+------+----+--------+------------+
```

### 建立 Sahara 服務映像檔
建立一個簡單叢集，以 ```CDH 5.3.0``` 、作業系統 ```Ubuntu```為範例，首先下載```sahara-image-elements```套件：
```sh
sudo pip install --upgrade sahara-image-elements
```

下載 ```sahara-image-elements```相關 Shell Script 的 Git資源庫：
```sh
git clone https://github.com/openstack/sahara-image-elements
cd sahara-image-elements
```

安裝 ```tox``` 套件來建置Images：
```sh
sudo pip install --upgrade  tox
```

透過 ```tox``` 建立一個 Image ：
```sh
sudo tox -e venv -- sahara-image-create -p cloudera -i ubuntu -v 5.3
```

> ```-p``` 為插件種類，有以下：
[vanilla|spark|hdp|cloudera|storm|mapr|plain]

> ```-i``` 為作業系統，有以下：
[ubuntu|fedora|centos|centos7]

> ```-v``` 為版本號，如以下範例：
[1|2|2.6|4|5.0|5.3|5.4]

> 更多的參數可以到 [sahara-image-elements](https://github.com/openstack/sahara-image-elements/blob/master/diskimage-create/README.rst) 查看。

> 建置Cloudera Plugin 可以參考 [cdh-on-openstack-with-sahara](http://blog.cloudera.com/blog/2015/05/how-to-get-started-with-cdh-on-openstack-with-sahara/)。

### 建立 Sahara 叢集
