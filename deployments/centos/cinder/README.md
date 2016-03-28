# Cinder 安裝與設定
本章節會說明與操作如何安裝```Block Storage```服務到OpenStack Controller節點上與Block Storage節點上，並設置相關參數與設定。若對於Cinder不瞭解的人，可以參考[Cinder 區塊儲存套件章節](http://kairen.gitbooks.io/openstack/content/cinder/index.html)

#### 架設前準備
當加入```Storage節點```時，我們要針對[Ubuntu Neutron 多節點安裝章節](ubuntu_neutron.html)的架構來做類似實現，但這邊比較不同的是我們使用了10.0.1.x的tunnel網路，而不是10.0.2.x：
#### Block Storage Node
*  **主機規格**：雙核處理器, 4 GB 記憶體, 500 GB+ 儲存空間(sda),250 GB+ 儲存空間(sdb), 兩張eth介面網卡
* **eth0 Management interface**:
    * IP address: 10.0.0.41
    * Network mask: 255.255.255.0 (or /24)
    * Default gateway: 10.0.0.1
* **eth1 Instance tunnel interface**：
    * IP address: 10.0.1.41
    * Network mask: 255.255.255.0 (or /24)
* 設定Hostname為```block1```
* 安裝```NTP```，並與Controller節點同步


在每個節點的```/etc/hosts```加入以下：
```sh
10.0.0.11 controller
10.0.0.21 network
10.0.0.31 compute1
10.0.0.41 block1
```
更新套件需更新OpenStack Repository：
```sh
yum install -y http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
yum install -y http://rdo.fedorapeople.org/openstack-kilo/rdo-release-kilo.rpm

yum upgrade -y && yum install -y openstack-selinux
```

# Controller節點安裝與設置
### 安裝前準備
設置OpenStack Cinder服務之前，必須建立資料庫、服務憑證和API 端點。
我們需要在Database底下建立儲存Cinder資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Cinder 資料庫與使用者：
```sql
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%'  IDENTIFIED BY 'CINDER_DBPASS';
```
> 這邊若```CINDER_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Cinder User
openstack user create --password CINDER_PASS --email cinder@example.com cinder
# 建立 Cinder Role
openstack role add --project service --user cinder admin
# 建立 Cinder service
openstack service create --name cinder  --description "OpenStack Block Storage" volume
# 建立 Cinder service v2
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
# 建立 Cinder Endpoints
openstack endpoint create  --publicurl http://controller:8776/v2/%\(tenant_id\)s  --internalurl http://controller:8776/v2/%\(tenant_id\)s --adminurl http://controller:8776/v2/%\(tenant_id\)s  --region RegionOne volume
# 建立 Cinder v2 Endpoints
openstack endpoint create --publicurl http://controller:8776/v2/%\(tenant_id\)s --internalurl http://controller:8776/v2/%\(tenant_id\)s --adminurl http://controller:8776/v2/%\(tenant_id\)s --region RegionOne volumev2
```

### 安裝與設置Cinder套件
我們透過```yum```來安裝相關套件：
```sh
yum install -y openstack-cinder python-cinderclient python-oslo-db
```

複製 ```/usr/share/cinder/cinder-dist.conf``` 檔案為 ```/etc/cinder/cinder.conf```：
```sh
cp /usr/share/cinder/cinder-dist.conf /etc/cinder/cinder.conf
chown -R cinder:cinder /etc/cinder/cinder.conf
```
接下來編輯```/etc/cinder/cinder.conf```，並在```[database]```部分設定以下：
```sh
[database]
connection = mysql://cinder:CINDER_DBPASS@controller/cinder
```
在```[DEFAULT]```部分，設定RabbitMQ存取、my ip提供管理與Keystone存取：
```sh
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 10.0.0.11
```
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
username = cinder
password = CINDER_PASS
```
> 這邊若```CINDER_PASS```有更改的話，請記得更改。

在```[oslo_concurrency]```部分，設定Lock path：
```sh
[oslo_concurrency]
lock_path = /var/lock/cinder
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成後，同步資料庫：
```sh
su -s /bin/sh -c "cinder-manage db sync" cinder
```
### 完成安裝
重新開啟服務，並設定boot啟動：
```sh
systemctl enable openstack-cinder-api.service openstack-cinder-scheduler.service
systemctl start openstack-cinder-api.service openstack-cinder-scheduler.service
```

# Storage節點安裝與設置
### 安裝前準備
首先需要配置完成本章節開頭的環境部分，當完成配置後，安裝```qemu```套件與```lvm```套件：
```sh
 yum install -y  qemu lvm2
```

設定與啟動```lvm2```服務：
```sh
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```
透過```pvcreate```指令，建立LVM Physical Volume ```/dev/sdb```：
```sh
pvcreate /dev/sdb
# 成功會看到以下資訊：
Physical volume "/dev/sdb" successfully created

pvcreate /dev/sda3
# 成功會看到以下資訊：
Physical volume "/dev/sda3" successfully created
```
> 若不知道disk資訊，可以透過```fdisk -l```查找。且有多個disk要當作storage時，也可以加入。
```sh
fdisk -l
```
若要格式化可以採用以下：
```sh
mkfs -t ext4 /dev/sdc1
```

透過```vgcreate```，建立LVM Volume 群組：
```sh
vgcreate cinder-volumes /dev/sdb /dev/sda3
# 成功會看到以下資訊：
Volume group "cinder-volumes" successfully created
```

因為只有Instance能夠存取區塊儲存Volume。但是底層的作業系統管理著這些裝置連結到Volume上。預設情況下，LVM Volume的掃描工具會掃到包含Vloume的區塊儲存裝置/dev。如果項目在Volume上使用LVM，掃描工具會檢查這些Volume，並嘗試緩存目錄，這會在底層系統與項目Volume上產生各式各樣問題，所以必須重新配置LVM，針對cinder-volume的Volume Group的設備。透過編輯```/etc/lvm/lvm.conf```完成以下設定，在```device```部分增加一個filter，只接收```/dev/sdb```：
```sh
devices {
...
filter = [ "a/.*/" ]
# 若想針對一個Disk過濾可用 filter = [ "a/sdb/", "r/.*/"]
```
> Filter參數中的元素以```a```開頭，即為accept，以```r``` 開頭，即為reject，並包括一些設備名稱的表示規則。您可以使用```vgs -vvvv```指令來測試filter。

最後透過 ```pvdisplay```查看Volume group：
```sh
pvdisplay
```
會看到類似以下資訊，因為我使採用兩顆不同硬碟組成的Volume，故會有點不一樣：
```
  --- Physical volume ---
  PV Name               /dev/sdb
  VG Name               cinder-volumes
  PV Size               232.89 GiB / not usable 3.18 MiB
  Allocatable           yes
  PE Size               4.00 MiB
  Total PE              59618
  Free PE               59362
  Allocated PE          256
  PV UUID               tsYzd6-a4s8-32iL-aYdA-AMol-qtj2-4RMhvA

```
### 安裝與設置Cinder套件
我們透過```yum```安裝相關套件：
```sh
yum install -y openstack-cinder targetcli python-oslo-db python-oslo-log MySQL-python
```
接下來編輯```/etc/cinder/cinder.conf```，並在```[database]```部分設定以下：
```sh
[database]
connection = mysql://cinder:CINDER_DBPASS@controller/cinder
```
在```[DEFAULT]```部分，設定RabbitMQ存取、my ip提供管理與Keystone存取、lvm backend：
```sh
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
enabled_backends = lvm
glance_host = controller
```
> ```MANAGEMENT_INTERFACE_IP_ADDRESS```為主機的管理網路IP位址，這邊為```10.0.0.41```

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
username = cinder
password = CINDER_PASS
```
> 這邊若```CINDER_PASS```有更改的話，請記得更改。

在```[lvm]```部分，使用LVM驅動配置後台的cinder-volumes的volume group、iSCSi協定與iSCSi服務：
```sh
[lvm]
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
iscsi_protocol = iscsi
iscsi_helper = tgtadm
```
在```[oslo_concurrency]```部分，加入lock path：
```sh
[oslo_concurrency]
lock_path = /var/lock/cinder
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
### 完成安裝
重新開啟服務，並設定boot開啟：
```sh
systemctl enable openstack-cinder-volume.service target.service
systemctl start openstack-cinder-volume.service target.service
```
### 驗證操作
首先我們要把Cinder v2的API加入到，```admin```與```demo```的環境變數檔案中：
```sh
echo "export OS_VOLUME_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.sh
```
之後導入```admin```參數，來驗證服務：
```sh
source admin-openrc.sh
```
透過```cinder service-list```列出服務套件以驗證Cinder成功啟動：
```sh
cinder service-list
```
成功的話，會看到類似以下資訊：
```
+------------------+------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |    Host    | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler | controller | nova | enabled |   up  | 2015-06-30T13:58:21.000000 |       None      |
|  cinder-volume   | block1@lvm | nova | enabled |   up  | 2015-06-30T13:58:22.000000 |       None      |
+------------------+------------+------+---------+-------+----------------------------+-----------------+
```
接下來驗證```demo```，導入環境變數檔案：
```sh
source demo-openrc.sh
```
透過```cinder create```建立1G Volume：
```sh
cinder create --name demo-volume1 1
```
成功的話，會看到類似以下資訊：
```
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
|                  name                 |             demo-volume1             |
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
驗證建立的Volume的是否有問題：
```sh
cinder list
```
會列出類似以下的列表：
```
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  |     Name     | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 0d81efd2-67f8-4abe-a9a7-ade2944168b5 | available | demo-volume1 |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```
若看到```available```代表著Volume沒問題。
> 若不是```available```，請檢查Controller節點的```/var/log/cinder```檔案的Logs。

也可以到```Dashboard```查看雲硬碟管理介面：
![Dashboard](images/dashboard_hard.png)
