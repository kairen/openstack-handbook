# Sahara 安裝與設定
本章節會說明與操作如何安裝```Data processing```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於 Sahara 不瞭解的人，可以參考[Sahara 資料處理套件](sahara.html)

# Controller節點安裝與設置
### 安裝前準備
我們需要在Database底下建立儲存 Sahara 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Sahara 資料庫與使用者：
```sql
CREATE DATABASE sahara;
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'localhost'  IDENTIFIED BY ' SAHARA_DBPASS';
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'%'  IDENTIFIED BY 'SAHARA_DBPASS';

```
> 這邊若```SAHARA_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立Sahara User
openstack user create --password SAHARA_PASS --email sahara@example.com sahara
# 建立Sahara Role
openstack role add --project service --user sahara admin
# 建立Sahara Service
openstack service create --name sahara  --description "Data processing service" data_processing
# 建立Sahara Endpoint
openstack endpoint create \
  --publicurl http://controller:8386/v1.1/%\(tenant_id\)s \
  --internalurl http://controller:8386/v1.1/%\(tenant_id\)s \
  --adminurl http://controller:8386/v1.1/%\(tenant_id\)s \
  --region RegionOne \
  data_processing
```
> 這邊若```SAHARA_PASS```要更改的話，可以更改。

### 安裝與設定 Sahara
我們要將 Sahara 安裝於 ```Controller``` 節點上，來提供資料處理服務，透過```apt-get```與```pip```安裝最新版本套件：
```sh
sudo apt-get install -y python-setuptools python-virtualenv python-dev
sudo apt-get install -y sahara
# sudo pip install http://tarballs.openstack.org/sahara/sahara-stable-kilo.tar.gz
# sudo pip install --upgrade sahara
```
> 其他版本套件可以到這邊下載 [OpenStack Sahara](http://tarballs.openstack.org/sahara/)。

安裝完成後，編輯 ```/etc/sahara/sahara.conf```在 ```[database]``` 加入以下：
```
[database]
connection = mysql://sahara:SAHARA_DBPASS@controller/sahara
```
在```[DEFAULT]```部分，若使用 Neutron 則將 ```use_neutron```設為```true```，否則為```false```：
```
[DEFAULT]
logging_exception_prefix = %(color)s%(asctime)s.%(msecs)03d TRACE %(name)s %(instance)s
logging_debug_format_suffix = from (pid=%(process)d) %(funcName)s %(pathname)s:%(lineno)d
logging_default_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [-%(color)s] %(instance)s%(color)s%(message)s
logging_context_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [%(request_id)s %(user_name)s %(project_name)s%(color)s] %(instance)s%(color)s%(message)s
use_syslog = False
infrastructure_engine = heat
use_neutron = true
use_floating_ip=False
plugins = vanilla,hdp,cdh,spark,fake
debug = True
verbose = True
notification_driver = messaging
enable_notifications = true
rpc_backend = rabbit
```
在```[keystone_authtoken]```部分設定Keystone的相關驗證參數：
```
[keystone_authtoken]
signing_dir = /var/cache/sahara
cafile = /opt/stack/data/ca-bundle.pem
auth_uri = http://controller:5000
project_domain_id = default
project_name = service
user_domain_id = default
password = SAHARA_PASS
username = sahara
auth_url = http://controller:35357
auth_plugin = password
```
在 ```[oslo_messaging_rabbit]``` 設定 RabbitMQ 資訊：
```sh
[oslo_messaging_rabbit]
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
rabbit_hosts = controller
```
在 ```[oslo_policy]```部分設定存取權限：
```
[oslo_policy]
policy_file = /etc/sahara/policy.json
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
建立一個 ```policy.json``` 檔案，並新增以下：
```
{
    "default": ""
}
```
如果使用 MySQL 或 MariaDB 做為 Data process服務的資料庫的話，需編輯```/etc/mysql/my.cnf ```設定maximum number允許的Packets：
```
[mysqld]
max_allowed_packet = 256M
```
完成後重啟 MySQL 服務：
```sh
sudo service mysql restart
```
完成設定後同步資料庫：
```sh
sudo sahara-db-manage --config-file /etc/sahara/sahara.conf upgrade head
```
重啟服務：
```sh
sudo service sahara restart
```

# 驗證操作
這部分我們將驗證 Data processing 是否正確運作。首先導入```admin```環境變數檔案：
```
source admin-openrc.sh
```
透過 ```sahara``` 指令來列出叢集資訊：
```sh
sahara cluster-list
```
成功的話會顯示以下資訊：
```sh
+------+----+--------+------------+
| name | id | status | node_count |
+------+----+--------+------------+
+------+----+--------+------------+
```
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

# 建立叢集




http://docs.openstack.org/developer/sahara/userdoc/configuration.guide.html
