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

# Neutron Compute 節點配置
在 Compute 節點編輯 ML2 Plugins 配置檔 ```/etc/neutron/plugins/ml2/ml2_conf.ini```，並在```[ml2]```加入以下：
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
[DEFAULT]
agent_mode = dvr
```
配置metadata


http://docs.openstack.org/networking-guide/scenario_dvr_ovs.html


