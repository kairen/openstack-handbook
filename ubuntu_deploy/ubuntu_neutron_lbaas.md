# LBaaS 安裝
LBaas 為 OpenStack Neutron 提供的```負載平衡即服務（Load Balance as a Service，LBaaS）```。負載平衡是分散式系統的基本元件，接收來至前端的 HTTP Request，然後將這些 Request 以某種演算法或規則來轉送給後端資源池中的某個單元來完成。

# 安裝 LBaaSv1
首先在```Controller```安裝```python-neutron-lbaas```：
```sh
$ sudo apt-get install python-neutron-lbaas -y
```

接著編輯```/etc/neutron/neutron_lbaas.conf```中的```service_provider```：
```sh
[service_providers]
service_provider = LOADBALANCER:Haproxy:neutron_lbaas.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

然後編輯```/etc/neutron/neutron.conf```中的```service_plugins```：
```sh
[DEFAULT]
...
service_plugins = neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPlugin
```
完成後，更新資料庫：
```sh
$ sudo neutron-db-manage --service lbaas upgrade head
```
之後重啓```neutron-server```服務：
```sh
$ sudo service neutron-server restart
```

在```Network```節點安裝```neutron-lbaas-agent```：
```sh
$ sudo apt-get install neutron-lbaas-agent 
```

安裝完成後，編輯```/etc/neutron/lbaas_agent.ini```，在```[DEFAULT]```加入以下：
```sh
[DEFAULT]
device_driver = neutron_lbaas.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver  
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```
在```[haproxy]```部分加入以下：
```sh
[haproxy]  
user_group = haproxy
```

完成後重啓服務：
```sh
$ sudo service neutron-lbaas-agent restart
```

# 安裝 LBaaSv2
首先在```Controller```安裝```neutron-lbaas-common```：
```sh
$ sudo apt-get install neutron-lbaasv2-common
```
接著編輯```/etc/neutron/neutron_lbaas.conf```中的```service_provider```：
```sh
service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

然後編輯```/etc/neutron/neutron.conf```中的```service_plugins```：
```sh
service_plugins = neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2, router 
```
> 如果已有定義的 plugins，可以用以下方是區分：
>
```sh
service_plugins = [already defined plugins], neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
```

更新資料庫：
```sh
$ sudo neutron-db-manage --service lbaas upgrade head
```

之後重啓```neutron-server```服務：
```sh
$ sudo service neutron-server restart
```

在```Network```節點安裝```neutron-lbaasv2-agent```：
```sh
$ sudo apt-get install neutron-lbaasv2-agent 
```
安裝完成後，編輯```/etc/neutron/lbaas_agent.ini```，在```[DEFAULT]```加入以下：
```sh
[DEFAULT]
device_driver = neutron_lbaas.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

在```[haproxy]```，加入以下：
```sh
[haproxy]  
user_group = haproxy
```

完成後重啓服務：
```sh
$ sudo service neutron-lbaasv2-agent restart
```
> Horizon 目前只支援```LBaaSV1```。

最後檢查 Dashboard 的檔案```local_settings.py ```，是否有開啟 UI：
```sh
OPENSTACK_NEUTRON_NETWORK = {
    'enable_lb': True,
    ...
}
```