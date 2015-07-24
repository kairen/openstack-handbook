# Neutron 安裝與設定
本章節會說明與操作如何安裝```Networking```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Neutron不瞭解的人，可以參考[Neutron 網路套件章節](neutron.html)

### 目錄
* [Controller節點安裝與設置](#Controller節點安裝與設置)
* [Network節點安裝與設置](#Network節點安裝與設置)
* [Compute節點安裝與設置](#Compute節點安裝與設置)

# Controller節點安裝與設置
### 安裝前準備
設置OpenStack 網路(neutron) 服務之前，必須建立資料庫、服務憑證和API 端點。
我們需要在Database底下建立儲存Neutron資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立Nova資料庫與使用者：
```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'  IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%'  IDENTIFIED BY 'NEUTRON_DBPASS';
```
> 這邊若```NEUTRON_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Neutron User
openstack user create --password NEUTRON_PASS --email neutron@example.com neutron
# 建立 Neutron Role
openstack role add --project service --user neutron admin
# 建立 Neutron service
openstack service create --name neutron --description "OpenStack Networking" network
# 建立 Neutron Endpoints
openstack endpoint create  --publicurl http://controller:9696  --adminurl http://controller:9696  --internalurl http://controller:9696  --region RegionOne  network
```
### 安裝與設置Neutron套件
透過```yum```來安裝套件：
```sh
yum install -y openstack-neutron openstack-neutron-ml2 python-neutronclient which
```
Networking 伺服器套件的設置包含資料庫、驗證機制、訊息佇列、拓撲變化通知和插件。編輯``` /etc/neutron/neutron.conf```並完成以下操作，修改```[database]```註解掉```SQLite```，加入以下：
```sh
[database]
...
# connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron
```
在```[DEFAULT]```部分加入以下設定，RabbitMQ存取、Keystone存取、啟用Modular Layer 2插件、router服務、overlapping IP與網路通知運算網路拓樸變化：
```sh
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
nova_url = http://controller:8774/v2
```
加入```[oslo_messaging_rabbit]```設定RabbitMQ存取：
```sh
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

加入```[keystone_authtoken]```設定Keystone驗證：
```sh
[keystone_authtoken]
# identity_uri = http://127.0.0.1:5000
# admin_tenant_name = %SERVICE_TENANT_NAME%
# admin_user = %SERVICE_USER%
# admin_password = %SERVICE_PASSWORD%
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

在```[nova]```設置以下：
```sh
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊若```NOVA_PASS```有更改的話，請記得更改。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```

### 設置 Modular Layer 2 (ML2) 插件
ML2 插件使用Open vSwitch (OVS) 機制(代理) 來為Instance建立虛擬網路架構。但是，Controller節點不需要OVS 套件，因為它並不處理Instance網路的傳輸。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```並完成以下操作，在```[ml2]```部分，啟用flat, VLAN, generic routing encapsulation (GRE), and virtual extensible LAN (VXLAN) 的網路類型驅動，GRE租戶網絡和OVS機制驅動：
```sh
[ml2]
...
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch
```
> 一旦您設定好了ML2 插件，修改type_drivers選項的值會導致資料庫的不一致。

在```[ml2_type_gre]```設定通道id：
```sh
[ml2_type_gre]
...
tunnel_id_ranges = 1:1000
```
在```[securitygroup]```設置啟用安全群組、ipset並設置OVS iptables 防火牆驅動：
```sh
[securitygroup]
...
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```
### 設定Controller Nova 使用 Networking
預設情況下，Nova 使用傳統網路(nova-network)。您必需重新配置 Nova 來透過Networking 來管理網路。編輯```/etc/nova/nova.conf```完成以下操作，在```[DEFAULT]```部分，設置APIs和drivers：
```sh
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
> 預設情況下，Nova 使用內部的防火牆服務。由於Networking 包含了一個防火牆服務，您必須使用nova.virt.firewall.NoopFirewallDriver 防火牆驅動來禁用 Nova 的防火牆服務。

在```[neutron]```部分，設置存取的參數：
```sh
[neutron]
...
url = http://controller:9696
auth_strategy = keystone
admin_auth_url = http://controller:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

### 完成安裝
將```/etc/neutron/plugin.ini```指向 ML2 Plugin 的設定檔```/etc/neutron/plugins/ml2/ml2_conf.ini```，利用以下指令：
```sh
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
完成後，同步資料庫：
```sh
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
重啟 Compute 服務：
```sh
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service openstack-nova-conductor.service
```
重啟 Networking 服務：
```sh
systemctl enable neutron-server.service
systemctl start neutron-server.service
```
### 驗證操作
透過```neutron ext-list```以驗證是否成功啟動了一個neutron-server 背景程式：
```sh
neutron ext-list
```
成功的話，會看到類似以下資訊：
```sh
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| security-group        | security-group                                |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| net-mtu               | Network MTU                                   |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| provider              | Provider Network                              |
| agent                 | agent                                         |
| quotas                | Quota management support                      |
| subnet_allocation     | Subnet Allocation                             |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| l3-ha                 | HA Router extension                           |
| multi-provider        | Multi Provider Network                        |
| external-net          | Neutron external network                      |
| router                | Neutron L3 Router                             |
| allowed-address-pairs | Allowed Address Pairs                         |
| extraroute            | Neutron Extra Route                           |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```

# Network節點安裝與設置
Network節點主要是為了處理虛擬內部和外部網路路由及DHCP服務。
### 安裝前準備
在安裝和設定OpenStack網路之前，必須設定某些核心網路參數。編輯```/etc/sysctl.conf```將以下參數加入：
```sh
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```
修改後，透過```sysctl -p```來載入：
```sh
sysctl -p
```
### 安裝與設定網路套件
首先透過```yum```安裝相關套件：
```sh
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
```
Networking套件的設定會包含驗證機制、訊息佇列和插件，編輯```/etc/neutron/neutron.conf```在```[database]```註解所有```connection```參數：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```
在```[DEFAULT]```加入RabbitMQ存取、Keystone存取、啟用Modular Layer 2插件、router服務、overlapping IP：
```sh
[DEFAULT]
auth_strategy = keystone
rpc_backend = rabbit
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
verbose = True
```
在```[oslo_messaging_rabbit]```加入RabbitMQ存取：
```sh
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```加入Keystone存取，註解其他：
```sh
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```

### 設定Modular Layer 2 (ML2) 插件
ML2 插件使用Open vSwitch (OVS) 機制(代理) 來為Instance建立虛擬網路框架。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```在```[ml2]```啟用flat, VLAN, generic routing encapsulation (GRE), and virtual extensible LAN (VXLAN) 的網路類型驅動，GRE租戶網絡和OVS機制驅動：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch
```
在```[ml2_type_flat]```部分設定外部網路：
```sh
[ml2_type_flat]
flat_networks = external
```
在```[ml2_type_gre]```部分設定通道ID：
```sh
[ml2_type_gre]
...
tunnel_id_ranges = 1:1000
```
在```[securitygroup]```部分設定啟用安全群組、ipset並設置OVS iptables 防火牆驅動：
```sh
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```
在```[ovs]```部分啟用tunnels，設定本地tunnel 端點，並映射外部供應商網路到```br-ex```外部網路橋接上：
```sh
[ovs]
local_ip = 10.0.1.21
bridge_mappings = external:br-ex
```
> 將```INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS```取代成與Instance溝通的ip，這邊為```10.0.1.21```。

在```[agent]```部分啟用GRE通道：
```sh
[agent]
tunnel_types = gre
```
### 設定 Layer-3 (L3) proxy
Layer-3 (L3) proxy為虛擬網路提供路由服務。編輯```/etc/neutron/l3_agent.ini```在```[DEFAULT]```部分設定介面驅動、外部網路橋接和啟用刪除廢棄路由器命名空間的功能:
```sh
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge = br-ex
router_delete_namespaces = True
```
> external_network_bridge 選項預留一個值，用來在單個代理上啟用多個外部網路。相關設定可以觀看[L3 Agent](http://docs.openstack.org/havana/config-reference/content/section_adv_cfg_l3_agent.html)。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
### 設定DHCP代理
DHCP proxy為Instance提供DHCP服務。編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```設定介面與DHCP驅動，啟用刪除廢棄路由器命名空間的功能：
```sh
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
dhcp_delete_namespaces = True
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成以上後，接下面的部分是一個選擇性設定，可依照個人需求設定。

由於類似GRE的協定包含了額外的Header封包，這些封包增加了網路開銷，而減少了有效的封包可用空間。在不了解虛擬網路架構的情況下，Instance會用預設的eth maximum transmission unit(MTU) 1500 bytes來傳送封包。IP網路利用path MTU discovery (PMTUD)機制來偵測與調整封包大小。但是有些作業系統或、網路阻塞、缺乏對PMTUD支援等因素，會造成效能的下效或是連接錯誤。

理想情況下，可以透過包含有租戶虛擬網路的物理網路上開啟jumbo frames來避免這些問題。Jumbo frames 支援最大接近9000bytes的MTU，它可以抵消虛擬網路上GRE開銷影響。但是，很多網絡設備缺乏對於Jumbo frames的支援，Openstack管理員也經常缺乏對網路架構的控制。考慮到後續的複雜性，也可以選擇降低GRE開銷的Instance MTU大小，來避免MTU的問題。要設定恰當的MTU需要經過實驗，在大多數環境下會採用```1454```bytes來運作。

編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```部分啟用```dnsmasq```設定檔案：
```sh
[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```
建立並修改```/etc/neutron/dnsmasq-neutron.conf```啟用DHCP MTU選項(26) 並設定為1454 bytes：
```sh
echo 'dhcp-option-force=26,1454' | sudo tee /etc/neutron/dnsmasq-neutron.conf
```
關閉所有存在的```dnsmasq```程式：
```sh
pkill dnsmasq
```
### 設定metadata代理
metadata agent 提供一些設定訊息，如Instance的的相關資訊。編輯```/etc/neutron/metadata_agent.ini```在```[DEFAULT]```部分設定服務存取與metadata主機，註解掉不必要設定：
```sh
[DEFAULT]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
nova_metadata_ip = controller
metadata_proxy_shared_secret = METADATA_SECRET
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

> 將其中的METADATA_SECRET 替換為一個合適的metadata 代理的secret。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
到```Controlelr```上編輯```/etc/nova/nova.conf```，在```[neutron]```部分啟用metadata代理並設定secret：
```sh
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```
> 將其中的METADATA_SECRET 替換為一個合適的metadata 代理的secret。

在```Controller```上重新啟動```Compute API```服務：
```sh
systemctl restart openstack-nova-api.service
```

### 設定Open vSwitch (OVS) 服務
OVS 服務為實例提供了底層的虛擬網絡框架。整合的橋接br-int 處理內部實例網絡在OVS 中的傳輸。外部橋接br-ex 處理外部實例網絡在OVS 中的傳輸。外部橋接需要一個在物理外部網絡接口上的端口來為實例提供外部網絡的訪問。本質上，這個端口連接了您環境中虛擬的和物理的外部網絡。**(未修改)**

回到```Network```節點，重啟Open Vswitch服務：
```sh
systemctl enable openvswitch.service
systemctl start openvswitch.service
```
增加外部網路橋接：
```sh
ovs-vsctl add-br br-ex
```
增加連接到實體外部網路介面的外部橋接埠口：
```sh
ovs-vsctl add-port br-ex INTERFACE_NAME
```
> ```INTERFACE_NAME``` 為外部網路的介面名稱，這邊為eth2。

根據網路介面的驅動，可能需要禁用generic receive offload (GRO)來實現Instance和外部網路之間的合適的吞吐量。測試環境時，在外部網路介面上暫時關閉GRO：
```sh
ethtool -K INTERFACE_NAME gro off
```
> ```INTERFACE_NAME``` 為外部網路的介面名稱，這邊為eth2。

### 完成安装
將```/etc/neutron/plugin.ini```指向 ML2 Plugin設定文件```/etc/neutron/plugins/ml2/ml2_conf.ini```，如果連接不存在，可以使用以下指令：
```sh
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
由於Package問題，Open vSwitch Agent 會尋找插件的設定檔，而不是指向```/etc/neutron/plugin.ini```，用以下指令可以解決：
```sh
cp /usr/lib/systemd/system/neutron-openvswitch-agent.service  /usr/lib/systemd/system/neutron-openvswitch-agent.service.orig
sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g'  /usr/lib/systemd/system/neutron-openvswitch-agent.service
```
重新啟動Networking服務：
```sh
systemctl enable neutron-openvswitch-agent.service neutron-l3-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service neutron-ovs-cleanup.service
systemctl start neutron-openvswitch-agent.service neutron-l3-agent.service neutron-dhcp-agent.service neutron-metadata-agent.service
```
### 驗證操作
回到```Controller```節點，導入Keystone的```admin```帳號來驗證服務：
```sh
source admin-openrc.sh
```
列出代理以驗證啟動neutron代理是否成功：
```sh
neutron agent-list
```
成功的話，會看到以下資訊：
```
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| id                                   | agent_type         | host    | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| 11e817d4-e0b6-4026-8c88-3a92de2daf47 | DHCP agent         | network | :-)   | True           | neutron-dhcp-agent        |
| 1bd2d179-f120-4de8-b3c3-777e2d57d61f | Open vSwitch agent | network | :-)   | True           | neutron-openvswitch-agent |
| eea0aa74-211c-4dd2-969d-5830771426be | L3 agent           | network | :-)   | True           | neutron-l3-agent          |
| faafcb0b-717c-47e4-a238-2b0699cdbc9e | Metadata agent     | network | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
```

# Compute節點安裝與設置
運算節點處理Instance的連接和安全群組。

### 安裝前準備
在安裝和設定OpenStack網路之前，必須設定某些核心網路參數，編輯```/etc/sysctl.conf```修改以下：
```sh
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
修改後，透過```sysctl -p```來載入：
```sh
sysctl -p
```

### 安裝與設定網路套件
首先透過```yum```安裝套件：
```sh
yum install -y openstack-neutron openstack-neutron-ml2 openstack-neutron-openvswitch
```
Networking套件的設定會包含驗證機制、訊息佇列和插件，編輯```/etc/neutron/neutron.conf```在```[database]```部分註解到所有```connection```選項：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```
在```[DEFAULT]```設定RabbitMQ存取、Keystone存取、啟用Modular Layer 2插件、router服務、overlapping IP：
```sh
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
```
在```[oslo_messaging_rabbit]```部分設定RabbitMQ：
```sh
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```部分設定Keystone服務，並註解到不要部分：
```sh
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
verbose = True
```
### 設定Modular Layer 2 (ML2) 插件
ML2 插件使用Open vSwitch (OVS) 機制(代理) 來為Instance建立虛擬網路框架。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```在```[ml2]```啟用flat, VLAN, generic routing encapsulation (GRE), and virtual extensible LAN (VXLAN) 的網路類型驅動，GRE租戶網絡和OVS機制驅動：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch
```
在```[ml2_type_gre]```設定通道ID：
```sh
[ml2_type_gre]
tunnel_id_ranges = 1:1000
```
在```[securitygroup]```部分設定啟用安全群組、ipset並設置OVS iptables 防火牆驅動：
```sh
[securitygroup]
enable_security_group = True
enable_ipset = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```
在```[ovs]```部分，部分啟用tunnels，設定本地tunnel 端點：
```sh
[ovs]
local_ip = INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS
```
> 將```INSTANCE_TUNNELS_INTERFACE_IP_ADDRESS```改為Compute與Netwrok Instance溝通IP，這邊為```10.0.1.31```。

在```[agent]```部分啟用GRE通道：
```sh
[agent]
tunnel_types = gre
```
### 設定Open vSwitch (OVS) 服務
重新開啟服務：
```sh
systemctl enable openvswitch.service
systemctl start openvswitch.service
```
### 設定Compute使用 Networking
預設情況下，Compute 會使用傳統網絡(nova-network)。您必需重新設定Compute 來透過Networking來管理網路。編輯```/etc/nova/nova.conf```在```[DEFAULT]```部分設定APIs和drivers：
```sh
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
> 因為 Compute 會預設使用預設的防火牆服務，但因為在 OpenStack 的環境中，防火牆服務是設定在 Network node 上面，因此要透過設定將防火牆的功能交給 Neutron 來做。

在```[neutron]```部分設定存取參數：
```sh
[neutron]
url = http://controller:9696
auth_strategy = keystone
admin_auth_url = http://controller:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

### 完成安装
將```/etc/neutron/plugin.ini```指向 ML2 Plugin 設定檔```/etc/neutron/plugins/ml2/ml2_conf.ini```，如果連結不存在，請使用以下指令：
```sh
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
由於Package問題，Open vSwitch Agent 會尋找插件的設定檔，而不是指向```/etc/neutron/plugin.ini```，用以下指令可以解決：
```sh
cp /usr/lib/systemd/system/neutron-openvswitch-agent.service  /usr/lib/systemd/system/neutron-openvswitch-agent.service.orig
sed -i 's,plugins/openvswitch/ovs_neutron_plugin.ini,plugin.ini,g'  /usr/lib/systemd/system/neutron-openvswitch-agent.service
```
重啟 Compute service：
```sh
systemctl restart openstack-nova-compute.service
```
重啟Open vSwitch (OVS) agent：
```sh
systemctl enable neutron-openvswitch-agent.service
systemctl start neutron-openvswitch-agent.service
```
### 驗證操作
回到```Controller```節點，導入Keystone的```admin```帳號來驗證：
```sh
source admin-openrc.sh
```
列出代理以驗證啟動neutron代理是否成功：
```sh
neutron agent-list
```
成功的話，會看到以下資訊：
```
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
| id                                   | agent_type         | host     | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
| 11e817d4-e0b6-4026-8c88-3a92de2daf47 | DHCP agent         | network  | :-)   | True           | neutron-dhcp-agent        |
| 1bd2d179-f120-4de8-b3c3-777e2d57d61f | Open vSwitch agent | network  | :-)   | True           | neutron-openvswitch-agent |
| cc5d402f-e0df-420a-b95e-7dd5805cb199 | Open vSwitch agent | compute1 | :-)   | True           | neutron-openvswitch-agent |
| eea0aa74-211c-4dd2-969d-5830771426be | L3 agent           | network  | :-)   | True           | neutron-l3-agent          |
| faafcb0b-717c-47e4-a238-2b0699cdbc9e | Metadata agent     | network  | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
```
> 若正確的話，會輸出顯示```四個```agent運作在```網路節點```上，```一個```agent運作在```運算節點```上。
