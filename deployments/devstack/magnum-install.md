# DevStack Magnum 安裝
要透過 DevStack 安裝 Magnum 只需要在```localrc```設定使用 magnum 即可，首先下載 DevStack：
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

# magnum requires the following to be set correctly
PUBLIC_INTERFACE=eth1
enable_plugin magnum https://git.openstack.org/openstack/magnum

# Enable barbican service and use it to store TLS certificates
# For details http://docs.openstack.org/developer/magnum/dev/dev-tls.html
enable_plugin barbican https://git.openstack.org/openstack/barbican
VOLUME_BACKING_FILE_SIZE=20G
```
> P.S 記得更新```PUBLIC_INTERFACE```的網卡介面名稱。

若要加入 ceilometer 進行監測的話，可以加入以下到```localrc```：
```sh
enable_plugin ceilometer https://git.openstack.org/openstack/ceilometer
```

完成後執行```./stack.sh```開始進行安裝：
```sh
$ ./stack.sh
```
