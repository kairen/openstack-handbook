# 建立初始化網路
啟動第一台Instance之前，必須建立要連接的必要```虛擬網路基礎設施```，包含```外部網路```、```租戶網路```。建立好這些基礎設施之後，建議驗證連接，並在繼續執行之前解決出現問題。下圖提供一個網路基本架構預覽，網路實現了初始化網路，並展示網路如何從Instance上，傳到外部網路與內部網路。

![架構圖](images/installguide-neutron-initialnetworks.png)
> 注意！這邊架構圖的```Internet```會隨著環境而不同。

## External network
外部網路作為Instance分配網際網路的連接。該網路一般僅允許透過使用```網路位址轉換(NAT)```的Instance存取Internet。也可以透過一個浮動IP位址與合適的安全群組規則來啟用Internet的存取到個別Instance。admin租戶擁有這個網路，因為他會替多個租戶提供了外部網路存取。

#### 建立外部網路

首先回到```Controller```上，導入```admin```身份驗證來執行管理權限：
```sh
source admin-openrc.sh
```
透過```neutron net-create```指令建立網路：
```sh
neutron net-create ext-net --router:external --provider:physical_network external --provider:network_type flat
```
成功的話會看到類似以下資訊：
```sh
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 3f2d51f2-e56e-4530-8b7b-7abb35eae793 |
| mtu                       | 0                                    |
| name                      | ext-net                              |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | cf8f9b8b009b429488049bb2332dc311     |
+---------------------------+--------------------------------------+
```
如同一個物理網路，一個虛擬網路需要為其分配一個子網路。外部網路通過```Netwrok```節點上的外部介面，分享與物理網路關連的相同子網路和Gateway。需要為路由和浮動IP指定一個此子網路的獨有網段，以免與外部網路上的其他設備相衝突。

#### 建立外部子網路
我們可以透過```neutron subnet-create```建立子網路：
```sh
neutron subnet-create ext-net EXTERNAL_NETWORK_CIDR --name ext-subnet  --allocation-pool start=FLOATING_IP_START,end=FLOATING_IP_END  --disable-dhcp --gateway EXTERNAL_NETWORK_GATEWAY
```
> * 使用想用來分配給浮動IP的第一個或最後一個IP來替換```FLOATING_IP_START```與```FLOATING_IP_END```。
* 用與物理網路相關的子網路替換```EXTERNAL_NETWORK_CIDR```。
* 用與物理網路相關的Gateway替換```EXTERNAL_NETWORK_GATEWAY```。
* 關閉此子網路上的DHCP，因為Instance不直接連接到外部網路，並且需要手動分配浮動IP。

如下面範例所示：
```sh
neutron subnet-create ext-net 192.168.20.0/24 --name ext-subnet --allocation-pool start=192.168.20.101,end=192.168.20.200 --disable-dhcp --gateway 192.168.20.1
```
成功的話會看到類似以下資訊：
```
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  |{"start": "192.168.20.2", "end": "192.168.20.254"}|
| cidr              | 192.168.20.0/24                                  |
| dns_nameservers   |                                                  |
| enable_dhcp       | False                                            |
| gateway_ip        | 192.168.20.254                                   |
| host_routes       |                                                  |
| id                | b5ba2ff2-e13b-4d30-831c-55bd10a3a5e7             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | ext-subnet                                       |
| network_id        | 3f2d51f2-e56e-4530-8b7b-7abb35eae793             |
| subnetpool_id     |                                                  |
| tenant_id         | cf8f9b8b009b429488049bb2332dc311                 |
+-------------------+--------------------------------------------------+
```

## Tenant network
租戶網路為Instance提供內部網路連接。架構確保這種網路在不同租戶之間分離。透過```demo```租戶擁有這個網路，因為其僅作為對部的Instacne提供網路連接。

#### 建立租戶網路
首先回到```Controller```上，導入```demo```身份驗證來執行管理權限：
```sh
source demo-openrc.sh
```
透過```neutron net-create```建立```demo```網路：
```sh
neutron net-create demo-net
```
成功的話，會看到類似以下資訊：
```sh
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 174bee75-7a06-44f8-8d68-8722d8d6912d |
| mtu                       | 0                                    |
| name                      | demo-net                             |
| provider:network_type     | gre                                  |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | cf8f9b8b009b429488049bb2332dc311     |
+---------------------------+--------------------------------------+
```
類似於外部網路，租戶網路也需要附加子網路。可以指定任意有效的子網路，因為架構分離了租戶網路。預設情況下，這個子網路會使用DHCP，因此Instance可以獲取到IP。

