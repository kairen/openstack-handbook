# Controller 節點安裝與設置
### 安裝前準備
設置 OpenStack 網路（Neutron）服務之前，必須建立資料庫、服務憑證和 API 端點。
我們需要在 Database 底下建立儲存 Neutron 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Neutron 資料庫與使用者：
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
openstack endpoint create \
--publicurl http://10.0.0.11:9696 \
--adminurl http://10.0.0.11:9696 \
--internalurl http://10.0.0.11:9696 \
--region RegionOne network
```
### 安裝與設置Neutron套件
透過```apt-get```來安裝套件：
```sh
sudo apt-get install neutron-server neutron-plugin-ml2 python-neutronclient
```
Networking 伺服器套件的設置包含資料庫、驗證機制、訊息佇列、拓撲變化通知和插件。編輯``` /etc/neutron/neutron.conf```，在```[DEFAULT]```部分加入以下設定：
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
nova_url = http://10.0.0.11:8774/v2
```

在```[database]```註解掉```SQLite```，並加入以下：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql://neutron:NEUTRON_DBPASS@10.0.0.11/neutron
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
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

在```[nova]```設定以下：
```sh
[nova]
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


### 設置 Modular Layer 2 (ML2) 插件
ML2 插件使用 Linux Bridge 機制(代理) 來為 Instance 建立虛擬網路架構。但是，Controller 節點不需要 Linux Bridge 套件，因為它並不處理 Instance 網路的傳輸。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```並完成以下操作，在```[ml2]```部分，新增以下內容：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
```
> 一旦您設定好了 ML2 插件，修改 type_drivers 選項的值會導致資料庫的不一致。

在```[ml2_type_vxlan]```設定 VNI 範圍：
```sh
[ml2_type_vxlan]
vni_ranges = 1:1000
```
在```[securitygroup]```設置啟用安全群組，並設置OVS iptables 防火牆驅動：
```sh
[securitygroup]
enable_ipset = True
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```
### 設定 Nova 套件使用 Networking
預設情況下，Nova  使用傳統網路（nova-network）。您必需重新配置 Nova 來透過 Networking 來管理網路。編輯```/etc/nova/nova.conf```完成以下操作，在```[DEFAULT]```部分，設置APIs和drivers：
```sh
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
```
> 預設情況下，Compute 使用內部的防火牆服務。由於 Networking 包含了一個防火牆服務，您必須使用nova.virt.firewall.NoopFirewallDriver 防火牆驅動來禁用Compute 的防火牆服務。

在```[neutron]```部分，設置存取的參數：
```sh
[neutron]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

### 完成安裝
完成後，同步資料庫：
```sh
sudo neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade liberty
```
重啟 Compute 服務：
```sh
sudo service nova-api restart
```
重啟 Networking 服務：
```sh
sudo service neutron-server restart
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
| dns-integration       | DNS Integration                               |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| agent                 | agent                                         |
| subnet_allocation     | Subnet Allocation                             |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| external-net          | Neutron external network                      |
| flavors               | Neutron Service Flavors                       |
| net-mtu               | Network MTU                                   |
| quotas                | Quota management support                      |
| l3-ha                 | HA Router extension                           |
| provider              | Provider Network                              |
| multi-provider        | Multi Provider Network                        |
| extraroute            | Neutron Extra Route                           |
| router                | Neutron L3 Router                             |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| security-group        | security-group                                |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| rbac-policies         | RBAC Policies                                 |
| port-security         | Port Security                                 |
| allowed-address-pairs | Allowed Address Pairs                         |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```

# Network 節點安裝與設置
Network 節點主要是為了處理虛擬的內部和外部網路路由及 DHCP 服務。
### 安裝前準備
在安裝和設定OpenStack網路之前，必須設定某些核心網路參數。編輯```/etc/sysctl.conf```將以下參數加入：
```sh
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```
修改後，透過```sysctl -p```來載入：
```sh
sudo sysctl -p
```
### 安裝與設定網路套件
首先透過```apt-get```安裝相關套件：
```sh
sudo apt-get install -y neutron-plugin-ml2 neutron-plugin-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```
Networking 套件的設定會包含驗證機制、訊息佇列和插件，編輯```/etc/neutron/neutron.conf```，在```[DEFAULT]```加入 RabbitMQ 存取、Keystone 存取、啟用 Modular Layer 2 插件、Router 服務、overlapping IP：
```sh
[DEFAULT]
...
verbose = True
rpc_backend = rabbit
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
```

