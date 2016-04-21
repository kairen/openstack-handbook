# Neutron Open vSwitch GRE 部署
本節將說明如何部署基於 Neutron Open vSwitch GRE 來提供多租戶網路。

- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
    - [Controller 設定 ML2](#controller-設定-ml2)
    - [Controller 設定 Nova 使用 Network](#controller-設定-nova-使用-network)
    - [Controller 驗證服務](#controller-驗證服務)
- [Network Node](#network-node)
    - [Network 安裝前準備](#network-安裝前準備)
    - [Network 套件安裝與設定](#network-套件安裝與設定)
    - [Network 設定 ML2](#network-設定-ml2)
    - [設定 Layer 3 Proxy](#設定-layer-3-proxy)
    - [設定 DHCP Proxy](#設定-dhcp-proxy)
    - [設定 Metadata Agent](#設定-metadata-agent)
    - [設定 Open vSwitch 服務](#設定-open-vswitch-服務)
    - [Network 驗證服務](#network-驗證服務)
- [Compute Node](#compute-node)
    - [Compute 安裝前準備](#compute-安裝前準備)
    - [Compute 套件安裝與設定](#compute-套件安裝與設定)
    - [Compute 設定 ML2](#compute-設定-ml2)
    - [設定 Compute 使用 Network](#設定-compute-使用-network)
    - [Compute 驗證服務](#compute-驗證服務)

# Controller Node
在 Controller 節點上，只需要安裝 Neutron API Server 與 ML2 Plugins 即可。

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Neutron 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Neutron 資料庫：
```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'  IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%'  IDENTIFIED BY 'NEUTRON_DBPASS';
```
> 這邊```NEUTRON_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Neutron 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Neutron User
$ openstack user create --domain default --password NEUTRON_PASS --email neutron@example.com neutron

# 新增 Neutron 到 Admin Role
$ openstack role add --project service --user neutron admin

# 建立 Neutron Service
$ openstack service create --name neutron --description "OpenStack Networking" network

# 建立 Neutron public endpoints
$ openstack endpoint create --region RegionOne \
network public http://10.0.0.11:9696

# 建立 Neutron internal endpoints
$ openstack endpoint create --region RegionOne \
network internal http://10.0.0.11:9696

# 建立 Neutron admin endpoints
$ openstack endpoint create --region RegionOne \
network admin http://10.0.0.11:9696
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install neutron-server neutron-plugin-ml2 python-neutronclient
```

安裝完成後，編輯 ```/etc/neutron/neutron.conf``` 設定檔，在```[DEFAULT]```部分加入以下設定：
```sh
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
rpc_backend = rabbit
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True
```

在```[database]```部分修改使用以下方式：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@10.0.0.11/neutron
```

在```[oslo_messaging_rabbit]```部分加入以下設定：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下設定：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

在```[nova]```部分加入以下設定：
```sh
[nova]
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊```NOVA_PASS```可以隨需求修改。

### Controller 設定 ML2
這邊 Neutron ML2 外掛使用 Open vSwitch agent 來提供給虛擬機建立虛擬網路。這邊 Controller 節點不需要 Open vSwitch agent，因為 Controller 不處理網路傳輸。要設定 ML2 可以編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```並在```[ml2]```部分加入以下設定：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```
> 一旦您設定好了 ML2 插件，修改 type_drivers 選項的值會導致資料庫的不一致。所以建議一開始就設定多個。

在```[ml2_type_gre]```部分加入以下設定：
```sh
[ml2_type_gre]
tunnel_id_ranges = 1:1000
```

在```[securitygroup]```部分加入以下設定：
```sh
[securitygroup]
enable_ipset = True
```

### Controller 設定 Nova 使用 Network
當完成 ML2 設定後，接著編輯```/etc/nova/nova.conf```設定檔，在```[neutron]```部分加入以下設定：
```sh
[neutron]
url = http://10.0.0.11:9696
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

完成所有設定後，即可同步資料庫來建立 Neutron 資料表：
```sh
$ sudo neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
upgrade head
```

之後重新啟動 Nova API Server：
```sh
$ sudo service nova-api restart
```

再接著重新啟動 Neutron Server：
```sh
$ sudo service neutron-server restart
```

### Controller 驗證服務
導入 Keystone 的 ```admin``` 帳號，來透過 Neutron client 查看服務目錄：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看外部網路列表，如以下方式：
```sh
$ neutron ext-list
+---------------------------+-----------------------------------------------+
| alias                     | name                                          |
+---------------------------+-----------------------------------------------+
| dns-integration           | DNS Integration                               |
| network_availability_zone | Network Availability Zone                     |
| address-scope             | Address scope                                 |
| ext-gw-mode               | Neutron L3 Configurable external gateway mode |
| binding                   | Port Binding                                  |
| agent                     | agent                                         |
| subnet_allocation         | Subnet Allocation                             |
| l3_agent_scheduler        | L3 Agent Scheduler                            |
| external-net              | Neutron external network                      |
| net-mtu                   | Network MTU                                   |
| availability_zone         | Availability Zone                             |
| quotas                    | Quota management support                      |
| l3-ha                     | HA Router extension                           |
| provider                  | Provider Network                              |
| multi-provider            | Multi Provider Network                        |
| extraroute                | Neutron Extra Route                           |
| router                    | Neutron L3 Router                             |
| extra_dhcp_opt            | Neutron Extra DHCP opts                       |
| security-group            | security-group                                |
| dhcp_agent_scheduler      | DHCP Agent Scheduler                          |
| router_availability_zone  | Router Availability Zone                      |
| rbac-policies             | RBAC Policies                                 |
| port-security             | Port Security                                 |
| allowed-address-pairs     | Allowed Address Pairs                         |
| dvr                       | Distributed Virtual Router                    |
+---------------------------+-----------------------------------------------+
```

# Network Node
Neutron 在 Network 節點會安裝提供虛擬機的 L2 與 L3 網路虛擬化，來處理內部與外部網路的路由，還有包含 DHCP 服務、Metadata 服務。

### Network 安裝前準備
在進行安裝 Neutron 相關套件之前，必須先讓 Network 節點主機設定一些 Kernel 網路參數，透過編輯 ```/etc/sysctl.conf``` 加入以下參數：
```sh
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

完成後，可以透過 sysctl 指令來將參數載入：
```sh
$ sudo sysctl -p
```

### Network 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y neutron-plugin-ml2 neutron-openvswitch-agent \
neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent
```

安裝完成後，編輯```/etc/neutron/neutron.conf```設定檔，在```[DEFAULT]```部分加入以下設定：
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

在```[database]```部分將所有 connection 與 sqlite 的參數註解掉：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在```[oslo_messaging_rabbit]```部分加入以下設定：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下設定：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

### Network 設定 ML2
這邊 Neutron ML2 外掛使用 Open vSwitch agent 來提供給虛擬機建立虛擬網路。這邊 Network 需要設定 Open vSwitch agent，因為 Network 是主要處理網路傳輸的節點。要設定 ML2 可以編輯```/etc/neutron/plugins/ml2/ml2_conf.ini```並在```[ml2]```部分加入以下設定：
```sh
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = gre
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```

在```[ml2_type_flat]```部分加入以下設定：
```sh
[ml2_type_flat]
flat_networks = external
```

在```[ml2_type_gre]```部分加入以下設定：
```sh
[ml2_type_gre]
tunnel_id_ranges = 1:1000
```

在```[securitygroup]```部分加入以下設定：
```sh
[securitygroup]
enable_ipset = True
```

接著要設定 Open vSwitch agent，編輯```/etc/neutron/plugins/ml2/openvswitch_agent.ini```在```[ovs]```部分加入以下設定：
```sh
[ovs]
local_ip = TUNNELS_IP
bridge_mappings = external:br-ex
```
> 將```TUNNELS_IP```取代成與 Compute 節點溝通的 IP，這邊為```10.0.1.21```。

在```[agent]```部分加入以下設定：
```sh
[agent]
tunnel_types = gre
l2_population = True
prevent_arp_spoofing = True
```

在```[securitygroup]```部分加入以下設定：
```sh
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

### 設定 Layer 3 Proxy
由於本架構採用 Self-service，故需要設定 Layer 3 Proxy 來提供虛擬化路由器。編輯```/etc/neutron/l3_agent.ini```在```[DEFAULT]```部分加入以下設定:
```sh
[DEFAULT]
...
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge =
```
> external_network_bridge 參數預設保留空白，該參數是使用單一的 Agent 來提供多個外部網路，細節設定可以參考 [L3 Agent](http://docs.openstack.org/havana/config-reference/content/section_adv_cfg_l3_agent.html)。

### 設定 DHCP Proxy
由於本架構採用 Self-service，故需要設定 DHCP Proxy 來提供 DHCP 服務給虛擬機使用。編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```部分加入以下設定：
```sh
[DEFAULT]
...
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

完成上述後，下一步可以依需求設定，因為類似 GRE 的協定包含了額外的 Header 封包，這些封包增加了網路開銷，而減少了有效的封包可用空間。在不了解虛擬網路架構的情況下，Instance 會用預設的 ETH Maximum Transmission Unit（MTU）1500 bytes 來傳送封包。IP 網路利用 Path MTU discovery（PMTUD）機制來偵測與調整封包大小。但是有些作業系統、網路阻塞、缺乏對 PMTUD 支援等因素，會造成效能的損失與連接錯誤。

最好情況下，可以在 Network 節點上開啟 Jumbo frames 來避免該問題。因為 Jumbo frames 支援最大 9000 bytes 的 MTU，這大大解決了覆蓋網路（Overlay Network）的封包開銷影響。但是很多網路設備不一定支援，且 OpenStack 管理人員也可能忽略對網路架構的控制，因此考慮到複雜性，這邊選擇降低 MTU 大小來避免該問題，使用 GRE 大多環境採用 ```1454``` Bytes 來執行。

設定 MTU 可以透過編輯```/etc/neutron/dhcp_agent.ini```在```[DEFAULT]```部分加入以下設定：
```sh
[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```

然後建立並編輯```/etc/neutron/dnsmasq-neutron.conf```來設定 MTU 大小：
```sh
$ echo 'dhcp-option-force=26,1454' | sudo tee /etc/neutron/dnsmasq-neutron.conf
```

### 設定 Metadata Agent
OpenStack Metadata 提供了一些主機客製化的設定訊息，諸如 Hostname、網路配置資訊等等。編輯```/etc/neutron/metadata_agent.ini```在```[DEFAULT]```部分加入以下設定：
```sh
[DEFAULT]
nova_metadata_ip = 10.0.0.11
metadata_proxy_shared_secret = METADATA_SECRET
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

> 將其中的```METADATA_SECRET```替換為一個合適的 metadata 代理的 secret。

完成上面設定後，先回到```Controller```節點，編輯```/etc/nova/nova.conf```，在```[neutron]```部分加入以下設定：
```sh
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```
> 將其中的```METADATA_SECRET```替換為一個合適的 metadata 代理的 secret。

然後在 ```Controller``` 節點上重新啟動 Nova API Server：
```sh
$ sudo service nova-api restart
```

### 設定 Open vSwitch 服務
回到```Network```節點上，由於外部網路透過 ovs bridge 來提供，首先重新啟動 openvswitch-switch：
```sh
$ sudo service openvswitch-switch restart
```

然後新增一個外部網路橋接：
```sh
$ sudo ovs-vsctl add-br br-ex
```
> 這邊```br-ex```可自行定義。

新增外部網路橋接映射到實體網卡埠口：
```sh
$ sudo ovs-vsctl add-port br-ex INTERFACE_NAME
```
> ```INTERFACE_NAME``` 為``` Public```網路的網卡介面名稱，這邊範例為```eth1```。

根據網卡介面驅動差異，可能需要關閉 GRO（Generic Receive Offload）來實現虛擬機與外部網路的合適頻寬。在測試環境上，我們在外部網卡介面上暫時關閉 GRO：
```sh
$ sudo ethtool -K INTERFACE_NAME gro off
```
> ```INTERFACE_NAME``` 為``` Public ```網路的網卡介面名稱，這邊範例為```eth1```。

當完成上述所有安裝與設定後，回到 ```Network``` 節點重新啟動所有 Neutron agents 與 openvswitch-switch：
```sh
sudo service openvswitch-switch restart
sudo service neutron-openvswitch-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

### Network 驗證服務
接著回到```Controller```節點導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看 Agents 狀態，如以下方式：
```sh
$ neutron agent-list
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+
| id                                   | agent_type         | host     | availability_zone | alive | admin_state_up | binary                    |
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+
| 35491fa4-3e8b-4281-9e18-642d40021f40 | DHCP agent         | network1 | nova              | :-)   | True           | neutron-dhcp-agent        |
| bfc44a1b-9ba2-45ca-b4d0-9ef520fa97c0 | L3 agent           | network1 | nova              | :-)   | True           | neutron-l3-agent          |
| d2475fa3-fc6b-447b-960e-83ac28f99cc7 | Open vSwitch agent | network1 |                   | :-)   | True           | neutron-openvswitch-agent |
| f2a93b69-8a82-4f50-aa89-6f2c1a384ddc | Metadata agent     | network1 |                   | :-)   | True           | neutron-metadata-agent    |
+--------------------------------------+--------------------+----------+-------------------+-------+----------------+---------------------------+
```

# Compute Node
Neutron 在 Compute 節點主要安裝 Neutron L2 Agent 來讓虛擬機連接虛擬網路與安全群組。

### Compute 安裝前準備
在進行安裝 Neutron 相關套件之前，必須先讓 Compute 節點主機設定一些 Kernel 網路參數，透過編輯```/etc/sysctl.conf```加入以下參數：
```sh
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

完成後，可以透過 sysctl 指令來將參數載入：
```sh
$ sudo sysctl -p
```

### Compute 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install neutron-openvswitch-agent
```

安裝完成後，編輯```/etc/neutron/neutron.conf```設定檔，在```[DEFAULT]```部分加入以下設定：
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

在```[database]```部分將所有 connection 與 sqlite 的參數註解掉：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在```[oslo_messaging_rabbit]```部分加入以下設定：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下設定：
```sh
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。


### Compute 設定 ML2
這邊 Neutron ML2 外掛使用 Open vSwitch agent 來提供給虛擬機建立虛擬網路。這邊 Compute 需要設定 Open vSwitch agent，因為 Compute 會透過 Agent 連接 Network 節點來使用虛擬化網路。要設定 Open vSwitch agent，編輯```/etc/neutron/plugins/ml2/openvswitch_agent.ini```在```[ovs]```部分加入以下設定：
```sh
[ovs]
local_ip = TUNNELS_IP
```
> 將```TUNNELS_IP```取代成與 Network 節點溝通的 IP，這邊為```10.0.1.31```。

在```[agent]```部分加入以下設定：
```sh
[agent]
tunnel_types = gre
l2_population = True
prevent_arp_spoofing = True
```

在```[securitygroup]```部分加入以下設定：
```sh
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

### 設定 Compute 使用 Network
當完成 ML2 設定後，接著編輯```/etc/nova/nova.conf```，在```[neutron]```部分加入以下設定：：
```sh
[neutron]
url = http://10.0.0.11:9696
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊```NEUTRON_PASS```可以隨需求修改。

當完成上述安裝與設定後，在```Compute```節點重新啟動 Nova compute：
```sh
$ sudo service nova-compute restart
```

重新啟動 Open vSwitch Agent：
```sh
sudo service openvswitch-switch restart
sudo service neutron-openvswitch-agent restart
```

### Compute 驗證服務
接著回到```Controller```節點導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看 Agents 狀態，如以下方式：
```sh
$ neutron agent-list
```
> 若正確的話，會看到多一個```compute```節點執行了 agent

完成後，就可以建立外部網路與租戶網路（Self-service），請參閱 [建立外部網路與租戶網路](create-network.md)。
