# DevStack Manila 安裝
若要安裝 manila，可以依照下面方式設定```localrc```安裝，首先下載 DevStack：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
```

編輯```localrc```，並加入以下內容：
```sh
HOST_IP=localhost
enable_service horizon
disable_service n-net

enable_service neutron
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta

enable_plugin manila https://github.com/openstack/manila

Q_PLUGIN=ml2
ENABLE_TENANT_VLANS=True
ML2_VLAN_RANGES=physnet1:100:200
PHYSICAL_NETWORK=physnet1
OVS_PHYSICAL_BRIDGE=br-eth1
Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch
Q_ML2_PLUGIN_TYPE_DRIVERS=vlan,vxlan
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