在```[database]```註解所有```connection```參數：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在```[oslo_messaging_rabbit]```加入RabbitMQ存取：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```加入 Keystone 存取服務，若已存在的參數要記得修改：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

### 設定 Modular Layer 2 (ML2) 插件
ML2 插件使用 Linux Bridge 機制(代理) 來為 Instance 建立虛擬網路框架。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```在```[ml2]```啟用 Flat, VLAN, GRE, VXLAN 的網路類型驅動，並設定 VXLAN 與 linuxbridge driver ：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
```
在```[ml2_type_flat]```部分設定外部網路：
```sh
[ml2_type_flat]
flat_networks = external
```
在```[ml2_type_vxlan]```部分設定 VNI：
```sh
[ml2_type_vxlan]
vni_ranges = 1:1000
```
在```[securitygroup]```部分設定啟用安全群組、ipset並設置OVS iptables 防火牆驅動：
```sh
[securitygroup]
enable_ipset = True
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

接著編輯```/etc/neutron/plugins/ml2/linuxbridge_agent.ini```，在```[linux_bridge]```加入以下內容：
```sh
[linux_bridge]
physical_interface_mappings = external:<physical_interface>
```
> 這邊```<physical_interface>```為```eth1```。

在```[vxlan]```部分加入以下內容：
```sh
[vxlan]
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
enable_vxlan = true
l2_population = True
```
> 這邊```OVERLAY_INTERFACE_IP_ADDRESS```為```10.0.1.21```。

在```[agent]```部分加入以下內容：
```sh
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True
```

在```[securitygroup]```部分加入以下內容：
```sh
[securitygroup]
enable_ipset = True
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### 設定 Layer-3 (L3) proxy
Layer-3 (L3) proxy 為虛擬網路提供路由服務。編輯```/etc/neutron/l3_agent.ini```在```[DEFAULT]```部分加入以下內容:
```sh
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
verbose = True
```
> external_network_bridge 選項預留一個值，用來在單個代理上啟用多個外部網路。相關設定可以觀看 [L3 Agent](http://docs.openstack.org/havana/config-reference/content/section_adv_cfg_l3_agent.html)。

### 設定 DHCP proxy
DHCP proxy 為 Instance 提供 DHCP 服務。編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
verbose = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```

完成以上後，接下面的部分是一個選擇性設定，可依照個人需求設定。

由於類似 VXLAN 的協定包含了額外的 Header 封包，這些封包增加了網路開銷，而減少了有效的封包可用空間。在不了解虛擬網路架構的情況下，Instance會用預設的 eth maximum transmission unit(MTU) 1500 bytes來傳送封包。IP 網路利用 Path MTU discovery (PMTUD)機制來偵測與調整封包大小。但是有些作業系統或、網路阻塞、缺乏對PMTUD支援等因素，會造成效能的下效或是連接錯誤。

理想情況下，可以透過包含有租戶虛擬網路的物理網路上開啟 Jumbo frames 來避免這些問題。Jumbo frames 支援最大接近 9000 bytes 的 MTU，它可以抵消虛擬網路上 VXLAN 開銷影響。但是，很多網絡設備缺乏對於 Jumbo frames的支援，Openstack 管理員也經常缺乏對網路架構的控制。考慮到後續的複雜性，也可以選擇降低 VXLAN 開銷的 Instance MTU 大小，來避免 MTU 的問題。要設定恰當的 MTU 需要經過實驗，在大多數環境下會採用```1454```bytes來運作。

