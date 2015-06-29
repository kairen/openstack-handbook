# Ubuntu 15.04 or 14.04 單節點安裝
### 安裝前準備
在本次安裝環境中，我們需要三張網卡介面，實體機可以透過安裝網卡來達到，虛擬機可建立虛擬網卡來達到該效果：
* **eth0** : public network
* **eth1** : 10.21.21.0
* **eth2** : 10.21.22.0

首先我們先將user設置為不需要密碼的suoder (以openstack名稱為例)：
```sh
echo "openstack ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/openstack && sudo chmod 440 /etc/sudoers.d/openstack
```

### 套件更新
這邊我們介紹如何安裝簡單的Openstack單一節點在Ubuntu上，首先若OS為Ubuntu 14.04的話，需要先加入repository：
```sh
sudo apt-get install ubuntu-cloud-keyring
echo "deb http://ubuntu-cloud.archive.canonical.com/ubuntu trusty-updates/kilo main" |  sudo tee /etc/apt/sources.list.d/cloudarchive-kilo.list
```
更新套件後，並重新開機：
```sh
sudo apt-get update && sudo apt-get -y upgrade
sudo reboot
```

### 安裝 RaabitMQ server
首先安裝套件：
```sh
sudo apt-get install -y rabbitmq-server
```
更改RabbitMQ的guest密碼：
```sh
sudo rabbitmqctl change_password guest rabbit
```

### 安裝 MySQL server
安裝Database:
```sh
sudo apt-get install -y mysql-server python-mysqldb
```
編輯``` /etc/mysql/my.cnf```檔案，若是15.04則編輯```/etc/mysql/mysql.conf.d/mysqld.cnf```，加入一下：
```txt
bind-address = 0.0.0.0
[mysqld]
...
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```
重啟服務：
```sh
sudo service mysql restart
```
### 安裝ntp、vlan、bridge-utils
安裝網路相關套件：
```sh
sudo apt-get install -y ntp vlan bridge-utils
```
並且編輯```/etc/sysctl.conf```，修改以下：
```txt
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```
載入：
```sh
sudo sysctl -p
```

# Keystone
安裝驗證套件：
```sh
sudo apt-get install -y keystone
```
在database新增，keystone套件資料庫：
```sh
mysql -u root -p
```
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone_dbpass';

quit
```
編輯```/etc/keystone/keystone.conf```檔案，修改註解以下：
```
# connection = sqlite:////var/lib/keystone/keystone.db
```
並加入：
```sh
connection = mysql://keystone:keystone_dbpass@172.16.240.135/keystone
```
重啟服務：
```sh
sudo service keystone restart
sudo keystone-manage db_sync
```
新增keystone 驗證，首先export兩個環境變數```api url```與```service account```:
```sh
export OS_SERVICE_TOKEN=ADMIN
export OS_SERVICE_ENDPOINT=http://172.16.240.135:35357/v2.0
```
開始新增驗證角色與服務：
```sh
keystone tenant-create --name=admin --description="Admin Tenant"
keystone tenant-create --name=service --description="Service Tenant"
keystone user-create --name=admin --pass=ADMIN --email=admin@example.com
keystone role-create --name=admin
keystone user-role-add --user=admin --tenant=admin --role=admin
```
建立Keystone service與endpoint :
```sh
keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"
keystone endpoint-create --service=keystone --publicurl=http://172.16.240.135:5000/v2.0 --internalurl=http://172.16.240.135:5000/v2.0 --adminurl=http://172.16.240.135:35357/v2.0
```
完成後Unset掉環境變數：
```sh
unset OS_SERVICE_TOKEN
unset OS_SERVICE_ENDPOINT
```
建立一個檔案名為```creds``，並加入以下，我們會用來設定環境變數用：
```sh
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://172.16.240.135:35357/v2.0
```
完成後，透過```source```來建置環境變數：
```sh
source creds
```
測試KeyStone是否正常：
```sh
keystone token-get
keystone user-list
```

