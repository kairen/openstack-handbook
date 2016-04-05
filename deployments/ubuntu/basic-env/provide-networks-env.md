# Neutron Provider networks 主機環境設定
 Provider networks 的網路服務，最簡單部署是連接 Layer 2（交換器/橋接）以及網路 VLAN 切割。本質上是橋接虛擬網路的物理網路與依賴於實體路設備（路由器）服務。Service Layout 如下圖所示：

![](images/scenario-provider-ovs-services.png)

> P.S. 若選擇部署 Provider networks 無法支援 Layer 3 虛擬化服務或更高層級的服務，如 LBaas、FWaaS。

一般 Provider network 的架構是採用物理網路基礎設施來處理流量的交換與路由，如下圖所示：

![](images/scenario-provider-general.png)

### 最小部署硬體規格分配
* **Controller Node**: 四核處理器, 8 GB 記憶體, 250 GB 儲存空間。
* **Compute Node**: 四核處理器, 8 GB 記憶體, 500 GB 儲存空間。

> 注意以上節點的 Linux OS 請採用 ```64``` 位元的版本，因為若是在 Compute 節點安裝 ```32``` 位元，在執行映像檔時，會出錯。

> 且每個節點儲存空間必須大於規格，若需要安裝其他服務，並分離儲存磁區時，請採用 ```Logical Volume Manager (LVM)```。

> 如果您選擇安裝在虛擬機上，請確認虛擬機是否允許混雜模式，並關閉 MAC 地址，在外部網絡上的過濾。


### 網路分配與說明
這邊採用 ```Neutron Provider networks``` 模式來提供 Layer 2 虛擬化網路給虛擬機使用，本架構最小安裝情況下會使用一台的 Controller 以及兩台 Compute 節點（可依服務需求增加），在不同節點上需要提供對映的多張網卡（NIC）來設定給不同網路使用：
* **Management（管理網路）**：10.0.0.0/24，需要一個 Gateway 並設定為 10.0.0.1。
> 這邊需要提供一個 Gateway 來提供給所有節點使用內部的私有網路，該網路必須能夠連接網際網路來讓主機進行套件安裝與更新等。

* **Provider（供應商網路）**：10.21.20.0/24，需要一個 Gateway 並設定為 10.21.20.254。
> Provider network 會連接到一個通用的網路物理基礎設施，來交換與路由網路流量到外部網路（typically the Interne）。

這邊假設作業系統已經安裝完成，且已正常執行系統後，我們必須在每個節點準備以下設定。首先下面指令可以設定系統的 User 執行 root 權限時不需要密碼驗證：
```sh
$ echo "ubuntu ALL = (root) NOPASSWD:ALL" \
| sudo tee /etc/sudoers.d/ubuntu && sudo chmod 440 /etc/sudoers.d/ubuntu
```

以下指令可以列出所有網路裝置：
```sh
$ dmesg | grep -i network
$ ifconfig -a
```

若要設定 ubuntu 網卡的話，可以編輯```/etc/network/interfaces```，並新增如以下內容：
```
auto lo

auto eth0
iface eth0 inet static
         address 10.0.0.11
         netmask 255.255.255.0
         gateway 10.0.0.1
         dns-nameservers 8.8.8.8
```
> 若要改網卡名稱，可以編輯```/etc/udev/rules.d/70-persistent-net.rules```。

一個簡易的設定腳本：
```sh
ID=$(ip route get 8.8.8.8 | awk '{print $7; exit}' | grep -o "[0-9]*$")
MANAGE_ETH=eth0
PUBLIC_ETH=eth1
echo "auto lo" | sudo tee /etc/network/interfaces
echo "
auto ${MANAGE_ETH}
iface ${MANAGE_ETH} inet static
         address 10.0.0.${ID}
         netmask 255.255.255.0
         gateway 10.0.0.1
         dns-nameservers 8.8.8.8
" | sudo tee -a /etc/network/interfaces

echo "
auto ${PUBLIC_ETH}
iface ${PUBLIC_ETH} inet manual
         up ip link set dev \$IFACE up
         down ip link set dev \$IFACE down
" | sudo tee -a /etc/network/interfaces
```

### Controller Node 設定
這邊將第一張網卡介面設定為 ```Management（管理網路）```：
* IP address：10.0.0.11
* Network mask：255.255.255.0 (or /24)
* Default gateway：10.0.0.1

將第二張網卡介面設定給```Provider（供應商網路）```使用。該網卡主要是提供虛擬機外部網路的流量出入口。這邊比較特殊，只需要設定手動啟動即可：
```sh
auto <INTERFACE_NAME>
iface <IINTERFACE_NAME> inet manual
         up ip link set dev $IFACE up
         down ip link set dev $IFACE down
```

完成上述後，請重新開機改變設定。

接著編輯```/etc/hostname```來改變主機名稱（Option）：
```sh
controller
```

最後並設定主機 IP 與名稱的對照，編輯```/etc/hosts```檔案加入以下內容：
```sh
10.0.0.11   controller
10.0.0.31   compute1
10.0.0.32   compute2
```
> P.S. 若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。

### Compute Node 設定
這邊將第一張網卡介面設定給```Management（管理網路）```使用：
* IP address: 10.0.0.31
* Network mask: 255.255.255.0 (or /24)
* Default gateway: 10.0.0.1

> 若有多個 Compute 節點，則 IP 也要跟著改變。

將第二張網卡介面設定給```Provider（供應商網路）```使用。該網卡主要是提供虛擬機外部網路的流量出入口。這邊比較特殊，只需要設定手動啟動即可：
```sh
auto <INTERFACE_NAME>
iface <IINTERFACE_NAME> inet manual
         up ip link set dev $IFACE up
         down ip link set dev $IFACE down
```

完成上述後，請重新開機來改變設定。

接著編輯```/etc/hostname```來改變主機名稱（Option）：
```sh
compute1
```
> 其他節點以此類推

最後並設定主機 IP 與名稱的對照，編輯```/etc/hosts```檔案加入以下內容：
```sh
10.0.0.11   controller
10.0.0.31   compute1
10.0.0.32   compute2
```
> P.S. 若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。

### 主機設定驗證
完成上述設定後，我們要透過網路工具來測試節點之間網路是否有正常連接，首先在```Controller```節點上驗證其他節點，如以下方式：
```sh
$ ping -c 2 compute1
PING compute (10.0.0.31) 56(84) bytes of data.
64 bytes from compute (10.0.0.31): icmp_seq=1 ttl=64 time=0.251 ms
64 bytes from compute (10.0.0.31): icmp_seq=2 ttl=64 time=0.253 ms

$ ping -c 2 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=44 time=16.1 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=44 time=15.6 ms
```
> 其他節點以此類推測試。
