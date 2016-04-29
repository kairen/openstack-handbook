
# DevStack DragonFlow 安裝
要透過 DevStack 安裝 DragonFlow 只需要在```local.conf```設定使用 DragonFlow 即可，首先下載 DevStack：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
```

編輯```local.conf```，並加入以下內容：
```sh
[[local|localrc]]
HOST_IP=10.0.0.191

DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_PASSWORD=password
SERVICE_TOKEN=password
ADMIN_PASSWORD=password

GIT_BASE=https://github.com:/
FLAT_INTERFACE=eth0
FLOATING_RANGE=172.16.1.0/24

Q_ENABLE_DRAGONFLOW_LOCAL_CONTROLLER=True

enable_plugin dragonflow http://git.openstack.org/openstack/dragonflow
enable_service df-etcd
enable_service df-etcd-server
enable_service df-controller
enable_service df-ext-services
enable_service df-zmq-publisher-service

disable_service n-net
enable_service q-svc
enable_service q-l3
disable_service heat
disable_service tempest

# Enable q-meta once nova is being used.
#enable_service q-meta

# We have to disable the neutron L2 agent. DF does not use the L2 agent.
disable_service q-agt

# We have to disable the neutron dhcp agent. DF does not use the dhcp agent.
disable_service q-dhcp
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
