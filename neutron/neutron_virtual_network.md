# Neutron 中的虛擬化網路
Neutron 為 Tenant 提供了虛擬化的網路、子網路、埠口、交換器、路由器等網路元件。以下我們針對 Neutron 中虛擬化網路進行介紹。

### Layer 2 Network
網路（Network）是一個隔離的第二層網段，類似實體網路中的虛擬區域網路（VLAN），它被用來建立 Tenant 的廣播區域，或是為共享網段。埠口與子網路將在之間被分配給某個特定網路。

根據建立網路的使用者權限，Neutron network 可以分為以下：
* **Provider network  **：管理者建立的一個直接與實體網路連接的網路。
* **Tenant network**：一般使用者建立的網路，由 Neutron 根據管理者的在系統中設定決定網路的分配。

根據網路的類型，Neutron network 可以分為：
* **VLAN network**： 基於實體 VLAN 網路實現的虛擬網路。共享同一個實體網路的多個 VLAN 網路是可以互相隔離的，甚至可以使用重疊的 IP 位址空間。每個支援 VLAN Network的實體網路可以被視為一個分離的 VLAN trunk，它使用一組獨佔的 VLAN ID。有效的 VLAN ID 範圍從 ```1 ~ 4094```。
* **Flat network**：基於不採用 VLAN 的實體網路實現虛擬網路。每個實體網路最多只能實現一個虛擬網路。
* **local network**：一個只能允許在本伺服器之間溝通的虛擬網路，無法知道跨伺服器的網路通信。主要用於單節點測試。
* **GRE network （General Routing Encapsulation）**：一個使用 GRE 封裝網路封包的虛擬網路。GRE 封裝的數據包含了基於 IP 的路由表進行路由協定，因此 GRE 不與實體網路綁定。用於 ```Multi namespace``` 部署模式。
* **VXLAN network（Virtual Extensible LAN）**：基於 VXLAN 的虛擬網路如同 GRE network 一樣，VXLAN network中的 IP 封包的路由協定也是基於 IP 路由表，也不與實體網路綁定。用於 ```Multi namespace``` 部署模式以及資料中心系統轉換模式。


