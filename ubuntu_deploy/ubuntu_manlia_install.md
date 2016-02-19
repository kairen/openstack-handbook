# Manila 安裝與設定
本章節會說明與操作如何安裝```Share Filesystem Service```到 OpenStack Controller 節點上，並設置相關參數與設定。若對於 Manila 不瞭解的人，可以參考 [Manila 共享式檔案系統服務]()。

### 安裝前準備
我們需要在 Database 底下建立儲存 Manila 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```

建立 Ironic 資料庫與使用者：
```sql
CREATE DATABASE manila;
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'localhost'  IDENTIFIED BY 'MANILA_DBPASSWORD';
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'%'  IDENTIFIED BY 'MANILA_DBPASSWORD';

```
> 這邊若```MANILA_DBPASSWORD```要更改的話，可以更改。

接著要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Manila User
openstack user create --password MANILA_PASSWORD --email manila@example.com manila

# 建立 Manila Role
openstack role add --project service --user manila admin

# 建立 Manila service v1
openstack service create --name manila  --description "Shared File System" share

# 建立 Manila service v2
openstack service create --name manilav2 --description "Shared File System" sharev2

# 建立 Manila URL v1
openstack endpoint create  --publicurl http://%controller%:8786/v1/%\(tenant_id\)s \
--internalurl http://%controller%:8786/v1/%\(tenant_id\)s \
--adminurl http://%controller%:8786/v1/%\(tenant_id\)s \
--region RegionOne share

# 建立 Manila URL v2
openstack endpoint create  --publicurl http://%controller%:8786/v2/%\(tenant_id\)s \
--internalurl http://%controller%:8786/v2/%\(tenant_id\)s \
--adminurl http://%controller%:8786/v2/%\(tenant_id\)s \
--region RegionOne sharev2
```
> 這邊若```MANILA_PASSWORD```要更改的話，可以更改。

# 安裝與設定 Manila 服務
我們透過```apt-get```來安裝套件：
```sh
sudo apt-get install manila-api manila-scheduler manila-common python-manila 
```
接下來編輯```/etc/manila/manila.conf```，並在```[DEFAULT]```部分，設定 RabbitMQ 存取、my ip提供管理與 Keystone 存取：
```sh
[DEFAULT]
...
rpc_backend = rabbit
manila_service_keypair_name = manila-service
enabled_share_backends = generic1
wsgi_keep_alive = False
enabled_share_protocols = NFS,CIFS
neutron_admin_password = NEUTRON_PASSWORD
cinder_admin_password = CINDER_PASSWORD
nova_admin_password = NOVA_PASSWORD
default_share_type = default
state_path = /opt/stack/data/manila
osapi_share_extension = manila.api.contrib.standard_extensions
rootwrap_config = /etc/manila/rootwrap.conf
api_paste_config = /etc/manila/api-paste.ini
share_name_template = share-%s
scheduler_driver = manila.scheduler.filter_scheduler.FilterScheduler
verbose = True
debug = True
auth_strategy = keystone
```
在```[database]```部分設定以下：
```sh
[database]
connection = mysql://manila:MANILA_DBPASSWORD@controller/manila
```
> 這邊若```MANILA_DBPASSWORD```有更改的話，請記得更改。

在```[oslo_messaging_rabbit]```部分，設定RabbitMQ資訊：
```sh
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```部分，設定Keystone資訊，並註解掉其他設定：
```sh
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = manila
password = MANILA_PASSWORD
```
> 這邊若```MANILA_PASSWORD```有更改的話，請記得更改。

在```[generic1]```部分，設定後端儲存：
```sh
[generic1]
driver_handles_share_servers = True
service_instance_user = manila
service_image_name = manila-service-image
path_to_private_key = /home/ubuntu/.ssh/id_rsa
path_to_public_key = /home/ubuntu/.ssh/id_rsa.pub
share_backend_name = GENERIC1
share_driver = manila.share.drivers.generic.GenericShareDriver
```

以上都完成後，同步資料庫：
```sh
sudo manila-manage db sync
```

重啟服務，並刪除預設的 SQLite 資料庫：
```sh
sudo service manila-api restart
sudo service manila-scheduler restart
```

## 參考文獻
* [官方文件](http://docs.openstack.org/developer/manila/adminref/quick_start.html)