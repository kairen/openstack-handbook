# Manila 安裝與設定
本章節會說明與操作如何安裝共享式檔案系統服務到 Controller 與 Share 節點上，並修改相關設定檔。若對於 Manila 不瞭解的人，可以參考 [Manila 共享式檔案系統服務章節](../../../conceptions/manila/README.md)。

- [部署前系統環境準備](#部署前系統環境準備)
    - [硬體規格與網路分配](#硬體規格與網路分配)
    - [系統環境設定與安裝](#系統環境設定與安裝)
- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [Share Node](#share-node)
    - [Share 套件安裝與設定](#share-套件安裝與設定)
- [驗證服務](#驗證服務)

# 部署前系統環境準備
當要加入 Manila 來提供持久性儲存給虛擬機實例使用時，必須額外新增節點來提供實際儲存。首先如教學最開始的步驟，要先設定基本主機環境與安裝基本套件。

### 硬體規格與網路分配
這邊只會加入一台 Share 節點，規格如下所示：
* **Share Node**: 雙核處理器, 8 GB 記憶體, 250 GB 硬碟（/dev/sda）。

在節點上需要提供對映的多張網卡（NIC）來設定給不同網路使用：
* **Management（管理網路）**：10.0.0.0/24，需要一個 Gateway 並設定為 10.0.0.1。
> 這邊需要提供一個 Gateway 來提供給所有節點使用內部的私有網路，該網路必須能夠連接網際網路來讓主機進行套件安裝與更新等。


這邊將第一張網卡介面設定為 ```Management（管理網路）```：
* IP address：10.0.0.61
* Network mask：255.255.255.0 (or /24)
* Default gateway：10.0.0.1

### 系統環境設定與安裝
這邊假設作業系統已經安裝完成，且已正常執行系統後，我們必須在每個節點準備以下設定。首先下面指令可以設定系統的 User 執行 root 權限時不需要密碼驗證：
```sh
$ echo "ubuntu ALL = (root) NOPASSWD:ALL" \
| sudo tee /etc/sudoers.d/ubuntu && sudo chmod 440 /etc/sudoers.d/ubuntu
```

安裝 NTP 來跟叢集時間作同步：
```sh
$ sudo apt-get install -y ntp
```

安裝完成後，編輯```/etc/ntp.conf```設定檔，並註解掉所有 ```server```，然後加入以下內容：
```sh
server 10.0.0.11 iburst
```
> 完成後重新啟動 NTP。

接著編輯```/etc/hostname```來改變主機名稱（Option）：
```sh
share
```

並設定主機 IP 與名稱的對照，編輯```/etc/hosts```檔案加入以下內容：
```sh
10.0.0.11   controller
10.0.0.21   network
10.0.0.31   compute1
10.0.0.61   share
```
> P.S. 若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。

最後要新增 OpenStack Repository，來取的要安裝套件：
```sh
$ sudo apt-get install -y software-properties-common
$ sudo add-apt-repository -y cloud-archive:mitaka
```
> 若要安裝 ``` pre-release``` 測試版本，修改為```cloud-archive:mitaka-proposed```。

> 若要安裝 ```liberty```，修改為```cloud-archive:liberty```。

更新 Repository 與系統核心套件：
```sh
$ sudo apt-get update && sudo apt-get -y dist-upgrade
```
> 如果 Upgrade 包含了新的核心套件的話，請重新開機。

# Controller Node
在 Controller 節點我們需要安裝 Manila 中的 API Server 與 Scheduler 等等服務。

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Manila 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Manila 資料庫：
```sql
CREATE DATABASE manila;
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'localhost' IDENTIFIED BY 'MANILA_DBPASS';
GRANT ALL PRIVILEGES ON manila.* TO 'manila'@'%' IDENTIFIED BY 'MANILA_DBPASS';
```
> 這邊```MANILA_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Manila 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Manila user
$ openstack user create --domain default --password MANILA_PASS --email manila@example.com manila

# 新增 Manila 到 Admin Role
$ openstack role add --project service --user manila admin

# 建立 Manila service
$ openstack service create --name manila \
--description "OpenStack Shared File Systems" share

# 建立 Manila v2 service
$ openstack service create --name manilav2 \
--description "OpenStack Shared File Systems" sharev2

# 建立 Manila v1 public endpoints
$ openstack endpoint create --region RegionOne \
share public http://10.0.0.11:8786/v1/%\(tenant_id\)s

# 建立 Manila v1 internal endpoints
$ openstack endpoint create --region RegionOne \
share internal http://10.0.0.11:8786/v1/%\(tenant_id\)s

# 建立 Manila v1 admin endpoints
$ openstack endpoint create --region RegionOne \
share admin http://10.0.0.11:8786/v1/%\(tenant_id\)s

# 建立 Manila v2 public endpoints
$ openstack endpoint create --region RegionOne \
sharev2 public http://10.0.0.11:8786/v2/%\(tenant_id\)s

# 建立 Manila v2 internal endpoints
$ openstack endpoint create --region RegionOne \
sharev2 internal http://10.0.0.11:8786/v2/%\(tenant_id\)s

# 建立 Manila v2 admin endpoints
$ openstack endpoint create --region RegionOne \
sharev2 admin http://10.0.0.11:8786/v2/%\(tenant_id\)s
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install manila-api manila-scheduler python-manilaclient
```

安裝完成後，編輯```/etc/manila/manila.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
default_share_type = default_share_type
scheduler_driver = manila.scheduler.filter_scheduler.FilterScheduler
share_name_template = share-%s
rootwrap_config = /etc/manila/rootwrap.conf
api_paste_config = /etc/manila/api-paste.ini
rpc_backend = rabbit
auth_strategy = keystone

my_ip = 10.0.0.11
```
> 這邊```NOVA_PASS```、```CINDER_PASS```、```NEUTRON_PASS```可以隨需求修改。

在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://manila:MANILA_DBPASS@10.0.0.11/manila
```
> 這邊```MANILA_DBPASS```可以隨需求修改。

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
auth_plugin = password
project_domain_name = default
user_domain_name = default
project_name = service
username = manila
password = MANILA_PASS
```
> 這邊```MANILA_PASS```可以隨需求修改。

在```[neutron]```部分加入以下內容：
```
[neutron]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

在```[nova]```部分加入以下內容：
```
[nova]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = nova
password = NOVA_PASS
```
> 這邊```NOVA_PASS```可以隨需求修改。

在```[cinder]```部分加入以下內容：
```
[cinder]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = cinder
password = CINDER_PASS
```
> 這邊```CINDER_PASS```可以隨需求修改。

在```[oslo_concurrency]```部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/manila/tmp
```

完成所有設定後，即可同步資料庫來建立 Manila 資料表：
```sh
$ sudo manila-manage db sync
```

資料庫建立完成後，就可以重新啟動所有 Manila 服務：
```sh
sudo service manila-scheduler restart
sudo service manila-api restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/manila/manila.sqlite
```

# Share Node
安裝與設定完成 Controller 上的 Manila 所有服務後，接著要來設定實際提供檔案系統的 Share 節點。該節點只會安裝一些 Linux 相關套件與 manila-share 服務。

### Share 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install manila-common python-pymysql neutron-linuxbridge-agent conntrack
```
> 這邊```neutron-linuxbridge-agent```要隨網路部署的 agent 而變。如採用 OVS 則使用以下方式：
```sh
$ sudo apt-get install manila-common python-pymysql neutron-openvswitch-agent conntrack
```

安裝完成後，編輯```/etc/manila/manila.conf```設定檔，並在```[DEFAULT]```部分設定以下內容：
```
[DEFAULT]
...
share_name_template = share-%s
rootwrap_config = /etc/manila/rootwrap.conf
api_paste_config = /etc/manila/api-paste.ini
default_share_type = default_share_type
enabled_share_backends = generic
enabled_share_protocols = NFS,CIFS
rpc_backend = rabbit
auth_strategy = keystone
network_api_class = manila.network.neutron.neutron_network_plugin.NeutronNetworkPlugin

my_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.61```。

在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://manila:MANILA_DBPASS@10.0.0.11/manila
```
> 這邊```MANILA_DBPASS```可以隨需求修改。

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
auth_plugin = password
project_domain_name = default
user_domain_name = default
project_name = service
username = manila
password = MANILA_PASS
```
> 這邊```MANILA_PASS```可以隨需求修改。

在```[generic]```部分加入以下內容：
```
[generic]
share_backend_name = GENERIC
share_driver = manila.share.drivers.generic.GenericShareDriver
share_volume_fstype = ext4

driver_handles_share_servers = True

service_instance_flavor_id = 2
service_image_name = manila-service-image
service_instance_user = manila
service_instance_password = manila
service_instance_network_helper_type = neutron
interface_driver = manila.network.linux.interface.BridgeInterfaceDriver
```
> 這邊```interface_driver```要隨部署的網路架構改變。

> Open vSwitch 使用 ```manila.network.linux.interface.OVSInterfaceDriver```。

在```[neutron]```部分加入以下內容：
```
[neutron]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

在```[nova]```部分加入以下內容：
```
[nova]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = nova
password = NOVA_PASS
```
> 這邊```NOVA_PASS```可以隨需求修改。

在```[cinder]```部分加入以下內容：
```
[cinder]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
project_domain_id = default
project_name = service
auth_type = password
user_domain_id = default
username = cinder
password = CINDER_PASS
```
> 這邊```CINDER_PASS```可以隨需求修改。

在```[oslo_concurrency]```部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/manila/tmp
```

建立一個 Upstart 檔案來管理 manila-share ：
```sh
cat > /etc/init/manila-share.conf << EOF
description "OpenStack Manila Share"
author "kyle Bai <kyle.b@inwinStack.com>"

start on runlevel [2345]
stop on runlevel [!2345]

exec start-stop-daemon --start --chuid manila --exec /usr/bin/manila-share -- --config-file=/etc/manila/manila.conf
EOF
```

完成所有安裝後，即可重新啟動服務：
```sh
$ sudo start manila-share
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/manila/manila.sqlite
```

# 驗證服務
首先回到 ```Controller``` 節點並接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

首先透過 Manila client 來查看服務列表，如以下方式：
```sh
$ manila service-list
+----+------------------+---------------+------+---------+-------+----------------------------+
| Id | Binary           | Host          | Zone | Status  | State | Updated_at                 |
+----+------------------+---------------+------+---------+-------+----------------------------+
| 1  | manila-scheduler | controller    | nova | enabled | up    | 2016-04-06T15:01:15.000000 |
| 2  | manila-share     | share@generic | nova | enabled | up    | 2016-04-06T15:01:13.000000 |
+----+------------------+---------------+------+---------+-------+----------------------------+
```

下載將使用的 Manila service 映像檔：
```sh
$ wget http://tarballs.openstack.org/manila-image-elements/images/manila-service-image-master.qcow2
```
> 其他版本可以參考 [Manila image elements](https://github.com/openstack/manila-image-elements)。

然後使用 Glance client 來上傳 Manila Service 映像檔：
```sh
$ glance image-create --name manila-service-image \
--disk-format qcow2 --container-format bare \
--file manila-service-image-master.qcow2
```

透過 Manila client 來建立 Share Type，如以下方式：
```sh
$ manila type-create default_share_type True
+----------------------+--------------------------------------+
| Property             | Value                                |
+----------------------+--------------------------------------+
| required_extra_specs | driver_handles_share_servers : True  |
| Name                 | default_share_type                   |
| Visibility           | public                               |
| is_default           | -                                    |
| ID                   | ae825759-a3e5-4aec-8c78-138866bab7c4 |
| optional_extra_specs | snapshot_support : True              |
+----------------------+--------------------------------------+
```

接著透過 Manila client 來建立 Share Network，如以下方式：
```sh
$ NET_ID=$(neutron net-list | awk '/ admin-net / { print $2 }')
$ SUBNET_ID=$(neutron net-list | awk '/ admin-net / { print $6 }')
$ manila share-network-create \
--name share_network \
--neutron-net-id ${NET_ID} \
--neutron-subnet-id ${SUBNET_ID}
```

完成上述後，即可建立檔案系統：
```sh
$ manila create NFS 1 --name share01 --share-network share_network
```