# Glance
首先安裝套件：
```sh
sudo apt-get install -y glance
```
一樣建立Service 資料庫:
```sh
mysql -u root -p
```
```sql
CREATE DATABASE glance;
GRANT ALL ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance_dbpass';

quit;
```
建立KeyStone的Glance entries:
```sh
keystone user-create --name=glance --pass=glance_pass --email=glance@example.com
keystone user-role-add --user=glance --tenant=service --role=admin
keystone service-create --name=glance --type=image --description="Glance Image Service"
keystone endpoint-create --service=glance --publicurl=http://172.16.240.135:9292 --internalurl=http://172.16.240.135:9292 --adminurl=http://172.16.240.135:9292
```
編輯```/etc/glance/glance-api.conf```，並修改一下：
```txt
[DEFAULT]
rabbit_password = rabbit
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql://glance:glance_dbpass@172.16.240.135/glance

[keystone_authtoken]
identity_uri = http://172.16.240.135:35357
admin_tenant_name = service
admin_user = glance
admin_password = glance_pass

[paste_deploy]
flavor = keystone
```
完成後，編輯```/etc/glance/glance-registry.conf```，並修改一下：
```txt
rabbit_password = rabbit
# sqlite_db = /var/lib/glance/glance.sqlite
connection = mysql://glance:glance_dbpass@172.16.240.135/glance

[keystone_authtoken]
identity_uri = http://172.16.240.135:35357
admin_tenant_name = service
admin_user = glance
admin_password = glance_pass

[paste_deploy]
flavor = keystone
```
重啟服務：
```sh
sudo service glance-api restart
sudo service glance-registry restart
```
同步資料庫：
```sh
sudo glance-manage db_sync
````
下載一個映像檔 Example```Cirros```:
```sh
glance image-create --name Cirros --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
glance image-list
```
> Cirros 是OpenStack的小型的測試OS。

# Nova
安裝套件：
```sh
sudo apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient nova-compute nova-console
```
建立Nova資料庫：
```sh
mysql -u root -p
```
```sql
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova_dbpass';

quit
```
建立Nova的KeyStone entries:
```sh
keystone user-create --name=nova --pass=nova_pass --email=nova@example.com
keystone user-role-add --user=nova --tenant=service --role=admin
keystone service-create --name=nova --type=compute --description="OpenStack Compute"
keystone endpoint-create --service=nova --publicurl=http://172.16.240.135:8774/v2/%\(tenant_id\)s --internalurl=http://172.16.240.135:8774/v2/%\(tenant_id\)s --adminurl=http://172.16.240.135:8774/v2/%\(tenant_id\)s
```
編輯```/etc/nova/nova.conf```，並修改一下：
```txt
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 172.16.240.135
vnc_enabled = True
vncserver_listen = 172.16.240.135
vncserver_proxyclient_address = 172.16.240.135
novncproxy_base_url = http://172.16.240.135:6080/vnc_auto.html

network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

scheduler_default_filters=AllHostsFilter

[database]
connection = mysql://nova:nova_dbpass@172.16.240.135/nova

[oslo_messaging_rabbit]
rabbit_host = 127.0.0.1
rabbit_password = rabbit

[keystone_authtoken]
auth_uri = http://172.16.240.135:5000
auth_url = http://172.16.240.135:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = nova_pass

[glance]
host = 172.16.240.135

[oslo_concurrency]
lock_path = /var/lock/nova

[neutron]
service_metadata_proxy = True
metadata_proxy_shared_secret = openstack
url = http://172.16.240.135:9696
auth_strategy = keystone
admin_auth_url = http://172.16.240.135:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = neutron_pass
```
同步資料庫：
```sh
sudo nova-manage db sync
```
重新開啟相關服務：
```sh
sudo service nova-api restart
sudo service nova-cert restart
sudo service nova-consoleauth restart
sudo service nova-scheduler restart
sudo service nova-conductor restart
sudo service nova-novncproxy restart
sudo service nova-compute restart
sudo service nova-console restart
```
測試Nova是否正常：
```sh
nova-manage service list
nova list
```

# Neutron
安裝套件：
```sh
sudo apt-get install -y neutron-server neutron-plugin-openvswitch neutron-plugin-openvswitch-agent neutron-common neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent openvswitch-switch
```
建立 Neutron 資料庫：
```sh
mysql -u root -p
```
```sql
CREATE DATABASE neutron;
GRANT ALL ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron_dbpass';

