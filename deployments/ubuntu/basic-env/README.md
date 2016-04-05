# 基本環境
在選擇部署一個 OpenStack 之前，要先評估網路服務是基於何種方式進行部署，因為這將影響最終交付的結果。為了便於選擇，這邊簡單介紹 OpenStack 最基本的幾個網路部署模式。

Nova network 是 OpenStack 最早支援了網路部署模式，又稱 Legacy networking。可以支援單一平面的網路部署，目前支援三種網路管理：
* **Flat Network**
* **Flat DHCP Network**
* **VLAN Network**

> Nova Network 目前已不提供手動安裝教學。

而在 Neutron 網路部署中，會進一步分成以下兩種服務模式：
* **Provider networks** 的網路服務，最簡單部署是連接 Layer 2（交換器/橋接）以及網路 VLAN 切割。本質上是橋接虛擬網路的物理網路與依賴於實體網路設備（路由器）服務。

![](images/scenario-provider-networks.png)

* **Self-service networks** 的網路服務，可以讓部署多了 Layer 3（路由）服務的選擇，使用覆蓋網路分割租戶網路（Tanent Network），如 VXLAN、GRE。本質上它路由是使用 NAT 來讓虛擬網路對應到實體網路。此外夠能夠根據需求建置 Layer 7 的網路服務，如 LBaaS 與 FWaaS。

![](images/scenario-classic-networks.png)

## 主機硬體與系統設定
在安裝 OpenStack 之前，我們必須先準備好主機的基本環境設定，並透過一些驗證方式來確保後續部署的流暢與排錯。在大多數的 OpenStack 部署中，網路往往是一開始就要被規劃的，因為網路部署的方式與架構將會很大的影響 OpenStack 的擴展、彈性與高可靠等問題，這邊針對兩個網路部署的基本設定進行說明與設定：
* [Provider networks 主機環境設定](provide-networks-env.md)
* [Self-service networks 主機環境設定](self-service-network-env.md)

## 主機套件安裝與設定
OpenStack 的節點需要進行一些基本套件的安裝與設定，其中包含新增 OpenStack 所有套件的資源庫（Repository）等。其中這些環境套件的部署包含 NTP、SQL Database、Message Queue等，如以下列表：
* [基本套件安裝與設定](basic-package-env.md)
  * [MySQL High Availability 安裝（Galera）](mariadb-galera-install.md)
  * [RabbitMQ Cluster 安裝](rabbitmq-cluster-insall.md)
