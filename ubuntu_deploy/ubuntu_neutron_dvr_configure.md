# Neutron DVR 配置
建置 DVR（Distributed Virtual Router）時，我們需要針對 Controller、Network、Compute 節點進行 Neutron 設定檔配置，我們將針對以下的網路架構來達到 DVR：

![DVR](images/dvr.png)
> 從以上架構圖可以看到，在 Network、Compute 節點需要安裝三張網卡，分別提供 Management Network、Tunnel Network、Exernal Network。

# Neutron Controller 配置
在 Controller 節點編輯 ```/etc/neutron/neutron.conf```，並在```[DEFAULT]```加入以下：
```sh
[DEFAULT]
...
router_distributed = True
```
完成後，編輯 ML2 Plugins 配置檔```/etc/neutron/plugins/ml2/ml2_conf.ini```，並在```[ml2]```加入以下：
```sh
[ml2]
...
mechanism_drivers = openvswitch,l2population
```
重啟 Compute 服務：
```sh
sudo service nova-api restart
```
重啟 Networking 服務：
```sh
sudo service neutron-server restart
```

# Neutron Network 節點配置
在 Network 節點編輯 ML2 Plugins 配置檔 ```/etc/neutron/plugins/ml2/ml2_conf.ini```，並在```[ml2]```加入以下：
```sh
[ml2]
mechanism_drivers = openvswitch,l2population
```
在```[agent]```加入以下：
```sh
[agent]
...
l2_population = True
enable_distributed_routing = True
arp_responder = True
```

完成後，編輯 L3 Plugins 配置檔```/etc/neutron/l3_agent.ini```，並加入以下：
```sh
agent_mode = dvr_snat
```
在```Controller```上重新啟動```Compute API```服務：
```sh
sudo service nova-api restart
```
回到```Network```節點，重啟Open Vswitch服務：
```sh
sudo service openvswitch-switch restart
```
重新啟動 Networking 服務：
```sh
sudo service neutron-plugin-openvswitch-agent restart
sudo service neutron-l3-agent restart
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
```

# Neutron Compute 節點配置
在安裝和設定 DVR 網路之前，必須設定某些核心網路參數，編輯```/etc/sysctl.conf```修改以下：
```sh
net.ipv4.ip_forward=1
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.all.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```
修改後，透過```sysctl -p```來載入：
```sh
sudo sysctl -p
```
透過```apt-get```安裝 L3 agent、Metadata agent：
```sh
sudo apt-get install -y neutron-l3-agent  neutron-metadata-agent
```

在 Compute 節點編輯 ML2 Plugins 配置檔 ```/etc/neutron/plugins/ml2/ml2_conf.ini```，並在```[ml2]```加入以下：
```sh
[ml2]
mechanism_drivers = openvswitch,l2population
```
在```[ovs]```加入以下：
```sh
[ovs]
local_ip = TUNNEL_INTERFACE_IP_ADDRESS
bridge_mappings = external:br-ex
```
在```[agent]```加入以下：
```sh
[agent]
l2_population = True
tunnel_types = gre
enable_distributed_routing = True
arp_responder = True
```
編輯 L3 Plugins 配置檔```/etc/neutron/l3_agent.ini```，並加入以下：
```sh
[DEFAULT]
...
verbose = True
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
external_network_bridge =
router_delete_namespaces = True
agent_mode = dvr
```
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
```
> 若```METADATA_SECRET```有修改，請跟著修改。

重新開啟服務：
```sh
sudo service openvswitch-switch restart
sudo service nova-compute restart
sudo service neutron-plugin-openvswitch-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

http://docs.openstack.org/networking-guide/scenario_dvr_ovs.html


