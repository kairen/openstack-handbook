# Nova 安裝與設定
本章節會說明與操作如何安裝```Compute```服務到 OpenStack  Controller 節點上與 Compute 節點上，並設置相關參數與設定。若對於 Nova 不瞭解的人，可以參考[Nova 運算套件章節](http://kairen.gitbooks.io/openstack/content/nova/index.html)

# Controller節點安裝與設置
### 安裝前準備
我們需要在Database底下建立儲存 Nova 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立Nova資料庫與使用者：
```sql
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'  IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'  IDENTIFIED BY 'NOVA_DBPASS';
```
> 這邊若```NOVA_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Nova User
openstack user create --password NOVA_PASS --email nova@example.com nova

# 建立 Nova Role
openstack role add --project service --user nova admin

# 建立 Nova service
openstack service create --name nova --description "OpenStack Compute" compute

# 建立 Nova Endpoints
openstack endpoint create  --publicurl http://10.0.0.11:8774/v2/%\(tenant_id\)s \
--internalurl http://10.0.0.11:8774/v2/%\(tenant_id\)s \
--adminurl http://10.0.0.11:8774/v2/%\(tenant_id\)s \
--region RegionOne compute
```
### 安裝與設置Nova套件
首先我們要透過```apt-get```安裝```nova```相關套件：
```sh
sudo apt-get install nova-api nova-cert nova-conductor nova-consoleauth  nova-novncproxy nova-scheduler python-novaclient
```
安裝完成後，編輯```/etc/nova/nova.conf```，在```[DEFAULT]```部分加入以下設定，RabbitMQ存取、Keystone存取、VNC代理：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 10.0.0.11
vncserver_listen = 10.0.0.11
vncserver_proxyclient_address = 10.0.0.11
```

在```[database]```部分，設定以下：
```sh
[database]
connection = mysql://nova:NOVA_DBPASS@10.0.0.11/nova
```
> 這邊若```NOVA_DBPASS```有更改的話，請記得更改。


加入```[oslo_messaging_rabbit]```設定RabbitMQ存取：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

加入```[keystone_authtoken]```設定Keystone驗證：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊若```NOVA_PASS```有更改的話，請記得更改。

加入```[glance]```與```[oslo_concurrency]```，設定Glance Host與lock_path：
```sh
[glance]
host = 10.0.0.11

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

完成後，同步資料庫：
```sh
sudo nova-manage db sync
```
並重新啟動服務：
```sh
sudo service nova-api restart
sudo service nova-cert restart
sudo service nova-consoleauth restart
sudo service nova-scheduler restart
sudo service nova-conductor restart
sudo service nova-novncproxy restart
```
最後刪除預設的SQLite資料庫：
```sh
sudo rm -f /var/lib/nova/nova.sqlite
```

# Compute節點安裝與設置
設定好 controller 上的 ````compute service```` 之後，接著要來設定實際執行 VM instance 的 compute node。

### 安裝和設定 Compute hypervisor 套件
首先在Compute節點透過```apt-get```安裝套件：
```sh
sudo apt-get install -y nova-compute sysfsutils
```
編輯```/etc/nova/nova.conf```並完成以下操作，在```[DEFAULT]```部分加入以下設定：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
resume_guests_state_on_host_boot = true
my_ip = 10.0.0.31
```
加入```[vnc]```設定 vnc server 資訊：
```sh
[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 10.0.0.31
novncproxy_base_url = http://10.0.0.11:6080/vnc_auto.html
```

加入```[oslo_messaging_rabbit]```設定RabbitMQ存取：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

加入```[keystone_authtoken]```設定Keystone驗證：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊若```NOVA_PASS```有更改的話，請記得更改。

加入```[glance]```與```[oslo_concurrency]```，設定Glance Host與lock_path：
```sh
[glance]
host = 10.0.0.11

[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

### 完成安裝
最後我們要確認Compute是否支援```虛擬化加速```，可以透過以下指令得知：
```sh
egrep -c '(vmx|svm)' /proc/cpuinfo
```

如果得到的值大於```1```，說明您的運算節點支援硬體加速，一般不需要進行額外的配置。如果為```0```，說明節點不支援硬體加速，故需要設定```libvirt```使用```QEMU```，而不是```KVM```。編輯```/etc/nova/nova-compute.conf```的```[libvirt]```部分：
```sh
[libvirt]
...
virt_type = qemu
```
> P.S 也可以用```kvm-ok```指令來檢查。

完成後，重開服務：
```sh
sudo service nova-compute restart
```
最後刪除預設的SQLite資料庫檔案：
```sh
sudo rm -f /var/lib/nova/nova.sqlite
```

# 驗證操作
回到```Controller節點```導入Keystone的```admin```帳號，來驗證服務：
```sh
source admin-openrc.sh
```
我們可以透過```nova service-list```來看是否已啟動服務：
```sh
nova service-list
```
成功會看到類似以下資訊：
```
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-cert        | controller | internal | enabled | up    | 2015-06-24T15:08:56.000000 | -               |
| 2  | nova-consoleauth | controller | internal | enabled | up    | 2015-06-24T15:08:56.000000 | -               |
| 3  | nova-scheduler   | controller | internal | enabled | up    | 2015-06-24T15:08:57.000000 | -               |
| 4  | nova-conductor   | controller | internal | enabled | up    | 2015-06-24T15:08:58.000000 | -               |
| 5  | nova-compute     | compute1   | nova     | enabled | up    | 2015-06-24T15:08:51.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
```
透過```nova endpoints```來驗證身份驗證服務的連通性：
```sh
nova endpoints
```
成功的話，會看到類似以下資訊：
```
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | 728743bcc1e246d9a9afdfd365887db7 |
| interface | internal                         |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:5000/v2.0      |
+-----------+----------------------------------+
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | e8178feafe21403b82a27da69e2b6ee8 |
| interface | admin                            |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:35357/v2.0     |
+-----------+----------------------------------+
+-----------+----------------------------------+
| keystone  | Value                            |
+-----------+----------------------------------+
| id        | ecfc80e5b07e4032a205516a3309443a |
| interface | public                           |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:5000/v2.0      |
+-----------+----------------------------------+

+-----------+------------------------------------------------------------+
| nova      | Value                                                      |
+-----------+------------------------------------------------------------+
| id        | baa70c55076841e082693ff117f4f91e                           |
| interface | public                                                     |
| region    | RegionOne                                                  |
| region_id | RegionOne                                                  |
| url       | http://controller:8774/v2/cf8f9b8b009b429488049bb2332dc311 |
+-----------+------------------------------------------------------------+
+-----------+------------------------------------------------------------+
| nova      | Value                                                      |
+-----------+------------------------------------------------------------+
| id        | ce48bcc026d24132b4ba1dd8bab4fe7c                           |
| interface | internal                                                   |
| region    | RegionOne                                                  |
| region_id | RegionOne                                                  |
| url       | http://controller:8774/v2/cf8f9b8b009b429488049bb2332dc311 |
+-----------+------------------------------------------------------------+
+-----------+------------------------------------------------------------+
| nova      | Value                                                      |
+-----------+------------------------------------------------------------+
| id        | db3aec66b6244065a04d8b8e72b9d21e                           |
| interface | admin                                                      |
| region    | RegionOne                                                  |
| region_id | RegionOne                                                  |
| url       | http://controller:8774/v2/cf8f9b8b009b429488049bb2332dc311 |
+-----------+------------------------------------------------------------+

+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 3362e3a63bd14b5ba3ad03b927f95149 |
| interface | public                           |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:9292           |
+-----------+----------------------------------+
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | 7a05f06134464576820b9cb5b23554dc |
| interface | admin                            |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:9292           |
+-----------+----------------------------------+
+-----------+----------------------------------+
| glance    | Value                            |
+-----------+----------------------------------+
| id        | dd962c1fde8b4bc782fd197892c18d33 |
| interface | internal                         |
| region    | RegionOne                        |
| region_id | RegionOne                        |
| url       | http://controller:9292           |
+-----------+----------------------------------+
```
透過```nova image-list```來驗證```image service```連通性：
```sh
nova image-list
```
成功會看到類似以下資訊：
```
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| 638a1646-bb2e-4f61-9bb9-d5280069b4a8 | cirros-0.3.4-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```


