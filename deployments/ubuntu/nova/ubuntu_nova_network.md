# Ubuntu Nova Network 多節點安裝
我們已經完成了單一節點的安裝，但是要將功能拆分成多台該如何實現呢？這次我們安裝包含了```nova-network``` 的兩個節點架構以及選擇性的```區塊儲存```和```物件儲存```服務節點：
* **Controller Node** : 會運作```KeyStone```、```Glance```、管理運算的部分，並運作```Horizon儀表板```。還包括一些支援的服務，例如```SQL資料庫```、```訊息佇列(message queue)```和```網路時間協定(NTP)```。
戶虛擬機實例的```外部網路(Internet)```連接。
* **Compute node** : 運作Compute的```hypervisor```部分，這個部分將操作tenant virtual machines或實例。預設情況下，Compute使用```KVM```作為hypervisor。Compute也提供連接租戶網路到虛擬機上，並提供防火牆( security groups )服務。

 > 可以運作不止一台運算節點。

* **Block Storage Node** :  區塊儲存節點包含磁碟，區塊儲存服務會向租戶虛擬機實例提供這些磁碟。

 > 可以運作多個區塊儲存節點，來擴展Volume。

* **Object Storage Node** : 物件儲存節點包含磁碟，物件儲存服務使用這些磁碟來儲存```帳戶```、```容器```和```物件```。

 > 可以運行多個物件儲存節點。但是在最小架構範例中，需要兩個節點。

## 最小架構環境需求
下圖包含OpenStack 傳統網路(nova-network) 的最小架構範例的硬體需求：
![硬體需求架構圖](images/installguidearch-nova-hw.png)

下圖包含OpenStack 傳統網路(nova-network) 的網路層的最小架構範例圖：
![網路架構](images/installguidearch-nova-networks.png)

下圖包含OpenStack 傳統網路(nova-network) 服務層的最小架構範例：
![服務層](images/installguidearch-nova-services.png)
