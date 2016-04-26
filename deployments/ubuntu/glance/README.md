# Glance 安裝與設定
本章節會說明與操作如何安裝映像檔服務到 Controller 節點上，並設置相關參數與設定。若對於 Glance 不瞭解的人，可以參考 [Glance 映像檔服務章節](../../../conceptions/glance/README.md)。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Glance 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Glance 資料庫：
```sql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'GLANCE_DBPASS';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';
```
> 這邊```GLANCE_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Glance 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Glance user
$ openstack user create --domain default --password GLANCE_PASS --email glance@example.com glance

# 新增 Glance 到 Admin Role
$ openstack role add --project service --user glance admin

# 建立 Glance service
$ openstack service create --name glance  --description "OpenStack Image service" image

# 建立 Glance public endpoints
$ openstack endpoint create --region RegionOne \
image public http://10.0.0.11:9292

# 建立 Glance internal endpoints
$ openstack endpoint create --region RegionOne \
image internal http://10.0.0.11:9292

# 建立 Glance admin endpoints
$ openstack endpoint create --region RegionOne \
image admin http://10.0.0.11:9292
```
> 在 v3 版本中，可以加入```-f <json, shell, table, yaml>```來檢視 keystone 資訊。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y glance python-glanceclient
```

安裝完成後，編輯 ```/etc/glance/glance-api.conf``` 設定檔，在```[database]```部分修改使用以下方式：
```sh
[database]
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:GLANCE_DBPASS@10.0.0.11/glance
```
> 這邊```GLANCE_DBPASS```可以隨需求修改。

接下來，在```[keystone_authtoken]```部分加入以下內容：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS
```
> 這邊```GLANCE_PASS```可以隨需求修改。

在```[paste_deploy]```部分加入以下內容：
```sh
[paste_deploy]
flavor = keystone
```

在```[glance_store]```部分加入以下內容：
```sh
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```
> 其中```filesystem_store_datadir```是當映像檔上傳時，檔案放置的目錄。

完成後，要接著編輯```/etc/glance/glance-registry.conf```並完成以下設定，在```[database]```部分修改使用以下方式：
```sh
[database]
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:GLANCE_DBPASS@10.0.0.11/glance
```
> 這邊```GLANCE_DBPASS```可以隨需求修改。

接下來，在```[keystone_authtoken]```部分加入以下內容：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = GLANCE_PASS
```
> 這邊```GLANCE_PASS```可以隨需求修改。

在```[paste_deploy]```部分加入以下內容：
```sh
[paste_deploy]
flavor = keystone
```

完成以上兩個檔案設定後，即可同步資料庫來建立資料表：
```sh
$ sudo glance-manage db_sync
```

完成後，重新啟動 Glance 的服務：
```sh
sudo service glance-registry restart
sudo service glance-api restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/glance/glance.sqlite
```

### 驗證服務
首先接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

從網路上下載一個測試用映像檔 ```cirros-0.3.4-x86_64```：
```sh
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

這邊透過指令將映像檔上傳，採用 ```QCOW2``` 格式，並且設定為公開的映像檔，來提供給雲端租戶們使用：
```sh
$ glance image-create --name "cirros-0.3.4-x86_64" \
--file cirros-0.3.4-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--visibility public --progress
```
> * 有關 ```glance image-create``` 指令的參數，可以參考 [Glacne Doc](http://docs.openstack.org/cli-reference/content/glanceclient_commands.html#glanceclient_subcommand_image-create) 的指令指南。
* 有關映像檔格式的資訊，可以參考 [Disk and container formats for images](http://docs.openstack.org/image-guide/content/image-formats.html)。

成功後會看到以下資訊：
```
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2016-03-30T16:04:32Z                 |
| disk_format      | qcow2                                |
| id               | 520bf946-436d-4fbd-a21b-62d5879c966e |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64                  |
| owner            | 136884a1934f4d4c950e1397797b7a68     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2016-03-30T16:04:32Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

完成後，可以透過 Glance client 程式來查看所有映像檔，指令如下：
```sh
$ glance image-list
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| 520bf946-436d-4fbd-a21b-62d5879c966e | cirros-0.3.4-x86_64 |
+--------------------------------------+---------------------+
```
