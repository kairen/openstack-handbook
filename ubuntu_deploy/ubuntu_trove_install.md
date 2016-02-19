# Trove 安裝與設定
本章節會說明與操作如何安裝```Database```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於 Trove 不瞭解的人，可以參考[Trove 資料庫服務套件](http://kairen.gitbooks.io/openstack/content/trove/index.html)

# Controller節點安裝與設置
### 安裝前準備
我們需要在Database底下建立儲存 Trove 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Trove 資料庫與使用者：
```sql
CREATE DATABASE trove;
GRANT ALL PRIVILEGES ON trove.* TO trove@'localhost' IDENTIFIED BY 'TROVE_DBPASS';
GRANT ALL PRIVILEGES ON trove.* TO trove@'%' IDENTIFIED BY 'TROVE_DBPASS';
```
> 這邊若```TROVE_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Trove User
openstack user create --password TROVE_PASS --email trove@example.com trove

# 建立 Trove Role
openstack role add --project service --user trove admin

# 建立 Trove Service
openstack service create --name trove  --description "OpenStack Database service" database

# 建立 Trove Endpoint
openstack endpoint create --publicurl http://10.0.0.11:8779/v1.0/%\(tenant_id\)s \
--internalurl http://10.0.0.11:8779/v1.0/%\(tenant_id\)s \
--adminurl http://10.0.0.11:8779/v1.0/%\(tenant_id\)s \
--region RegionOne database
```
> 這邊若 ```TROVE_PASS``` 要更改的話，可以更改。

### 安裝與設定 Trove
我們要將 Trove 安裝於 ```Controller``` 節點上，來提供資料庫服務，透過```apt-get```安裝最新版本套件：
```sh
sudo apt-get install python-trove python-troveclient trove-common trove-api trove-taskmanager trove-conductor
```
完成安裝後，我們將設定 ```/etc/trove``` 目錄底下的所有檔案包含```api-paste.ini```、 ```trove.conf```、```trove-taskmanager.conf```、```trove-conductor.conf```，首先編輯```api-paste.ini```並修改以下：
```
[composite:trove]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = trove
password = TROVE_PASS
```
> 這邊若 ```TROVE_PASS``` 要更改的話，可以更改。

編輯除了```api-paste.ini```以外所有檔案中的```[DEFAULT]```部分，來設定服務的URL、Logging、訊息配置以及 SQL 連接：
```
[DEFAULT]
log_dir = /var/log/trove
trove_auth_url = http://controller:5000/v2.0
nova_compute_url = http://controller:8774/v2
cinder_url = http://controller:8776/v2
swift_url = http://controller:8080/v1/AUTH_

notifier_queue_hostname = controller
```
在```[database]```部分加入MySQL設定，並註解掉不必要的資訊：
```sh
[database]
connection = mysql://trove:TROVE_DBPASS@controller/trove
```
在```[DEFAULT]```部分，來設定每個設定檔使用RabbitMQ proxy：
```
[DEFAULT]
control_exchange = trove
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
rabbit_virtual_host= /
rpc_backend = trove.openstack.common.rpc.impl_kombu
```
> 這邊若 ```RABBIT_PASS``` 要更改的話，可以更改。

編輯```trove.conf```檔案，在```[DEFAULT]```部分設定資料儲存與網路的label regex：
```
[DEFAULT]
add_addresses = True
network_label_regex = ^NETWORK_LABEL$
control_exchange = trove
```
編輯```trove-taskmanager.conf```在```[DEFAULT]```部分設定連接到Compute服務：
```
[DEFAULT]
nova_proxy_admin_user = admin
nova_proxy_admin_pass = ADMIN_PASS
nova_proxy_admin_tenant_name = service
taskmanager_manager = trove.taskmanager.manager.Manager
log_file = trove-taskmanager.log
```
完成後同步資料庫：
```sh
sudo trove-manage db_sync
```
建立一個資料庫，可以選擇 MySQL、MongoDB、Cassandra 等，以下為一個 MySQL database 範例：
```sh
sudo trove-manage datastore_update mysql ''
```
編輯```trove-guestagent.conf```檔案，來設定提供Agent的連接：
```
rabbit_host = controller
rabbit_password = RABBIT_PASS
nova_proxy_admin_user = admin
nova_proxy_admin_pass = ADMIN_PASS
nova_proxy_admin_tenant_name = service
trove_auth_url = http://controller:35357/v2.0
log_file = trove-guestagent.log
```
更新資料庫與版本，可以使用以下指令：
```sh
trove-manage datastore_update [datastore_name] [datastore_version]

trove-manage datastore_version_update [datastore_name] [version_name]  [datastore_manager] [glance_image_id] [packages active]
```
一個建立 MySQL 5.5的範例：
```sh
trove-manage datastore_update mysql ''
trove-manage datastore_version_update mysql 5.5 mysql glance_image_ID mysql-server-5.5 1
trove-manage datastore_update mysql 5.5
```
上傳設定的驗證規則，可以使用以下指令：
```sh
trove-manage db_load_datastore_config_parameters [datastore_name] [version_name] [rules file]
```
一個 MySQL Database 的規則範例：
```sh
sudo cp /usr/lib/python2.7/dist-packages/trove/templates/mysql/validation-rules.json ~/validation-rules.json

sudo trove-manage db_load_datastore_config_parameters mysql 5.5 validation-rules.json
```
完成上述後，重新啟動服務：
```sh
sudo service trove-api restart
sudo service trove-taskmanager restart
sudo service trove-conductor restart
```
