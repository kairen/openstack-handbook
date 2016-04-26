# DevStack Murano 安裝
要透過 DevStack 安裝 Murano 只需要在```localrc```設定使用 murano 即可，首先下載 DevStack：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
$ cd devstack
```

編輯```localrc```，並加入以下內容：
```sh
HOST_IP=localhost
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_TOKEN=password
SERVICE_PASSWORD=password
ADMIN_PASSWORD=password

ENABLED_SERVICES+=,q-svc,q-agt,q-dhcp,q-l3,q-meta,neutron
enable_service heat h-api h-api-cfn h-api-cw h-eng

enable_plugin murano git://git.openstack.org/openstack/murano
enable_service murano-cfapi
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
