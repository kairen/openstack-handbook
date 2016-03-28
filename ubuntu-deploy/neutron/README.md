# Neutron OVS GRE 安裝與設定
本章節會說明與操作如何安裝```Networking```服務到 Controller、Compute、Network 節點上，並設置相關參數與設定。若對於Neutron不瞭解的人，可以參考 [Neutron 網路套件章節](http://kairen.gitbooks.io/openstack/content/neutron/index.html)

Neutron 提供了虛擬機器與運算之間的軟體定義網路功能。Neutron 引用了 plugin 來實現網路定義 API 的後端。這些 plugin 可能使用了基本的 Linux 網路功能，如 Linux Bridge、VLANs、IP Tables等，更能夠使用 SDN 技術，如 OpenFlow、L2 - L3 隧道技術。目前 Neutron 有許多部署方式，如 Linix Bridge、Open vSwitch、廠商 Driver等，以下為目前已完成教學的部署：

* [Neutron VXLAN with Linux Bridge](linuxbridge_vxlan_install.md)
* [Neutron GRE with Open vSwitch](openvswitch_gre_install.md)
