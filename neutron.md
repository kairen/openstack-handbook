# Neutron 網路套件
Neutron提供了```Network Service  ```負責虛擬網路架構與外部實體網路架構(包含各廠商的設備)之間的整合 & 存取，用以提供 tenant 可自行建立像是防火牆、負載平衡、VPN … 等網路環境。

網路設定的概念其實跟我們之前所學的並沒有差太多，包含 DHCP、VLAN、Routing … 等等。

比較值得一提的是 Networking Service 支援了 security group 的概念：

* 管理者可以針對每個 security group 進行防火牆規則的設定
* 每個 VM 可以屬於一個或多個 security group

有了以上的機制，讓 Firewall-as-a-Service & Load-Balancing-as-a-Service 變的很容易實現。

![架構圖](images/1aa-network-domains-diagram.png)
