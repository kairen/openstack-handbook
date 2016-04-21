# DevStack Sahara 安裝
若要安裝 Sahara，可以依照下面方式設定```localrc```安裝，首先下載 DevStack：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
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
enable_service s-proxy s-object s-container s-account
enable_service tempest

SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5
SWIFT_REPLICAS=1
SWIFT_DATA_DIR=$DEST/data

enable_plugin sahara git://git.openstack.org/openstack/sahara
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
