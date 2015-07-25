# Glance 安裝與設定
本章節會說明與操作如何安裝```Image```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Glance不瞭解的人，可以參考[Glance 映像檔套件章節](glance.html)。

### 安裝前準備
我們需要在Database底下建立儲存 Glance 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Glance 資料庫與使用者：
```sql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```
> 這邊若```GLANCE_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Glance User
openstack user create --password GLANCE_PASS --email glance@example.com glance
# 建立 Glance Role
openstack role add --project service --user glance admin
# 建立 Glance service
openstack service create --name glance  --description "OpenStack Image service" image
# 建立 Glance Endpoints
openstack endpoint create  --publicurl http://controller:9292  --internalurl http://controller:9292  --adminurl http://controller:9292  --region RegionOne image
```

# 安裝與設置Glance套件
首先要透過```apt-get```來安裝套件：
```sh
sudo apt-get install -y glance python-glanceclient
```
安裝完成後，編輯```/etc/glance/glance-api.conf```，在```[database]```，註解掉sqlite，並加入以下：
```sh
[database]
...
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql://glance:GLANCE_DBPASS@controller/glance
```
> 這邊若```GLANCE_DBPASS```有更改的話，請記得更改。

接下來，在```[keystone_authtoken]```和```[[paste_deploy]```[部分，```Keystone```的admin：
```sh
[keystone_authtoken]
...
# identity_uri = http://127.0.0.1:35357
# admin_tenant_name = %SERVICE_TENANT_NAME%
# admin_user = %SERVICE_USER%
# admin_password = %SERVICE_PASSWORD%
# revocation_cache_time = 10

auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone
```
在```[glance_store]```部分，加入以下：
```sh
[glance_store]
...
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
在```[DEFAULT]```部分，設定noop 訊息驅動來禁用訊息，因為它們只與選擇性套件```Telemetry (Ceilometer)```服務有關：
```sh
[DEFAULT]
...
notification_driver = noop
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成後，還要編輯```/etc/glance/glance-registry.conf```並完成以下設定，在```[database]```部分如上面一樣：
```sh
[database]
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql://glance:GLANCE_DBPASS@controller/glance
```
接下來，在```[keystone_authtoken]```和```[paste_deploy]```部分，```Keystone```的admin：
```sh
[keystone_authtoken]
...
# identity_uri = http://127.0.0.1:35357
# admin_tenant_name = %SERVICE_TENANT_NAME%
# admin_user = %SERVICE_USER%
# admin_password = %SERVICE_PASSWORD%

auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = GLANCE_PASS

[paste_deploy]
...
flavor = keystone
```
在```[DEFAULT]```部分，設定noop 訊息驅動來禁用訊息，因為它們只與選擇性套件```Telemetry (Ceilometer)```服務有關：
```sh
[DEFAULT]
...
notification_driver = noop
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成以上兩個檔案```/etc/glance/glance-api.conf```與```/etc/glance/glance-registry.conf```後，即可同步資料庫：
```sh
sudo glance-manage db_sync
```
完成後，重啟服務，並刪除SQLite檔案：
```sh
sudo service glance-registry restart
sudo service glance-api restart
sudo rm -f /var/lib/glance/glance.sqlite
```
# 驗證操作
首先我們要在```admin-openrc.sh```與```demo-openrc.sh```加入Glance API環境變數：
```sh
echo "export OS_IMAGE_API_VERSION=2" | sudo  tee -a admin-openrc.sh demo-openrc.sh
```
然後透過```source```來加入環境變數：
```sh
source admin-openrc.sh
```
建立一個暫時用目錄，並下載測試用映像檔：
```sh
mkdir /tmp/images
wget -P /tmp/images http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```
利用```QCOW2```磁碟格式，與```空白的```容器格式，並且為公有映樣檔，來提供往後使用：
```sh
glance image-create --name "cirros-0.3.4-x86_64" --file /tmp/images/cirros-0.3.4-x86_64-disk.img  --disk-format qcow2 --container-format bare --visibility public --progress
```
> * 有關```glance image-create```指令的參數，可以參考[Glacne Doc](http://docs.openstack.org/cli-reference/content/glanceclient_commands.html#glanceclient_subcommand_image-create)的指令指南。
* 有關映像檔格式的資訊，可以參考[Disk and container formats for images](http://docs.openstack.org/image-guide/content/image-formats.html)。

成功後會看到以下資訊：
```
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2015-06-24T07:13:45Z                 |
| disk_format      | qcow2                                |
| id               | 638a1646-bb2e-4f61-9bb9-d5280069b4a8 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64                  |
| owner            | cf8f9b8b009b429488049bb2332dc311     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2015-06-24T07:13:45Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```
我們可以透過```glance image-list```來看所有的映像檔：
```sh
glance image-list
```
最後刪除剛剛的映像檔：
```sh
rm -r /tmp/images
```
