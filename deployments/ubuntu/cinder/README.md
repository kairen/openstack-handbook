# Cinder 安裝與設定
本章節會說明與操作如何安裝區塊儲存服務到 Controller 與 Storage 節點上，並修改相關設定檔。若對於 Cinder 不瞭解的人，可以參考 [Cinder 區塊儲存服務章節](../../../conceptions/cinder/README.md)。

- [部署前系統環境準備](#部署前系統環境準備)
    - [硬體規格與網路分配](#硬體規格與網路分配)
    - [系統環境設定與安裝](#系統環境設定與安裝)
- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [Storage Node](#storage-node)
    - [Storage 安裝前準備](#storage-安裝前準備)
    - [Storage 套件安裝與設定](#storage-套件安裝與設定)
- [驗證服務](#驗證服務)

# 部署前系統環境準備
當要加入 Cinder 來提供持久性儲存給虛擬機實例使用時，必須額外新增節點來提供實際儲存。首先如教學最開始的步驟，要先設定基本主機環境與安裝基本套件。

### 硬體規格與網路分配
這邊只會加入一台 Storage 節點，規格如下所示：
* **Storage Node**: 雙核處理器, 8 GB 記憶體, 250 GB 硬碟（/dev/sda）與 500 GB 硬碟（/dev/sdb）。

在節點上需要提供對映的多張網卡（NIC）來設定給不同網路使用：
* **Management（管理網路）**：10.0.0.0/24，需要一個 Gateway 並設定為 10.0.0.1。
> 這邊需要提供一個 Gateway 來提供給所有節點使用內部的私有網路，該網路必須能夠連接網際網路來讓主機進行套件安裝與更新等。

* **Storage（儲存網路）**：10.0.2.0/24，不需要 Gateway。
> P.S. 該網路並非必要，若想使用如 NAS 或 SAN 的儲存來給 Cinder 充當後端儲存的話，可以考慮加入一個獨立的網路給這些儲存系統使用。

這邊將第一張網卡介面設定為 ```Management（管理網路）```：
* IP address：10.0.0.51
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
block1
```

並設定主機 IP 與名稱的對照，編輯```/etc/hosts```檔案加入以下內容：
```sh
10.0.0.11   controller
10.0.0.21   network
10.0.0.31   compute1
10.0.0.41   block1
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
在 Controller 節點我們需要安裝 Cider 中的 API Server 與 Scheduler 服務。

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Cinder 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Cinder 資料庫：
```sql
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%'  IDENTIFIED BY 'CINDER_DBPASS';
```
> 這邊```CINDER_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Cinder 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Cinder user
$ openstack user create --domain default --password CINDER_PASS --email cinder@example.com cinder

# 新增 Cinder 到 Admin Role
$ openstack role add --project service --user cinder admin

# 建立 Cinder service
$ openstack service create --name cinder \
--description "OpenStack Block Storage" volume

# 建立 Cinder service v2
$ openstack service create --name cinderv2 \
--description "OpenStack Block Storage" volumev2

# 建立 Cinder v1 public endpoints
$ openstack endpoint create --region RegionOne \
volume public http://10.0.0.11:8776/v1/%\(tenant_id\)s

# 建立 Cinder v1 internal endpoints
$ openstack endpoint create --region RegionOne \
volume internal http://10.0.0.11:8776/v1/%\(tenant_id\)s

# 建立 Cinder v1 admin endpoints
$ openstack endpoint create --region RegionOne \
volume admin http://10.0.0.11:8776/v1/%\(tenant_id\)s

# 建立 Cinder v2 public endpoints
$ openstack endpoint create --region RegionOne \
volumev2 public http://10.0.0.11:8776/v2/%\(tenant_id\)s

# 建立 Cinder v2 internal endpoints
$ openstack endpoint create --region RegionOne \
volumev2 internal http://10.0.0.11:8776/v2/%\(tenant_id\)s

# 建立 Cinder v2 admin endpoints
$ openstack endpoint create --region RegionOne \
volumev2 admin http://10.0.0.11:8776/v2/%\(tenant_id\)s
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install cinder-api cinder-scheduler python-cinderclient
```

安裝完成後，編輯```/etc/cinder/cinder.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.11```。

在```[database]```部分修改使用以下方式：
```sh
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@10.0.0.11/cinder
```

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
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```
> 這邊```CINDER_PASS```可以隨需求修改。

在```[oslo_concurrency]```部分加入以下內容：
```sh
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

完成所有設定後，即可同步資料庫來建立 Cinder 資料表：
```sh
$ sudo cinder-manage db sync
```

接著編輯```/etc/nova/nova.conf```設定檔，在```[cinder]```加入以下內容，讓 Nova 使用 Volume：
```
[cinder]
os_region_name = RegionOne
```

重新啟動 Nova API 服務：
```sh
$ sudo service nova-api restart
```

重新啟動所有 Cinder 服務：
```sh
sudo service cinder-scheduler restart
sudo service cinder-api restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/cinder/cinder.sqlite
```

# Storage Node
安裝與設定完成 Controller 上的 Cinder 所有服務後，接著要來設定實際儲存資料的 Storage 節點。該節點只會安裝一些 Linux 相關套件與 cinder-volume 服務。

### Storage 安裝前準備
在開始設定之前，首先要安裝 ```lvm2``` 軟體：
```sh
$ sudo apt-get install -y lvm2
```
> P.S. 有些版本的 Ubuntu 預設可能已經安裝 ```lvm2```。

由於本教學採用 LVM（Logical Volume Manager）來提供儲存給虛擬機使用，因此這邊需要先建立 LVM Physical Volume。這邊將 Storage 節點的 ```/dev/sdb``` 硬碟當作 Physical Volume 使用：
```sh
$ sudo pvcreate /dev/sdb
Physical volume "/dev/sdb" successfully created
```
> 若有多顆硬碟則重複方式建立。
> 若該硬碟有過去使用的資料與分區的話，可以使用以下兩個指令來解決：
```sh
$ sudo fdisk /dev/sdb
$ sudo mkfs -t ext4 /dev/sdb
```

接著要建立一個 LVM Volume Group 來讓多顆硬碟當作一個邏輯儲存使用：
```sh
$ sudo vgcreate cinder-volumes /dev/sdb
Volume group "cinder-volumes" successfully created
```

由於 Cinder 會使用被建立成 LVM 的裝置來提供區塊儲存。為了確保儲存安全問題，故任何作業系統底層不應該被任意存取該 LVM volume。且預設下的 LVM 會透過工具搜尋包含 ```/dev``` 的區塊儲存裝置目錄。如果部署時是使用 LVM 來提供 Volume 的話，該工具會檢查這些 Volume，並試著快取目錄，這樣將會造成各式各樣的問題，因此要編輯 ```/etc/lvm/lvm.conf``` 來正確的提供 Volume Group 的硬碟使用。這邊設定只使用 ```/dev/sdb```：
```sh
filter = [ 'a/sdb/', 'r/.*/']
```
> 這邊以 ```a``` 開頭的表示 accept，而 ```r``` 則表示 reject。

當完成上述過程後，最後要確認建置無誤，透過以下指令來查看 Volume Group：
```sh
$ sudo  pvdisplay
--- Physical volume ---
PV Name               /dev/sdb
VG Name               cinder-volumes
PV Size               465.81 GiB / not usable 3.18 MiB
Allocatable           yes
PE Size               4.00 MiB
Total PE              59618
Free PE               59362
Allocated PE          256
PV UUID               tsYzd6-a4s8-32iL-aYdA-AMol-qtj2-4RMhvA
```

### Storage 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install cinder-volume python-mysqldb
```

安裝完成後，編輯```/etc/cinder/cinder.conf```設定檔，並在```[DEFAULT]```部分設定以下內容：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
enabled_backends = lvm

my_ip = MANAGEMENT_IP
glance_api_servers = http://10.0.0.11:9292
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.41```。

在```[database]```部分修改使用以下方式：
```sh
[database]
connection = mysql+pymysql://cinder:CINDER_DBPASS@10.0.0.11/cinder
```

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
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = CINDER_PASS
```
> 這邊```CINDER_PASS```可以隨需求修改。

在```[lvm]```部分加入以下內容：
```sh
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm
```

在```[oslo_concurrency]```部分加入以下內容：
```sh
[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```

完成所有安裝後，即可重新啟動服務：
```sh
sudo service tgt restart
sudo service cinder-volume restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/cinder/cinder.sqlite
```

# 驗證服務
首先回到 ```Controller``` 節點並在```admin-openrc```與```demo-openrc```加入 Cinder API 使用版本的環境變數：
```sh
$ echo "export OS_VOLUME_API_VERSION=2" \
| sudo tee -a admin-openrc demo-openrc
```

接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Cinder client 來查看服務列表，如以下方式：
```sh
$ cinder service-list
+------------------+------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |    Host    | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller | nova | enabled |   up  | 2015-06-30T13:58:21.000000 |       None      |
|  cinder-volume   | block1@lvm | nova | enabled |   up  | 2015-06-30T13:58:22.000000 |       None      |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
```

透過 Cinder client 來建立區塊儲存，如以下方式：
```sh
$ cinder create --name admin-volume1 1
+---------------------------------------+--------------------------------------+
|                Property               |                Value                 |
+---------------------------------------+--------------------------------------+
|              attachments              |                  []                  |
|           availability_zone           |                 nova                 |
|                bootable               |                false                 |
|          consistencygroup_id          |                 None                 |
|               created_at              |      2015-04-21T23:46:08.000000      |
|              description              |                 None                 |
|               encrypted               |                False                 |
|                   id                  | 6c7a3d28-e1ef-42a0-b1f7-8d6ce9218412 |
|                metadata               |                  {}                  |
|              multiattach              |                False                 |
|                  name                 |             admin-volume1            |
|      os-vol-tenant-attr:tenant_id     |   ab8ea576c0574b6092bb99150449b2d3   |
|   os-volume-replication:driver_data   |                 None                 |
| os-volume-replication:extended_status |                 None                 |
|           replication_status          |               disabled               |
|                  size                 |                  1                   |
|              snapshot_id              |                 None                 |
|              source_volid             |                 None                 |
|                 status                |               creating               |
|                user_id                |   3a81e6c8103b46709ef8d141308d4c72   |
|              volume_type              |                 None                 |
+---------------------------------------+--------------------------------------+
```

這邊可以透過 Cinder client 來查看區塊儲存列表，如以下方式：
```sh
$ cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0d81efd2-67f8-4abe-a9a7-ade2944168b5 | available |admin-volume1 |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```
> 若看到 ```available``` 表示建立沒有錯誤。若不幸發生錯誤可以到 Controller 節點查看 Logs。

最後也可以開啟 Dashboard 來查看部署是否成功，這邊查看[雲硬碟]管理介面。
![Dashboard](images/dashboard_hard.png)