quit;
```
建立 Neutron 的KeyStone entries:
```sh
keystone user-create --name=neutron --pass=neutron_pass --email=neutron@example.com
keystone service-create --name=neutron --type=network --description="OpenStack Networking"
keystone user-role-add --user=neutron --tenant=service --role=admin
keystone endpoint-create --service=neutron --publicurl http://172.16.240.135:9696 --adminurl http:/172.16.240.135:9696  --internalurl http://172.16.240.135:9696
```
編輯```/etc/neutron/neutron.conf```，並修改一下：
```txt
[DEFAULT]
......
verbose = True
debug = True
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://172.16.240.135:8774/v2
nova_region_name = regionOne
nova_admin_username = nova
nova_admin_tenant_id = 2cd03b576bcd44599e4fdcd15453b6f0
nova_admin_tenant_name = service
nova_admin_password = nova_pass
nova_admin_auth_url = http://172.16.240.135:35357/v2.0
notification_driver=neutron.openstack.common.notifier.rpc_notifier
rpc_backend=rabbit

[agent]
......
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
# ===========  end of items for agent management extension =====


[keystone_authtoken]
auth_uri = http://172.16.240.135:35357/v2.0/
auth_url = http://172.16.240.135:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = neutron_pass

[database]
......
connection = mysql://neutron:neutron_dbpass@172.16.240.135/neutron

[nova]
......
auth_url = http://172.16.240.135:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = regionOne
project_name = service
username = nova
password = nova_pass

[oslo_concurrency]
......
lock_path = /var/lock/neutron/

[oslo_messaging_rabbit]
......
rabbit_host = localhost
rabbit_userid = guest
rabbit_password = rabbit
rabbit_virtual_host = /
```
編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```，並修改以下：
```txt
[ml2]
type_drivers=flat,vlan
tenant_network_types=vlan,flat
mechanism_drivers=openvswitch
[ml2_type_flat]
flat_networks=External
[ml2_type_vlan]
network_vlan_ranges=Intnet1:100:200
[ml2_type_gre]
[ml2_type_vxlan]
[securitygroup]
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group=True
[ovs]
bridge_mappings=External:br-ex,Intnet1:br-eth1
```
透過vSwitch建立虛擬網卡橋接介面：
```sh
sudo ovs-vsctl add-br br-int
sudo ovs-vsctl add-br br-eth1
sudo ovs-vsctl add-br br-ex
sudo ovs-vsctl add-port br-eth1 eth1
sudo ovs-vsctl add-port br-ex eth2
```

> 可以參考以下連結：
https://fosskb.wordpress.com/2014/06/10/managing-openstack-internaldataexternal-network-in-one-interface

完成後，編輯 ```/etc/neutron/metadata_agent.ini```，並修改一下：
```txt
[DEFAULT]
auth_url = http://172.16.240.135:5000/v2.0
auth_region = RegionOne
admin_tenant_name = service
admin_user = neutron
admin_password = neutron_pass
metadata_proxy_shared_secret = openstack
```
編輯Neutron dhcp ```/etc/neutron/dhcp_agent.ini```，並修改如下：
```txt
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
```
編輯Neutron L3 Agent``` /etc/neutron/l3_agent.ini```，並修改一下：
```txt
[DEFAULT]
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
```
同步資料庫：
```sh
sudo neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade kilo
```
重啟服務：
```sh
sudo service neutron-server restart
sudo service neutron-plugin-openvswitch-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-l3-agent restart
```
檢查neutron：
```sh
neutron agent-list
```
> * [L2 connectivity](https://fosskb.wordpress.com/2014/06/19/l2-connectivity-in-openstack-using-openvswitch-mechanism-driver/)
* [L3 connectivity](https://fosskb.wordpress.com/2014/09/15/l3-connectivity-using-neutron-l3-agent/)
* [A bite of virtual linux networking](https://fosskb.wordpress.com/2014/06/25/a-bite-of-virtual-linux-networking/)

# Horizon
安裝Ubuntu的Horizon：
```sh
sudo apt-get install -y openstack-dashboard
```
完成後，可以到Browser登入[Horizon Dashboard](http://172.16.240.135/horizon)。

> * **Username**: admin
> * **Password**: ADMIN

# 參考
* [OpenStack 單節點安裝](https://fosskb.wordpress.com/2015/04/18/installing-openstack-kilo-on-ubuntu-15-04-single-machine-setup/)
* [OpenStack 官方安裝文件](http://docs.openstack.org/kilo/install-guide/install/apt/content/ch_overview.html)
* [OpenStack 大陸文件](http://www.aboutyun.com/thread-13063-1-1.html)
* [OpenStack Youtube install](https://www.youtube.com/watch?v=8xx_qwaCWFg)
