# DevStack Sahara 安裝
若要安裝 Sahara，可以依照下面方式設定```localrc```安裝，首先下載 DevStack：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
```

編輯```localrc```，並加入以下內容：
```sh
SERVICE_TOKEN=passwd
ADMIN_PASSWORD=passwd
MYSQL_PASSWORD=passwd
RABBIT_PASSWORD=passwd
SERVICE_PASSWORD=$ADMIN_PASSWORD

ENABLED_SERVICES+=,sahara
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service neutron
enable_service s-proxy s-object s-container s-account

FIXED_RANGE=10.0.0.0/24
NETWORK_GATEWAY=10.0.0.1
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=eth0

HOST_IP=<YOUR_HOST_IP>
LOGFILE=$DEST/logs/stack.sh.log
LOGDAYS=2
SWIFT_HASH=66a3d6b56c1f479c8b4e70ab5c2000f5

SWIFT_REPLICAS=1
SWIFT_DATA_DIR=$DEST/data
enable_service tempest
enable_plugin sahara git://git.openstack.org/openstack/sahara
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