#### 租戶網路建立子網路
透過```neutron subnet-create```建立子網路：
```sh
neutron subnet-create demo-net TENANT_NETWORK_CIDR  --name demo-subnet --gateway TENANT_NETWORK_GATEWAY
```
> * 將```TENANT_NETWORK_CIDR```替換為你想關聯到租戶網路的子網路。
* 將```TENANT_NETWORK_GATEWAY```替換為你想關聯的子網路Gateway。

這邊建立一個```192.168.1.0/24```的子網路：
```sh
neutron subnet-create demo-net 192.168.1.0/24  --name demo-subnet --gateway 192.168.1.1
```
成功會看到類似以下資訊：
```
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.1.2", "end": "192.168.1.254"} |
| cidr              | 192.168.1.0/24                                   |
| dns_nameservers   |                                                  |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.1.1                                      |
| host_routes       |                                                  |
| id                | 469dfc0d-80f8-4c1e-ab38-5be9ec937282             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | demo-subnet                                      |
| network_id        | 174bee75-7a06-44f8-8d68-8722d8d6912d             |
| subnetpool_id     |                                                  |
| tenant_id         | cf8f9b8b009b429488049bb2332dc311                 |
+-------------------+--------------------------------------------------+
```
虛擬路由在兩個或多個虛擬網路之間傳輸網路數據。每個路由需要一個或多個介面、子網路或者提供Gateway。如下面部分，將建立一個路由，然後將租戶網路與外部網路連接到其之上。

####  在租戶網路上，建立路由並將外部網路與租戶網路附加給它
透過```neutron router-create```指令建立路由：
```sh
neutron router-create demo-router
```
成功的話會看到以下資訊：
```sh
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | a65b2209-1473-4f4f-8a3b-a99ea9f7020b |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | cf8f9b8b009b429488049bb2332dc311     |
+-----------------------+--------------------------------------+
```
附加路由給```demo```租戶子網路：
```sh
neutron router-interface-add demo-router demo-subnet
```
會看到加入介面成功：
```sh
Added interface 276fc58e-bd54-4229-be4f-6e0abe5272f3 to router demo-router.
```
通過將路由設定為Gateway，來將路由附加給外部網路：
```sh
neutron router-gateway-set demo-router ext-net
```
輸入指令會看到以下資訊：
```sh
Set gateway for router demo-router
```

# 驗證操作
我們建議在繼續進行前，驗證網路連通性與解決其他任何問題。沿襲外部網路的子網路使用```192.168.20.0/24```的例子，租戶路由Gateway應該佔用了浮動IP位址範圍內的最小IP位址```192.168.20.101```。如果正確設定的外部物理網路與虛擬網路，應能夠從外部網路上的任何主機ping這個IP。
> 如果您在虛擬機上設定您的OpenStack節點，您必須設定管理程式以允許外部網路上的混雜模式。

### 驗證網路是否能夠連接
透過```ping```來驗證：
```sh
ping -c 4 192.168.20.1
```

### 無法解析 domain name 問題
當我們建立一個```Instance```時，已經連到Internet，但是卻無法正常上網，這邊我們可能是忘了設定DNS，可以透過```neutron subnet-update```來更新網路DNS，首先透過```neutron net-list```取得目前網路列表：
```sh
+--------------------------------------+-----------+------------------------------------------------------+
| id                                   | name      | subnets                                              |
+--------------------------------------+-----------+------------------------------------------------------+
| 62c38b7c-4ef9-4441-b25e-28e5971a3f88 | demo-net  | cb12c391-9796-49d4-b370-4c3e97333044 192.168.1.0/24  |
| 081ca7ce-2114-4523-bb13-98cc4c209336 | ext-net   | 561caa10-c7a8-4192-bb8a-f513b4e3390c 192.168.20.0/24 |
| db882c34-57c8-4e69-922d-6577a594b693 | admin-net | d4987ea4-c227-43e4-be9d-86ab4fdd0397 192.168.2.0/24  |
+--------------------------------------+-----------+------------------------------------------------------+
```
接下來我們針對想要更新dns-nameservers的網路做設定：
```sh
neutron subnet-update demo-subnet --dns_nameservers list=true 8.8.8.8 8.8.4.4 163.17.131.2
```

![Ubuntu instance](images/ubuntu_instance.png)


