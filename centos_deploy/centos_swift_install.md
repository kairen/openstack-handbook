# Swift 安裝與設定
本章節會說明與操作如何安裝```Object Storage```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Swift不瞭解的人，可以參考[Swift 物件儲存套件章節](swift.html)

#### 架設前準備
當加入```Object storage```節點時，我們要針對[Ubuntu Neutron 多節點安裝章節](ubuntu_neutron.html)的架構來做類似實現，但這邊比較不同的是我們使用了10.0.1.x的tunnel網路，而不是10.0.2.x：
#### Object Storage Node 1
* **主機規格**：雙核處理器, 4 GB 記憶體, 500 GB+ 儲存空間(sda),250 GB+ 儲存空間(sdb), 兩張eth介面網卡
* **eth0 Management interface**:
    * IP address: 10.0.0.51
    * Network mask: 255.255.255.0 (or /24)
    * Default gateway: 10.0.0.1
* **eth1 Instance tunnel interface**：
    * IP address: 10.0.1.51
    * Network mask: 255.255.255.0 (or /24)
* 設定Hostname為```object1```
* 安裝```NTP```，並與Controller節點同步
#### Object Storage Node 2
* **主機規格**：雙核處理器, 4 GB 記憶體, 500 GB+ 儲存空間(sda),250 GB+ 儲存空間(sdb), 兩張eth介面網卡
* **eth0 Management interface**:
    * IP address: 10.0.0.52
    * Network mask: 255.255.255.0 (or /24)
    * Default gateway: 10.0.0.1
* **eth1 Instance tunnel interface**：
    * IP address: 10.0.1.52
    * Network mask: 255.255.255.0 (or /24)
* 設定Hostname為```object2```
* 安裝```NTP```，並與Controller節點同步



在每個節點的```/etc/hosts```加入以下：
```sh
10.0.0.11 controller
10.0.0.21 network
10.0.0.31 compute1
10.0.0.41 block1
10.0.0.51 object1
10.0.0.52 object2
```
更新套件需更新OpenStack Repository：
```sh
yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
yum install -y http://rdo.fedorapeople.org/openstack-kilo/rdo-release-kilo.rpm

yum upgrade -y && yum install -y openstack-selinux
```
關閉每個儲存節點的防火牆：
```sh
systemctl disable firewalld.service
systemctl stop firewalld.service
```

# Controller節點安裝與設置
### 安裝前準備
在Controller上的Swift套件，不使用SQL資料庫，取而代之，我們在每個Object storage節點上使用分散式的SQLite資料庫。

對象存儲服務在控制節點上不使用SQL 數據庫。取而代之，在每個存儲節點上它使用分佈式SQLite數據庫。

首先我們要導入Keystone的```admin```帳號，來建立服務：
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
openstack endpoint create  --publicurl 'http://controller:8080/v1/AUTH_%(tenant_id)s'  --internalurl 'http://controller:8080/v1/AUTH_%(tenant_id)s'  --adminurl http://controller:8080  --region RegionOne  object-store
```
### 安裝與設置Swift套件
我們透過 ```yum``` 來安裝套件：
```sh
yum install -y openstack-swift-proxy python-swiftclient python-keystone-auth-token python-keystonemiddleware memcached
```
安裝完成後，建立```/etc/swift```，並透過網路來從物件儲存```Repository```中取得代理服務設定檔案：
```sh
curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/kilo
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
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo proxy-logging proxy-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。
在```[app:proxy-server] ```部分，啟用帳號建立：
```
[app:proxy-server]
...
account_autocreate = true
```
在```[filter:keystoneauth]```部分，啟用設定操作者角色：
```
[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = admin,user
```
在```[filter:authtoken]```部分，設定身份驗證存取，並註解掉其他不需要的：
```
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
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
...
memcache_servers = 127.0.0.1:11211
```

