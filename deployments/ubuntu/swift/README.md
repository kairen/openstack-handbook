# Swift 安裝與設定
本章節會說明與操作如何安裝物件儲存服務到 Controller 與 Storage 節點上，並修改相關設定檔。若對於 Swift 不瞭解的人，可以參考 [Swift 物件儲存套件章節](../../../conceptions/swift/README.md)。

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
當要加入 Swift 來提供物件儲存給雲端租戶時，必須額外新增節點來提供實際儲存。首先如教學最開始的步驟，要先設定基本主機環境與安裝基本套件。

### 硬體規格與網路分配
這邊會加入兩台 Storage 節點，規格如下所示：
* **Storage Node**: 雙核處理器, 8 GB 記憶體, 250 GB 硬碟（/dev/sda）與 500 GB 硬碟（/dev/sdb）。

在節點上需要提供對映的多張網卡（NIC）來設定給不同網路使用：
* **Management（管理網路）**：10.0.0.0/24，需要一個 Gateway 並設定為 10.0.0.1。
> 這邊需要提供一個 Gateway 來提供給所有節點使用內部的私有網路，該網路必須能夠連接網際網路來讓主機進行套件安裝與更新等。

* **Storage（儲存網路）**：10.0.2.0/24，不需要 Gateway。
> P.S. 該網路並非必要，若想使用如 NAS 或 SAN 的儲存來給 Swift 充當後端儲存的話，可以考慮加入一個獨立的網路給這些儲存系統使用。

這邊將第一張網卡介面設定為 ```Management（管理網路）```：
* IP address：10.0.0.51
* Network mask：255.255.255.0 (or /24)
* Default gateway：10.0.0.1

> 另一台以尾數 IP 類推。

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
object1
```
> 另一台以尾數數字類推。

並設定主機 IP 與名稱的對照，編輯```/etc/hosts```檔案加入以下內容：
```sh
10.0.0.11   controller
10.0.0.21   network
10.0.0.31   compute1

10.0.0.51   object1
10.0.0.52   object2
```
> P.S. 若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。

最後要新增 OpenStack Repository，來取的要安裝套件：
```sh
$ sudo apt-get install -y software-properties-common
$ sudo add-apt-repository -y cloud-archive:liberty
```
> 若要安裝 ```kilo```，修改為```cloud-archive:kilo```。

更新 Repository 與系統核心套件：
```sh
$ sudo apt-get update && sudo apt-get -y dist-upgrade
```
> 如果 Upgrade 包含了新的核心套件的話，請重新開機。

# Controller Node
在 Controller 節點我們需要安裝 Swift 中的 Proxy 服務。

### Controller 安裝前準備
Swift 與其他服務不同，Controller 節點不使用任何資料庫，取而代之是在每個 Storage 節點上安裝 SQLite 資料庫。

首先要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Swift 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Swift user
$ openstack user create --domain default --password SWIFT_PASS --email swift@example.com swift

# 建立 Swift role
$ openstack role add --project service --user swift admin

# 建立 Swift service
$ openstack service create --name swift  --description "OpenStack Object Storage" object-store

# 建立 Swift v1 public endpoints
$ openstack endpoint create --region RegionOne \
object-store public http://10.0.0.11:8080/v1/AUTH_%\(tenant_id\)s

# 建立 Swift v1 internal endpoints
$ openstack endpoint create --region RegionOne \
object-store internal http://10.0.0.11:8080/v1/AUTH_%\(tenant_id\)s

# 建立 Swift v1 admin endpoints
$ openstack endpoint create --region RegionOne \
object-store admin http://10.0.0.11:8080/v1
```


### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install swift swift-proxy python-swiftclient \
python-keystoneclient python-keystonemiddleware \
memcached
```

安裝完成後，建立```/etc/swift```，並透過網路來從物件儲存```Repository```中取得代理服務設定檔案：
```sh
sudo curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/liberty
```
編輯```/etc/swift/proxy-server.conf```在```[DEFAULT]```部分設定服務port與目錄：
```
[DEFAULT]
...
bind_port = 8080
user = swift
swift_dir = /etc/swift
```

在```[pipeline:main]```部分，啟用合適的模組：
```
[pipeline:main]
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[app:proxy-server] ```部分，啟用帳號建立：
```
[app:proxy-server]
...
use = egg:swift#proxy
account_autocreate = true
```

在```[filter:keystoneauth]```部分，啟用設定操作者角色：
```
[filter:keystoneauth]
use = egg:swift#keystoneauth
...
operator_roles = admin,user
```

在```[filter:authtoken]```部分，設定身份驗證存取，並註解掉其他不需要的：
```
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = swift
password = SWIFT_PASS
delay_auth_decision = true
```
> 這邊若```SWIFT_PASS```有更改的話，請記得更改。

在```[filter:cache]```部分，設定```memcached```位置：
```sh
[filter:cache]
use = egg:swift#memcache
...
memcache_servers = 127.0.0.1:11211
```

# Storage Node
安裝與設定完成 Controller 上的 Swift 所有服務後，接著要來設定實際儲存資料的 Storage 節點。

### 安裝前準備
首先透過```apt-get```安裝相關套件：
```sh
sudo apt-get install xfsprogs rsync
```

然後將```/dev/sdb```進行格式化：
```sh
sudo mkfs.xfs -f /dev/sdb
```

建立 Monut 載點目錄：
```sh
sudo mkdir -p /srv/node/sdb
```

然後編輯```/etc/fstab```檔案，加入以下：
```
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
```

Mount 裝置：
```sh
sudo mount /srv/node/sdb
```

編輯```/etc/rsyncd.conf```，加入以下：
```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_IP

