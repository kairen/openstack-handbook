# OpenStack Ubuntu Manual 安裝
當您已完成單節點或是 DevStack 後，想當然會進一步嘗試架設多台的 OpenStack 叢集，這邊安裝提供了基於 OpenStack Neutron 的網路虛擬化部署模式，該架構需利用到三個節點進行最小架構安裝，當然這邊可以依需求增加節點來安裝諸如 Cinder 區塊儲存、Swift 物件儲存、Manila 共享式檔案系統等其他服務。以下為使用到節點角色介紹：
* **Controller Node**：OpenStack 叢集中的主要控制者，會透過 Message Queue 系統來與其他節點進行溝通，該節點也會安裝 MySQL
* **Network Node**：
* **Compute node**：

以下節點屬於非核心套件的服務，可以依需求加入：
* **Block Storage Node**：
* **Object Storage Node**：

* **Controller Node** : 會運作```KeyStone```、```Glance```、管理運算的部分和網路服務，並運作```網路plugin```以及```Horizon儀表板```。還包括一些支援的服務，例如```SQL資料庫```、```訊息佇列(message queue)```和```網路時間協定(NTP)```。
* **Network Node** : 運作Networking plugin和一些租戶(Tenant)網路提供的代理，並提供```switching```、```routing```、```NAT```和```DHCP```服務。這個節點也處理租戶虛擬機實例的```外部網路(Internet)```連接。
* **Compute node** : 運作Compute的```hypervisor```部分，這個部分將操作tenant virtual machines或實例。預設情況下，Compute使用```KVM```作為hypervisor。運算節點也可以運行Networking plugin和代理，它們連接租戶網路到虛擬機上，並提供防火牆( security groups )服務。

 > 可以運作不止一台運算節點。

* **Block Storage Node** :  區塊儲存節點包含磁碟，區塊儲存服務會向租戶虛擬機實例提供這些磁碟。

 > 可以運作多個區塊儲存節點，來擴展Volume。

* **Object Storage Node** : 物件儲存節點包含磁碟，物件儲存服務使用這些磁碟來儲存```帳戶```、```容器```和```物件```。

 > 可以運行多個物件儲存節點。但是在最小架構範例中，需要兩個節點。

## 最小架構環境
下圖包含OpenStack 網路(neutron) 的最小架構範例的硬體需求：
![硬體需求架構圖](images/installguidearch-neutron-hw.png)

下圖包含OpenStack 網路(neutron) 的網路層的最小架構範例圖：
![網路架構](images/installguidearch-neutron-networks.png)

下圖包含OpenStack 網路(neutron) 服務層的最小架構範例：
![服務層](images/installguidearch-neutron-services.png)
