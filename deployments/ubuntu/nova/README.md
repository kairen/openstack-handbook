# Nova 安裝與設定
本章節會說明與操作如何安裝運算服務到 Controller 與 Compute 節點上，並修改相關設定檔。若對於 Nova 不瞭解的人，可以參考 [Nova 運算服務章節](../../../conceptions/nova/README.md)。

- [Controller Node](#controller-node)
    - [安裝前準備](#安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [Compute Node](#compute-node)
    - [Compute 套件安裝與設定](#compute-套件安裝與設定)
- [驗證服務](#驗證服務)

# Controller Node
在 Controller 節點我們需要安裝 Nova 中的 API Server、Scheduler、noVNC Proxy 等等的服務。

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Nova 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Nova 與 Nova API 資料庫：
```sql
CREATE DATABASE nova;
CREATE DATABASE nova_api;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
```
> 這邊```NOVA_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Nova 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Nova user
$ openstack user create --domain default --password NOVA_PASS --email nova@example.com nova

# 新增 Nova 到 Admin Role
$ openstack role add --project service --user nova admin

# 建立 Nova service
$ openstack service create --name nova --description "OpenStack Compute" compute

# 建立 Nova public endpoints
$ openstack endpoint create --region RegionOne \
compute public http://10.0.0.11:8774/v2.1/%\(tenant_id\)s

# 建立 Nova internal endpoints
$ openstack endpoint create --region RegionOne \
compute internal http://10.0.0.11:8774/v2.1/%\(tenant_id\)s

# 建立 Nova admin endpoints
$ openstack endpoint create --region RegionOne \
compute admin http://10.0.0.11:8774/v2.1/%\(tenant_id\)s
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install nova-api nova-cert nova-conductor \
nova-consoleauth nova-novncproxy nova-scheduler python-novaclient
```

安裝完成後，編輯```/etc/nova/nova.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
enabled_apis = osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone

use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

my_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.11```。

在```[vnc]```部分加入以下內容：
```sh
[vnc]
vncserver_listen = 10.0.0.11
vncserver_proxyclient_address = 10.0.0.11
```

在```[database]```部分修改使用以下方式：
```sh
[database]
connection = mysql+pymysql://nova:NOVA_DBPASS@10.0.0.11/nova
```
> 這邊```NOVA_DBPASS```可以隨需求修改。

在```[api_database]```部分修改使用以下方式：
```sh
[api_database]
connection = mysql+pymysql://nova:NOVA_DBPASS@10.0.0.11/nova_api
```
> 這邊```NOVA_DBPASS```可以隨需求修改。

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
username = nova
password = NOVA_PASS
```
> 這邊```NOVA_PASS```可以隨需求修改。

在```[glance]```部分加入以下內容：
```sh
[glance]
api_servers = http://10.0.0.11:9292
```

在```[oslo_concurrency]```部分加入以下內容：
```sh
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

完成所有設定後，即可同步資料庫來建立 Nova 與 Nova API 資料表：
```sh
sudo nova-manage api_db sync
sudo nova-manage db sync
```

資料庫建立完成後，就可以重新啟動所有 Nova 服務：
```sh
sudo service nova-api restart
sudo service nova-cert restart
sudo service nova-consoleauth restart
sudo service nova-scheduler restart
sudo service nova-conductor restart
sudo service nova-novncproxy restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/nova/nova.sqlite
```

# Compute Node
安裝與設定完成 Controller 上的 Nova 所有服務後，接著要來設定實際執行虛擬機實例的 Compute 節點。該節點只會安裝一些 Linux 相關套件與 nova-compute 服務。

### Compute 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y nova-compute sysfsutils
```

安裝完成後，編輯```/etc/nova/nova.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone

use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
resume_guests_state_on_host_boot = true

my_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.31```。

在```[vnc]```部分加入以下內容：
```sh
[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = MANAGEMENT_IP
novncproxy_base_url = http://10.0.0.11:6080/vnc_auto.html
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.31```。
> 這邊```novncproxy_base_url```的 port 要隨著 proxy 節點設定改變。

在```[oslo_messaging_rabbit]```部分加入以下內容：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```設部分加入以下內容：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊```NOVA_PASS```可以隨需求修改。

在```[glance]```部分加入以下內容：
```sh
[glance]
api_servers = http://10.0.0.11:9292
```

在```[oslo_concurrency]```，部分加入以下內容：
```sh
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

最後要設定虛擬機使用的虛擬化技術，首先檢查主機是否支援```硬體加速```，利用以下方式得知：
```sh
egrep -c '(vmx|svm)' /proc/cpuinfo
```
> 或使用以下方式：
```sh
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

若取得的值 ```> 1``` 的話，表示該節點可能支援硬體加速，故使用預設的 KVM 技術。若是 ```0``` 的話，則需設定 nova-compute 使用其他虛擬化技術。要設定可以透過編輯```/etc/nova/nova-compute.conf```來修改```[libvirt]```部分：
```sh
[libvirt]
virt_type = qemu
```

完成所有安裝後，即可重新啟動服務：
```sh
$ sudo service nova-compute restart
```

最後刪除預設的 SQLite 資料庫檔案：
```sh
$ sudo rm -f /var/lib/nova/nova.sqlite
```

# 驗證服務
回到 ```Controller``` 節點導入 ```admin``` 帳號，來透過 Nova client 查看服務狀態：
```sh
$ . admin-openrc
```

這邊可以透過 Nova client 來查看服務列表，如以下方式：
```sh
$ nova service-list
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
| 1  | nova-consoleauth | controller | internal | enabled | up    | 2016-03-30T17:36:24.000000 | -               |
| 2  | nova-cert        | controller | internal | enabled | up    | 2016-03-30T17:36:24.000000 | -               |
| 3  | nova-scheduler   | controller | internal | enabled | up    | 2016-03-30T17:36:24.000000 | -               |
| 4  | nova-conductor   | controller | internal | enabled | up    | 2016-03-30T17:36:25.000000 | -               |
| 5  | nova-compute     | compute    | nova     | enabled | up    | 2016-03-30T17:36:24.000000 | -               |
+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
```

也可以透過 Nova client 來列出映像檔列表：
```sh
$ nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| 520bf946-436d-4fbd-a21b-62d5879c966e | cirros-0.3.4-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```
