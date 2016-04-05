# Neutron 安裝與設定
本章節會說明與操作如何安裝網路服務到 Controller、Compute 與 Network 節點上，並設定安裝相關參數與套件。若對於 Neutron 不瞭解的人，可以參考 [Neutron 網路服務章節](../../../conceptions/neutron/README.md)。

Neutron 提供了虛擬機器與運算之間的軟體定義網路功能。Neutron 引用了 plugin 來實現網路定義 API 的後端。這些 plugin 可能使用了基本的 Linux 網路功能，如 Linux Bridge、VLANs、IP Tables 等，更能夠使用 SDN 技術，如 OpenFlow、L2 - L3 隧道技術。目前 Neutron 有許多部署方式，如 Linix Bridge、Open vSwitch、廠商 Driver 等，以下為目前提供的教學：

* [Neutron VXLAN with Linux bridge agent](linuxbridge-vxlan-install.md)
* [Neutron GRE with Open vSwitch agent](openvswitch-gre-install.md)
* [Neutron VXLAN with Open vSwitch agent](openvswitch-vxlan-install.md)
