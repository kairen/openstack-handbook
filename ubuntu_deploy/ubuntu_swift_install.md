# Swift 安裝與設定
本章節會說明與操作如何安裝```Object Storage```服務到OpenStack Controller節點上與Object Storage節點上，並設置相關參數與設定。若對於Swift不瞭解的人，可以參考[Swift 物件儲存套件章節](http://kairen.gitbooks.io/openstack/content/swift/index.html)

#### 架設前準備
當加入```Object storage```節點時，我們要針對[Ubuntu Neutron 多節點安裝章節](ubuntu_neutron.html)的架構來做類似實現，但這邊比較不同的是我們使用了10.0.1.x的tunnel網路，而不是10.0.2.x：
> 這邊可以選擇是否使用兩張網卡，若一張只需要設定 Managementnet

#### Object Storage Node 1
主機規格為雙核處理器，4 GB 記憶體，250 GB+ 儲存空間(sda)，500 GB+ 儲存空間(sdb)，兩張 eth 介面網卡
* **eth0 Management interface**:
    * IP address: 10.0.0.51
    * Network mask: 255.255.255.0 (or /24)
    * Default gateway: 10.0.0.1
* **eth1 Storage interface**（Option）：
    * IP address: 10.0.1.51
    * Network mask: 255.255.255.0 (or /24)
* 設定 Hostname 為```object1```
* 安裝```NTP```，並與 Controller 節點同步

#### Object Storage Node 2
主機規格為雙核處理器, 4 GB 記憶體, 500 GB+ 儲存空間(sda),250 GB+ 儲存空間(sdb), 兩張eth介面網卡
* **eth0 Management interface**:
    * IP address: 10.0.0.52
    * Network mask: 255.255.255.0 (or /24)
    * Default gateway: 10.0.0.1
* **eth1 Storage interface**（Option）：
    * IP address: 10.0.1.52
    * Network mask: 255.255.255.0 (or /24)
* 設定 Hostname 為```object2```
* 安裝```NTP```，並與Controller節點同步

設定```sudo```不需要密碼：
```sh
echo "openstack ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/openstack && sudo chmod 440 /etc/sudoers.d/openstack
```
在每個節點的```/etc/hosts```加入以下：
```sh
10.0.0.11 controller
10.0.0.21 network
10.0.0.31 compute1
10.0.0.41 block1
10.0.0.51 object1
10.0.0.52 object2
```
並更新套件，若是```Ubuntu 14.04```需更新OpenStack Repository：

```sh
sudo apt-get install software-properties-common
sudo add-apt-repository cloud-archive:liberty

sudo apt-get update && sudo apt-get -y  dist-upgrade
```

# Controller 節點安裝與設置
### 安裝前準備
在Controller上的 Swift 套件，不使用 SQL 資料庫，取而代之，我們在每個 Object storage 節點上使用分散式的 SQLite 資料庫。

首先我們要導入 Keystone 的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Swift User
openstack user create --password SWIFT_PASS --email swift@example.com swift

# 建立 Swift Role
openstack role add --project service --user swift admin

# 建立 Swift service
openstack service create --name swift  --description "OpenStack Object Storage" object-store

# 建立 Swift URL
openstack endpoint create \
--publicurl 'http://10.0.0.11:8080/v1/AUTH_%(tenant_id)s' \
--internalurl 'http://10.0.0.11:8080/v1/AUTH_%(tenant_id)s' \
--adminurl http://10.0.0.11:8080/v1 \
--region RegionOne  object-store
```
### 安裝與設置Swift套件
我們透過```apt-get```來安裝套件：
```sh
sudo apt-get install swift swift-proxy python-swiftclient python-keystoneclient python-keystonemiddleware memcached
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

# Object Storage節點安裝與設置
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

# 其他參考檔案
* [proxy-server.conf](https://launchpadlibrarian.net/190377130/proxy-server.conf)