編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```部分啟用```dnsmasq```設定檔案：
```sh
[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```
建立並修改```/etc/neutron/dnsmasq-neutron.conf```啟用 DHCP MTU 選項(26) 並設定為 1450 bytes：
```sh
echo 'dhcp-option-force=26,1450' | sudo tee /etc/neutron/dnsmasq-neutron.conf
```
關閉所有存在的```dnsmasq```程式：
```sh
pkill dnsmasq
```
### 設定 Metadata agent
metadata agent 提供一些設定訊息，如 Instance 的的相關資訊。編輯```/etc/neutron/metadata_agent.ini```在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
verbose = True
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_region = RegionOne
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
nova_metadata_ip = 10.0.0.11
metadata_proxy_shared_secret = METADATA_SECRET
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

> 將其中的```METADATA_SECRET```替換為一個合適的 metadata 代理的 secret。

到```Controller```上編輯```/etc/nova/nova.conf```，在```[neutron]```部分啟用 metadata agent 並設定 secret：
```sh
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```
> 將其中的METADATA_SECRET 替換為一個合適的metadata 代理的secret。

在```Controller```上重新啟動```Compute API```服務：
```sh
sudo service nova-api restart
```

### 完成安装
重新啟動Networking服務：
```sh
sudo service neutron-plugin-linuxbridge-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

回到```Controller```節點，導入 Keystone 的```admin```帳號來驗證服務：
```sh
source admin-openrc.sh
```
列出 agent 以驗證啟動 neutron agents 是否成功：
```sh
neutron agent-list
```
成功的話，會看到以下資訊：
```
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| id                                   | agent_type         | host    | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| 54a6b2ff-5d02-4554-b3c8-9e29cd845195 | L3 agent           | network | :-)   | True           | neutron-l3-agent          |
| 590e2749-7caa-4621-9cbc-75d892df1b6b | DHCP agent         | network | :-)   | True           | neutron-dhcp-agent        |
| e7981fa5-af38-4da9-9ff7-de1f456ac720 | Linux bridge agent | network | :-)   | True           | neutron-linuxbridge-agent |
| fffc4360-2507-4fb8-af9a-2d3c13f5942d | Metadata agent     | network | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
```

# Compute 節點安裝與設置
運算節點處理 Instance 的連接和安全群組。

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
sudo sysctl -p
```

### 安裝與設定網路套件
首先透過```apt-get```安裝套件：
```sh
sudo apt-get install neutron-plugin-ml2 neutron-plugin-linuxbridge-agent
```
Networking套件的設定會包含驗證機制、訊息佇列和插件，編輯```/etc/neutron/neutron.conf```，在```[DEFAULT]```部分加入以下內容：
```sh
[DEFAULT]
...
verbose = True
rpc_backend = rabbit
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
```

在```[database]```部分註解到所有```connection```選項：
```sh
[database]
...
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在```[oslo_messaging_rabbit]```部分設定RabbitMQ：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```部分設定Keystone服務，並註解到不要部分：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。


### 設定Modular Layer 2 (ML2) 插件
ML2 插件使用 Linux Bridge 機制(代理) 來為 Instance 建立虛擬網路框架。編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```在```[ml2]```啟用 Flat, VLAN, GRE, VXLAN 的網路類型驅動，並設定 VXLAN 與 linuxbridge driver ：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security
```
在```[ml2_type_vxlan]```設定 VNI：
```sh
[ml2_type_vxlan]
vni_ranges = 1:1000
```
在```[securitygroup]```部分加入以下內容：
```sh
[securitygroup]
enable_ipset = True
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

接著編輯```/etc/neutron/plugins/ml2/linuxbridge_agent.ini```，在```[vxlan]```部分加入以下內容：
```sh
[vxlan]
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
enable_vxlan = true
l2_population = True
```
> 這邊```OVERLAY_INTERFACE_IP_ADDRESS```為```10.0.1.31```。

在```[agent]```部分加入以下內容：
```sh
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True
```

在```[securitygroup]```部分加入以下內容：
```sh
[securitygroup]
enable_ipset = True
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### 設定 Compute 使用 Networking
預設情況下，Compute 會使用傳統網絡（nova-network）。您必需重新設定 Compute 來透過 Networking 來管理網路。編輯```/etc/nova/nova.conf```在```[DEFAULT]```部分加入以下內容：
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
url = http://10.0.0.11:9696
auth_strategy = keystone
admin_auth_url = http://10.0.0.11:35357/v2.0
admin_tenant_name = service
admin_username = neutron
admin_password = NEUTRON_PASS
```
> 這邊若```NEUTRON_PASS```有更改的話，請記得更改。

### 完成安装
重啟 Compute service：
```sh
sudo service nova-compute restart
```
重啟 Linux Bridge Agent：
```sh
sudo service neutron-plugin-linuxbridge-agent restart
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
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| id                                   | agent_type         | host    | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
| 54a6b2ff-5d02-4554-b3c8-9e29cd845195 | L3 agent           | network | :-)   | True           | neutron-l3-agent          |
| 590e2749-7caa-4621-9cbc-75d892df1b6b | DHCP agent         | network | :-)   | True           | neutron-dhcp-agent        |
| 961f8c55-6ef6-4ea2-917b-148e82bc1720 | Linux bridge agent | compute | :-)   | True           | neutron-linuxbridge-agent |
| e7981fa5-af38-4da9-9ff7-de1f456ac720 | Linux bridge agent | network | :-)   | True           | neutron-linuxbridge-agent |
| fffc4360-2507-4fb8-af9a-2d3c13f5942d | Metadata agent     | network | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+---------+-------+----------------+---------------------------+
```
> 若正確的話，會輸出顯示```四個```agent運作在```網路節點```上，```一個```agent運作在```運算節點```上。