[account]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/object.lock
```
> 將```MANAGEMENT_IP```取代為主機 Management 網路介面IP，rsync 服務不需要認證，因此可以考慮將其運作在私有網路。

編輯```/etc/default/rsync```，並開啟rsync服務：
```
RSYNC_ENABLE=true
```

重啟服務：
```sh
sudo service rsync start
```

### 安裝與設置Swift套件
透過```apt-get```安裝相關套件：
```sh
sudo apt-get install swift swift-account swift-container swift-object
```

從物件儲存```Repository```取得accounting, container, object, container-reconciler, and object-expirer service設定檔：
```sh
# Account
sudo curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/liberty

# Container server
sudo curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/liberty

# Object
sudo curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/liberty

```
編輯```/etc/swift/account-server.conf```，在```[DEFAULT]```設定IP、Port、帳號、設定檔案目錄和掛載目錄：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_IP
bind_port = 6002
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
```
> 將```MANAGEMENT_IP```取代為主機 Management 網路介面IP

在```[pipeline:main]```部分，啟用適合的模組：
```sh
[pipeline:main]
pipeline = healthcheck recon account-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[filter:recon]```部分，設定 recon (metrics) 快取目錄：
```
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
```

編輯```/etc/swift/container-server.conf ```，在```[DEFAULT]```設定IP、Port、帳號、設定檔案目錄和掛載目錄：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6001
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
```
> 將```MANAGEMENT_INTERFACE_IP_ADDRESS```取代為主機Management網路介面IP

在```[pipeline:main]```部分，啟用適合的模組：
```
[pipeline:main]
pipeline = healthcheck recon container-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[filter:recon]```部分，設定recon (metrics)快取目錄：
```
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
```

編輯```/etc/swift/object-server.conf```，在```[DEFAULT]```部分設定IP、Port、帳號、設定檔案目錄和掛載目錄：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_IP
bind_port = 6000
user = swift
swift_dir = /etc/swift
devices = /srv/node
mount_check = true
```
> 將```MANAGEMENT_IP```取代為主機Management網路介面IP

在```[pipeline:main]```部分，啟用適合的模組：
```
[pipeline:main]
pipeline = healthcheck recon object-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[filter:recon]```部分，設定recon (metrics)快取與lock：
```
[filter:recon]
use = egg:swift#recon
...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
```

確認掛載的目錄結構是否擁有權限：
```
sudo chown -R swift:swift /srv/node
```

建立 recon目錄，並確認他有適合的權限：
```
sudo mkdir -p /var/cache/swift
sudo chown -R root:swift /var/cache/swift
```

# 建立初始化的 Rings
首先我們要回到```Controller```來執行以下動作。

### Account Ring
帳號伺服器使用帳號的ring，來維護一個容器的列表，首先透過```cd```到```/etc/swift```底下：
```sh
cd /etc/swift
```

建立一個```account.builder```檔案：
```sh
sudo swift-ring-builder account.builder create 10 3 1
```

增加每個節點到Ring中：
```sh
# Object1 sdb
sudo swift-ring-builder account.builder add  --region 1 --zone 1 --ip 10.0.0.51 --port 6002 --device sdb --weight 100

# Object2 sdb
sudo swift-ring-builder account.builder add  --region 1 --zone 2 --ip 10.0.0.52 --port 6002 --device sdb --weight 100
```

完成後，驗證內容是否正確：
```sh
sudo swift-ring-builder account.builder
```

成功會看到類似以下的資訊：
```
account.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             1       1     1       10.0.0.51  6002       10.0.0.51              6002       sdb 100.00          0 -100.00
             2       1     2       10.0.0.52  6002       10.0.0.52              6002       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```sh
sudo swift-ring-builder account.builder rebalance
```

成功會看到類似以下資訊：
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### Container ring
容器伺服器使用容器Ring來維護物件的列表。但是他不追蹤物件的位置，首先透過```cd```到```/etc/swift```底下
```sh
cd /etc/swift
```

建立檔案```container.builder```：
```sh
sudo swift-ring-builder container.builder create 10 3 1
```

增加每個節點到Ring中：
```sh
# Object1 sdb
sudo swift-ring-builder container.builder add --region 1 --zone 1 --ip 10.0.0.51 --port 6001 --device sdb --weight 100