# Object Storage節點安裝與設置
### 安裝前準備
首先透過 ```yum``` 安裝相關套件：
```sh
yum install -y xfsprogs rsync
```
然後將```/dev/sda3```與```/dev/sdb```進行格式化：
```sh
mkfs.xfs -f /dev/sda3
mkfs.xfs -f /dev/sdb
```
建立Monut載點目錄：
```sh
mkdir -p /srv/node/sda3
mkdir -p /srv/node/sdb
```
然後編輯```/etc/fstab```檔案，加入以下：
```
/dev/sda3 /srv/node/sda3 xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
/dev/sdb /srv/node/sdb xfs noatime,nodiratime,nobarrier,logbufs=8 0 2
```
Mount裝置：
```sh
mount /srv/node/sda3
mount /srv/node/sdb
```
編輯```/etc/rsyncd.conf```，加入以下：
```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = MANAGEMENT_INTERFACE_IP_ADDRESS

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
> 將```MANAGEMENT_INTERFACE_IP_ADDRESS```取代為主機Management網路介面IP，rsync 服務不需要認證，因此可以考慮將其運作在私人網絡中。

重啟服務，並設定boot時開啟：
```sh
systemctl enable rsyncd.service
systemctl start rsyncd.service
```

### 安裝與設置Swift套件
透過 ```yum``` 安裝相關套件：
```sh
yum install -y openstack-swift-account openstack-swift-container openstack-swift-object
```
從物件儲存```Repository```取得accounting, container, object, container-reconciler, and object-expirer service設定檔：
```sh
# Account
curl -o /etc/swift/account-server.conf  https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/kilo
# Container server
curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/kilo
# Object
curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/kilo
# Container reconciler
curl -o /etc/swift/container-reconciler.conf  https://git.openstack.org/cgit/openstack/swift/plain/etc/container-reconciler.conf-sample?h=stable/kilo
# Object expirer
curl -o /etc/swift/object-expirer.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-expirer.conf-sample?h=stable/kilo
```
編輯```/etc/swift/account-server.conf```，在```[DEFAULT]```設定IP、Port、帳號、設定檔案目錄和掛載目錄：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6002
user = swift
swift_dir = /etc/swift
devices = /srv/node
```
> 將```MANAGEMENT_INTERFACE_IP_ADDRESS```取代為主機Management網路介面IP

在```[pipeline:main]```部分，啟用適合的模組：
```sh
[pipeline:main]
pipeline = healthcheck recon account-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[filter:recon]```部分，設定 recon (metrics) 快取目錄：
```
[filter:recon]
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
...
recon_cache_path = /var/cache/swift
```
編輯```/etc/swift/object-server.conf```，在```[DEFAULT]```部分設定IP、Port、帳號、設定檔案目錄和掛載目錄：
```
[DEFAULT]
...
bind_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
bind_port = 6000
user = swift
swift_dir = /etc/swift
devices = /srv/node
```
> 將```MANAGEMENT_INTERFACE_IP_ADDRESS```取代為主機Management網路介面IP

