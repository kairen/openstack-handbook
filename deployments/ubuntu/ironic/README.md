# Ironic 安裝與設定
本章節會說明與操作如何安裝裸機服務到 Controller 與 Compute 節點上，並修改相關設定檔。若對於 Ironic 不瞭解的人，可以參考 [Ironic 裸機服務套件](../../../conceptions/ironic/README.md)。

- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [設定 Compute Service](#設定-compute-service)
    - [Share 套件安裝與設定](#share-套件安裝與設定)
- [驗證服務](#驗證服務)

# Controller Node
在 Controller 節點我們需要安裝 Ironic 中的 API Server、Conductor 等等的服務。然而在安裝 Ironic 之前需要滿足以下服務才能進行：
* Nova ------- Compute Service
* Keystone --- Identity Service
* Glance ----- Image Service
* Neutron ---- Network Service

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Glance 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Ironic 資料庫：
```sql
CREATE DATABASE ironic;
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost'  IDENTIFIED BY 'IRONIC_DBPASS';
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' IDENTIFIED BY 'IRONIC_DBPASS';
```
> 這邊```IRONIC_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Ironic 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Ironic User
$ openstack user create --domain default --password IRONIC_PASS --email ironic@example.com ironic

# 新增 Ironic 到 Admin Role
$ openstack role add --project service --user ironic admin

# 建立 Ironic service
$ openstack service create --name ironic \
--description "OpenStack Bare Metal Provisioning Service" baremetal

# 建立 Ironic public endpoints
$ openstack endpoint create --region RegionOne \
baremetal public http://10.0.0.11:6385

# 建立 Ironic internal endpoints
$ openstack endpoint create --region RegionOne \
baremetal internal http://10.0.0.11:6385

# 建立 Ironic admin endpoints
$ openstack endpoint create --region RegionOne \
baremetal admin http://10.0.0.11:6385
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install ironic-api ironic-conductor python-ironicclient
```

安裝完成後，編輯```/etc/ironic/ironic.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
enabled_drivers = pxe_ipmitool
my_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.11```。

在```[database]```部分修改使用以下方式：
```sh
[database]
connection = mysql+pymysql://ironic:IRONIC_DBPASS@10.0.0.11/ironic
```
> 這邊```IRONIC_DBPASS```可以隨需求修改。

在```[oslo_messaging_rabbit]```部分加入以下內容：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下內容：
```sh
[keystone_authtoken]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = ironic
password = IRONIC_PASS
```
> 這邊```IRONIC_PASS```可以隨需求修改。

在```[glance]```部分加入以下內容：
```sh
[glance]
glance_host = 10.0.0.11
```

在```[neutron]```部分加入以下內容：
```sh
[neutron]
url = http://10.0.0.11:9696
```

在```[pxe]```部分加入以下內容：
```sh
[pxe]
pxe_append_params = nofb nomodeset vga=normal console=tty0 console=ttyS0,115200n8
tftp_server = 10.0.0.11
tftp_root = /tftpboot
```

在```[oslo_concurrency]```部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/ironic/tmp
```

完成所有設定後，即可同步資料庫來建立 Ironic 資料表：
```sh
$ sudo ironic-dbsync --config-file /etc/ironic/ironic.conf create_schema
```

資料庫建立完成後，就可以重新啟動所有 Ironic 服務：
```sh
sudo service ironic-api restart
sudo service ironic-conductor restart
```

# 設定 Compute Service
在所有的 Compute Service 節點都需要設定使用 Bare Metal 服務驅動程式。這個設定檔一般是```/etc/nva/nova.conf```。該檔案必須設定 Controller 與 Compute 節點。

在任一有 Computer Service 的節點上編輯```/etc/nova/nova.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
compute_scheduler_driver = nova.scheduler.filter_scheduler.FilterScheduler
compute_driver = ironic.nova.virt.ironic.IronicDriver
compute_manager = ironic.nova.compute.manager.ClusteredComputeManager
scheduler_host_manager = nova.scheduler.ironic_host_manager.IronicHostManager
ram_allocation_ratio = 1.0
reserved_host_memory_mb = 0
scheduler_use_baremetal_filters = True
scheduler_tracks_instance_changes = False
```

在```[ironic]```部分加入以下內容：
```
[ironic]
admin_url = http://10.0.0.11:35357/v2.0
api_endpoint = http://10.0.0.11:6385/v1
admin_tenant_name = service
admin_username = ironic
admin_password = IRONIC_PASS
```

所有 Compute Service 節點完成後，在 ```Controller``` 節點重新啟動以下服務：
```sh
$ sudo service nova-scheduler restart
```

在 ```Compute``` 節點重新啟動以下服務：
```sh
$ sudo service nova-compute restart
```

* [Ironic installation and configuration guide](http://amar266.blogspot.tw/2014/12/ironic-installation-and-configuration.html)
* [Ironic 部署](http://m.blog.csdn.net/article/details?id=50162441)
