# LBaaS 安裝
LBaas 為 OpenStack Neutron 提供的```負載平衡即服務（Load Balance as a Service，LBaaS）```。負載平衡是分散式系統的基本元件，接收來至前端的 HTTP Request，然後將這些 Request 以某種演算法或規則來轉送給後端資源池中的某個單元來完成。

- [安裝 LBaaSv1](#安裝-lbaasv1)
    - [LBaaSv1 Controller 節點安裝](#lbaasv1-controller-節點安裝)
    - [LBaaSv1 Network 節點安裝](#lbaasv1-network-節點安裝)
- [安裝 LBaaSv2](#安裝-lbaasv2)
    - [LBaaSv2 Controller 節點安裝](#lbaasv2-controller-節點安裝)
    - [LBaaSv2 Network 節點安裝](#lbaasv2-network-節點安裝)

# 安裝 LBaaSv1
Neutron LBaaSv1 提供了負載平衡的服務，本節將說明如何安裝 LBaaSv1。
> 注意，目前 Liberty 版本已棄用 LBaaSv1。但依然可以安裝。

### LBaaSv1 Controller 節點安裝
首先在```Controller```節點安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install python-neutron-lbaas -y
```

完成後編輯```/etc/neutron/neutron.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
service_plugins = router,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPlugin
```

接著編輯```/etc/neutron/neutron_lbaas.conf```設定檔，在```[service_providers]```中加入以下內容：
```
[service_providers]
service_provider = LOADBALANCER:Haproxy:neutron_lbaas.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

完成後在 ```Controller```節點更新 Neutron 資料庫：
```sh
$ sudo neutron-db-manage --service lbaas upgrade head
```

之後重新啟動```neutron-server```服務：
```sh
$ sudo service neutron-server restart
```

### LBaaSv1 Network 節點安裝
在```Network```節點安裝```neutron-lbaas-agent```：
```sh
$ sudo apt-get install neutron-lbaas-agent
```

安裝完成後，編輯```/etc/neutron/lbaas_agent.ini```，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
device_driver = neutron_lbaas.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver  
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

在```[haproxy]```部分加入以下內容：
```
[haproxy]  
user_group = haproxy
```

完成後重新啟動```neutron-lbaas-agent```服務：
```sh
$ sudo service neutron-lbaas-agent restart
```

## 安裝 LBaaSv2
由於 Liberty 版本中，LBaaS 更新到了 V2 版本，若是在 Liberty 以後的版本盡可能的使用 V2 版本，本節將介紹如何安裝 LBaaSv2。

### LBaaSv2 Controller 節點安裝
首先在```Controller```節點安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install python-neutron-lbaas -y
```

安裝完成後，編輯```/etc/neutron/neutron.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
service_plugins = router, neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
```
> 如果已有定義的 plugins，可以用以下方是區分：
>
```sh
service_plugins = [already defined plugins], neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2
```

接著編輯```/etc/neutron/neutron_lbaas.conf```設定檔，在```[service_providers]```中加入以下內容：
```
[service_providers]
service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

完成後在 ```Controller```節點更新 Neutron 資料庫：
```sh
$ sudo neutron-db-manage --service lbaas upgrade head
```

之後重新啟動```neutron-server```服務：
```sh
$ sudo service neutron-server restart
```

### LBaaSv2 Network 節點安裝
到```Network```節點，安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install neutron-lbaasv2-agent
```

安裝完成後，編輯```/etc/neutron/lbaas_agent.ini```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
device_driver = neutron_lbaas.drivers.haproxy.namespace_driver.HaproxyNSDriver
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
```

在```[haproxy]```部分加入以下內容：
```
[haproxy]  
user_group = haproxy
```

接著編輯編輯```/etc/neutron/neutron_lbaas.conf```設定檔，在```[service_providers]```部分加入以下內容：
```
[service_providers]
service_provider = LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

完成後重新啟動```neutron-lbaasv2-agent```服務：
```sh
$ sudo service neutron-lbaasv2-agent restart
```
> 預設的 Horizon 目前只支援```LBaaSv1```，若要使用```LBaaSv2```則需要安裝 [neutron-lbaas-dashboard](https://github.com/openstack/neutron-lbaas-dashboard)。

最後檢查 Dashboard 的檔案```local_settings.py ```，是否有開啟 UI：
```sh
OPENSTACK_NEUTRON_NETWORK = {
    'enable_lb': True,
    ...
}
```