# Object2 sdb
sudo swift-ring-builder container.builder add --region 1 --zone 2 --ip 10.0.0.52 --port 6001 --device sdb --weight 100
```

完成後，驗證內容是否正確：
```sh
sudo swift-ring-builder container.builder
```

成功會看到類似以下資訊：
```
container.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             1       1     1       10.0.0.51  6001       10.0.0.51              6001       sdb 100.00          0 -100.00
             2       1     2       10.0.0.52  6001       10.0.0.52              6001       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```sh
sudo swift-ring-builder container.builder rebalance
```

成功會看到類似以下資訊：
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### Object ring
物件伺服器使用物件Ring來維護物件在本地裝置上的位置列表，首先透過```cd```到```/etc/swift```底下：
```sh
cd /etc/swift
```

建立檔案```object.builder```：
```sh
sudo swift-ring-builder object.builder create 10 3 1
```

增加每個節點到Ring中：
```sh
# Object1 sdb
sudo swift-ring-builder object.builder add --region 1 --zone 1 --ip 10.0.0.51 --port 6000 --device sdb --weight 100

# Object2 sdb
sudo swift-ring-builder object.builder add --region 1 --zone 2 --ip 10.0.0.52 --port 6000 --device sdb --weight 100
```

完成後，驗證內容是否正確：
```sh
sudo swift-ring-builder object.builder
```

成功的話，會看到類似以下資訊：
```
object.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             1       1     1       10.0.0.51  6000       10.0.0.51              6000       sdb 100.00          0 -100.00
             2       1     2       10.0.0.52  6000       10.0.0.52              6000       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```
sudo swift-ring-builder object.builder rebalance
```

成功會看到類似以下資訊：
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```

### 分散檔案到儲存節點
將 account.ring.gz、container.ring.gz 和 object.ring.gz複製到其他儲存節點與代理服務節點上的目錄```/etc/swift```：
```sh
scp account.ring.gz container.ring.gz object.ring.gz object1:~/
scp account.ring.gz container.ring.gz object.ring.gz object2:~/
ssh object1
sudo mv ~/account.ring.gz /etc/swift/
sudo mv ~/container.ring.gz /etc/swift/
sudo mv ~/object.ring.gz /etc/swift/

ssh object2
sudo mv ~/account.ring.gz /etc/swift/
sudo mv ~/container.ring.gz /etc/swift/
sudo mv ~/object.ring.gz /etc/swift/
```

# 完成安裝
設定Hash與預設儲存策略，首先從Repository獲取設定檔```/etc/swift/swift.conf ```：
```sh
sudo curl -o /etc/swift/swift.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/liberty
```

編輯```/etc/swift/swift.conf```，在```[swift-hash]```部分設定前輟與後輟參數：
```sh
[swift-hash]
...
swift_hash_path_suffix = 1505cb4249801981da86
swift_hash_path_prefix = 42da359c6af55b2e3f7d
```
> 可透過```openssl rand -hex 10```產生。

在```[storage-policy:0]```部分，設定預設策略：
```
[storage-policy:0]
...
name = Policy-0
default = yes
```

複製```swift.conf```到每個Object節點與代理服務的額外套件的```/etc/swift```目錄：
```sh
scp /etc/swift/swift.conf object1:~/
scp /etc/swift/swift.conf object2:~/

ssh object1 sudo mv ~/swift.conf /etc/swift/
ssh object2 sudo mv ~/swift.conf /etc/swift/
```

在所有節點設定```/etc/swift```權限：
```sh
chown -R root:swift /etc/swift
ssh object1 sudo chown -R root:swift /etc/swift
ssh object2 sudo chown -R root:swift /etc/swift
```

重啟服務：
```sh
sudo service memcached restart
sudo service swift-proxy restart
```

在儲存節點上重啟服務：
```sh
ssh object1 sudo swift-init all start
ssh object2 sudo swift-init all start
```

# 驗證操作
這個部分，將描述如何在```Controller```驗證物件存儲服務的操作。
> Swift Client需要使用 -V 3參數，來使用驗證版本V3 API。
```sh
$ echo "export OS_AUTH_VERSION=3" | tee -a admin-openrc.sh demo-openrc.sh
```

首先使用```demo```來驗證：
```sh
source demo-openrc.sh
```

透過```swift -V 3 stat```來驗證：
```sh
swift -V 3 stat
```

成功會看到以下資訊：
```
Account: AUTH_aa2829b38026474ea4048d4adc807806
     Containers: 0
        Objects: 0
          Bytes: 0
X-Put-Timestamp: 1435852736.76235
     Connection: keep-alive
    X-Timestamp: 1435852736.76235
     X-Trans-Id: tx47d3a78a45fe491eafb27-0055955fc0
   Content-Type: text/plain; charset=utf-8
```
上傳一個檔案：
```sh
 swift -V 3 upload demo-container1 [FILE]
```

列出所有容器：
```sh
swift -V 3 list
```

下載一個檔案：
```sh
swift -V 3 download demo-container1 [FILE]
```

成功的話，會看到以下資訊：
```
[FILE] [auth 0.235s, headers 0.400s, total 0.420s, 0.020 MB/
```
