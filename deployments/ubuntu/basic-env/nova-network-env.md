# Nova-network 基本環境
### 硬體規格分配
* **Controller Node**: 四核處理器, 8 GB 記憶體, 20 GB 儲存空間
* **Compute Node**: 四核處理器, 8 GB 記憶體, 32 GB 儲存空間

> 注意以上節點的 Linux OS 請採用```64位元```的版本，因為若是在 Compute 節點安裝```32位元```，在執行映像檔時，會出錯。

> 且每個節點儲存空間必須大於規格，若需要安裝其他服務，並分離儲存磁區時，請採用```Logical Volume Manager (LVM)```。

### 網路分配與說明
這次安裝會採用```nova-network```網路套件來網路的處理，本次架構需要一台Controller、Compute節點(若加其他則不同)，在 Controller上需要有一個管理的網路介面，然而Compute需要有一個管理與外部的網路介面，詳細分配如下：

* **管理網路**：10.0.0.0/24，有Gateway 10.0.0.1。

 > 需要提供一個Gateway來為所有節點提供內部管理存取，例如套件的安裝、安全更新、DNS 以及 NTP等。

* **外部網路**：192.168.20.0/24，有Gateway 192.168.20.1。

 > 提供一個外部網路，使Compute節點上的Instance存取網際網路，```注意！這邊會根據環境的不同而改變```。

### Controller node 設定
將第一張網卡介面設定為```管理網路```：
* IP address: 10.0.0.11
* Network mask: 255.255.255.0 (or /24)
* Default gateway: 10.0.0.1

完成後，重新開機來改變設定。

再來修改```/etc/hostname```來改變主機名稱為```controller```，並設定主機的```/etc/hosts```：
```sh
# controller
10.0.0.11   controller
# compute1
10.0.0.31   compute1
```
> 注意！若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。

### Compute node設定
將第一張網卡介面設定為```管理網路```：
* IP address: 10.0.0.31
* Network mask: 255.255.255.0 (or /24)
* Default gateway: 10.0.0.1

> 若有多個Compute node，則以10.0.0.32類推。

然後將第二張設定為```外部網路介面```，編輯```/etc/network/interfaces```更改為以下：
```
# The external network interface
auto INTERFACE_NAME
iface INTERFACE_NAME inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
```

> 若有多個Compute node，則以10.0.1.32類推。

完成後，重新開機來改變設定。

再來修改```/etc/hostname```來改變主機名稱為```compute1```，並設定主機的```/etc/hosts```：
```sh
# compute1
10.0.0.31       compute1
# controller
10.0.0.11       controller
```

> 注意！若有```127.0.1.1```存在的話，請將之註解掉，避免解析問題。


完成上述設定後，可利用```ping```指令於每個節點做網路測試。