在```[pipeline:main]```部分，啟用適合的模組：
```
[pipeline:main]
pipeline = healthcheck recon object-server
```
> 其他的模組資訊，可以參考 [Deployment Guide](http://docs.openstack.org/developer/swift/deployment_guide.html)。

在```[filter:recon]```部分，設定recon (metrics)快取與lock：
```
[filter:recon]
...
recon_cache_path = /var/cache/swift
recon_lock_path = /var/lock
```
確認掛載的目錄結構是否擁有權限：
```
chown -R swift:swift /srv/node
```
建立 recon目錄，並確認他有適合的權限：
```
mkdir -p /var/cache/swift
chown -R swift:swift /var/cache/swift
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
# Object1 sda3
swift-ring-builder account.builder add  --region 1 --zone 1 --ip 10.0.0.51 --port 6002 --device sda3 --weight 100
# Object1 sdb
swift-ring-builder account.builder add  --region 1 --zone 2 --ip 10.0.0.51 --port 6002 --device sdb --weight 100
# Object2 sda3
swift-ring-builder account.builder add  --region 1 --zone 3 --ip 10.0.0.52 --port 6002 --device sda3 --weight 100
# Object1 sdb
swift-ring-builder account.builder add  --region 1 --zone 4 --ip 10.0.0.52 --port 6002 --device sdb --weight 100
```
完成後，驗證內容是否正確：
```sh
swift-ring-builder account.builder
```
成功會看到類似以下的資訊：
```
account.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6002       10.0.0.51              6002      sda6 100.00          0 -100.00
             1       1     2       10.0.0.51  6002       10.0.0.51              6002       sdb 100.00          0 -100.00
             2       1     3       10.0.0.52  6002       10.0.0.52              6002      sda6 100.00          0 -100.00
             3       1     4       10.0.0.52  6002       10.0.0.52              6002       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```sh
swift-ring-builder account.builder rebalance
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
swift-ring-builder container.builder create 10 3 1
```
增加每個節點到Ring中：
```sh
# Object1 sda3
swift-ring-builder container.builder add --region 1 --zone 1 --ip 10.0.0.51 --port 6001 --device sda3 --weight 100
# Object1 sdb
swift-ring-builder container.builder add --region 1 --zone 2 --ip 10.0.0.51 --port 6001 --device sdb --weight 100
# Object2 sda3
swift-ring-builder container.builder add --region 1 --zone 3 --ip 10.0.0.52 --port 6001 --device sda3 --weight 100
# Object2 sdb
swift-ring-builder container.builder add --region 1 --zone 4 --ip 10.0.0.52 --port 6001 --device sdb --weight 100
```
完成後，驗證內容是否正確：
```sh
swift-ring-builder container.builder
```
成功會看到類似以下資訊：
```
container.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6001       10.0.0.51              6001      sda6 100.00          0 -100.00
             1       1     2       10.0.0.51  6001       10.0.0.51              6001       sdb 100.00          0 -100.00
             2       1     3       10.0.0.52  6001       10.0.0.52              6001      sda6 100.00          0 -100.00
             3       1     4       10.0.0.52  6001       10.0.0.52              6001       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```sh
swift-ring-builder container.builder rebalance
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
swift-ring-builder object.builder create 10 3 1
```
增加每個節點到Ring中：
```sh
# Object1 sda3
swift-ring-builder object.builder add r1z1-10.0.0.51:6000/sda3 100
# Object1 sdb
swift-ring-builder object.builder add r1z2-10.0.0.51:6000/sdb 100
# Object2 sda3
swift-ring-builder object.builder add r1z3-10.0.0.52:6000/sda3 100
# Object1 sdb
swift-ring-builder object.builder add r1z4-10.0.0.52:6000/sdb 100
```
完成後，驗證內容是否正確：
```sh
swift-ring-builder object.builder
```
成功的話，會看到類似以下資訊：
```
object.builder, build version 4
1024 partitions, 3.000000 replicas, 1 regions, 4 zones, 4 devices, 100.00 balance, 0.00 dispersion
The minimum number of hours before a partition can be reassigned is 1
The overload factor is 0.00% (0.000000)
Devices:    id  region  zone      ip address  port  replication ip  replication port      name weight partitions balance meta
             0       1     1       10.0.0.51  6000       10.0.0.51              6000      sda6 100.00          0 -100.00
             1       1     2       10.0.0.51  6000       10.0.0.51              6000       sdb 100.00          0 -100.00
             2       1     3       10.0.0.52  6000       10.0.0.52              6000      sda6 100.00          0 -100.00
             3       1     4       10.0.0.52  6000       10.0.0.52              6000       sdb 100.00          0 -100.00
```
將Ring作平衡調整：
```
swift-ring-builder object.builder rebalance
```
成功會看到類似以下資訊：
```
Reassigned 1024 (100.00%) partitions. Balance is now 0.00.  Dispersion is now 0.00
```
### 分佈檔案
將 account.ring.gz、container.ring.gz 和 object.ring.gz複製到其他儲存節點與代理服務節點上的目錄```/etc/swift```：
```sh
scp account.ring.gz container.ring.gz object.ring.gz object1:/etc/swift/
scp account.ring.gz container.ring.gz object.ring.gz object2:/etc/swift/
```

# 完成安裝
設定Hash與預設儲存策略，首先從Repository獲取設定檔```/etc/swift/swift.conf ```：
```sh
curl -o /etc/swift/swift.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/kilo
```
編輯```/etc/swift/swift.conf```，在```[swift-hash]```部分設定前輟與後輟參數：
```sh
[swift-hash]
...
swift_hash_path_suffix = e7cea072c76c0fc87e41
swift_hash_path_prefix = 1ccd84fadc23e50ab53f
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
scp /etc/swift/swift.conf object1:/etc/swift/
scp /etc/swift/swift.conf object2:/etc/swift/
```
在所有節點設定```/etc/swift```權限：
```sh
chown -R swift:swift /etc/swift
ssh object1 chown -R swift:swift /etc/swift
ssh object2 chown -R swift:swift /etc/swift
```
重啟服務，並設定boot開啟：：
```sh
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service
```
在儲存節點上重啟服務，並設定boot開啟：
```sh
systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service

systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service

systemctl enable openstack-swift-container.service openstack-swift-container-auditor.service \
  openstack-swift-container-replicator.service openstack-swift-container-updater.service

systemctl start openstack-swift-container.service openstack-swift-container-auditor.service \
  openstack-swift-container-replicator.service openstack-swift-container-updater.service

systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service

systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
```

# 驗證操作
這個部分，將描述如何在```Controller```驗證物件存儲服務的操作。
> Swift Client需要使用 -V 3參數，來使用驗證版本V3 API。

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
