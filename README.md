# OpenStack 技術
OpenStack是一個```美國國家航空暨太空總署```和```Rackspace```合作研發的雲端運算軟體，以```Apache```許可證授權，並且是一個自由軟體和開放原始碼項目，來打造```基礎設施即服務(Infrastructure as a Service)```。OpenStack內部包括了```運算模組```、```網通模組```和```儲存模組```，再搭配一個可以集中管理上述三大類模組的儀表板模組，最後組合而成一套OpenStack共享服務，並且可以提供虛擬機器的方式，對外提供運算資源以便彈性擴充或調度。

從2010年10月到現今已歷經12個版本，來到了```Kilo```與下一個版本```Liberty```，套件數也從A版的兩個發展到現今```12```個套件，許多大廠也紛紛加入該行列，打造一套自己的雲端平台。

![OpenStack](images/openstack_kilo_conceptual_arch.png)

# 套件介紹
### Keystone 身分識別套件 (Identity service)
Keystone套件作為OpenStack的```身分驗證系```統，具有中央目錄，能查看哪位使用者可存取哪些服務，並且，提供了多種驗證方式，包括使用者帳號密碼、Token以及類似AWS的登入機制。另外，Keystone可以整合現有的中央控管系統，像是LDAP（輕型目錄訪問協議）。

### Nova 運算套件 (Compute)
Nova 主要負責```部署```與```管理虛擬機角```色。Nova提供了一套API來開發額外的應用程式，IT人員可以透過網頁介面來查看與管理資源狀態，且可以控制啟動、停止、調整虛擬機。。

IT人員可將Nova套件部署在多家廠商的虛擬化平臺上，目前來說，以KVM和Xen虛擬化平臺最為穩定。除了支援不同的虛擬化平臺之外，在硬體架構的部份，OpenStack支援x86架構、ARM架構等。另外，Nova套件還支援Linux羽量級的虛擬化技術LXC，能夠再切割虛擬機器，分出更多的虛擬化執行環境。

此外，Nova套件還具有管理LAN網路的功能，可程式化的分配IP位址與VLAN，快速部署網路與資安功能。Nova套件還可將某幾臺虛擬機器設為群組，和不同群組作隔離，並有基於角色的訪問控制（RBAC）功能，可根據使用者的角色確保可存取的資源為何。

### Glance 映象檔管理套件 (Image Service)
Glance套件提供提供硬碟或伺服器的```映象檔尋找```、```註冊```以及```服務交付```等功能。儲存的映象檔可作為新伺服器部署所需的範本，加快服務上線速度。若是有多臺伺服器需要配置新服務，就不需要額外花費時間單獨設定，也可做為備份時所用。

### Horizon 儀表板套件 (Dashboard)
Horizon套件提供IT人員一個```圖形化的網頁介面```，讓IT人員可以綜觀雲端服務目前的規模與狀態，並且，能夠統一存取、部署與管理所有雲端服務所使用到的資源。

Horizon套件是個可擴展的網頁式App。所以，Horizon套件可以整合第三方的服務或是產品，像是計費、監控或是額外的管理工具。

### Neutron 網通套件 (Networking)
Neutron套件確保為其它OpenStack服務提供```網路連接即服務（Network-Connectivity-as-a-Service）```，比如OpenStack運算。為租戶提供API定義網路和使用。基於插件的架構，使其支援眾多的網路提供商和技術，，IT人員可配置IP位址，分配靜態IP或是動態IP。而且，IT人員可使用SDN技術，像是OpenFlow協定來打造更大規模或是多租戶的網路環境。

此外，允許部署和管理其他網路服務，像是入侵偵測系統（IDS）、負載平衡、防火牆、VPN等。

### Swift 物件儲存套件 (Object Storage)
Swift套件提供可擴展的```分散式儲存平```臺，以防止```單點故```障的情況產生。使用者可透過API進行存取，可存放非結構化的資料，像是圖片、網頁、網誌等，並可作為應用程式資料備份、歸檔以及保留之用。

透過Swift套件，可讓業界標準的設備存放PB等級的資料量。而且，當新增伺服器後，儲存群集可輕易的橫向擴充。

此外，因為Swift套件是透過軟體的邏輯，確保資料被複製與分布在不同設備上，這可讓企業使用較便宜的設備，節省成本。

### Cinder 區塊儲存套件 (Block Storage)
Cinder套件允許```區塊儲存設備能夠整合商業化的企業儲存平臺```，像是NetApp、Nexenta、SolidFire等。區塊儲存系統可讓IT人員設置伺服器和區塊儲存設備的各項指令，包括建立、連接和分離等，並整合了運算套件，可讓IT人員查看儲存設備的容量使用狀態。

Cinder套件並提供```快照管理功能```，可保護虛擬機器上的資料，作為系統回復時所用，快照甚至可用來建立一個新的區塊儲存容量。

### Ceilometer 資料監控套件(Telemetry)
Ceilometer為OpenStack雲端服務可藉由```監控```與```量測```OpenStack的使用，來收集CPU與網路的使用資料，以提供```收費計價（Billing）```、```評測（Benchmarking）```等使用，或是使用這些資料當作評估系統延展性以及進行系統相關統計之用。

### Heat 協調整合套件 (Orchestration)
Heat主要提供一個以```模板（Templeate）```為基礎的架構來描述雲端的應用，模板中可以讓使用者建立如虛擬映像實體（Instance）、浮動IP位址、安全群組（Security Group）或是使用者等OpenStack各種資源，也就是說，Heatn讓使用者可以設定一個雲端應用模板來串連建立設定相關所需的OpenStack服務資源，而不必一個個分別去建立設定。

### Sahara 資料處理套件 (Hadoop as a Service)
Sahara 的目的是提供給```搭建Haddoop 分散式集群```的工程師能用簡單的概念，
就能在 OpenStack 上面部署和管理「Haddoop 分散式集群」。

### Trove 資料庫服務套件 (Database as a Service)
為Trove主要負責```銜接```與```簡化```實際資料庫的使用，提供OpenStack各個服務一個具延展性且可靠的```雲端資料庫服務（Cloud Database-as-a-Service）```，Database服務包含了銜接傳統關聯式資料庫與新興非關聯式資料庫。

### Ironic 裸機部署套件 (Bare Metal Service)
Ironic裸機部署功能，在Kilo版中釋出，IT人員可以在實體伺服器自動化部署OpenStack，等於能用管理虛擬機器的方式，來管理實體伺服器，有助於一次部署大量OpenStack主機來滿足大型IaaS環境的需要。

### Zaqar

# 各套件Launchpad網址
| 名稱 | 網址 |
| -- | -- |
| Keystone | https://launchpad.net/keystone |
| Nova | https://launchpad.net/nova |
| Glance | https://launchpad.net/glance |
| Neutron | https://launchpad.net/neutron |
| Horizon | https://launchpad.net/horizon |
| Cinder | https://launchpad.net/cinder |
| Swift | https://launchpad.net/swift |
| Ceilometer | https://launchpad.net/ceilometer |
| Heat | https://launchpad.net/heat |
| Sahara | https://launchpad.net/sahara |
| Trove | https://launchpad.net/trove |
| Ironic | https://launchpad.net/ironic |

# 版本資訊
| 名稱 | 官方網址 |
| -- | -- |
| 版本藍圖 | https://wiki.openstack.org/wiki/Releases |
| Kilo版本 | https://wiki.openstack.org/wiki/ReleaseNotes/Kilo |
| Stackalytics | http://stackalytics.com/ |

# 安裝工具
* [DevStack](http://docs.openstack.org/developer/devstack/)
* [Red Hat OpenStack](https://www.rdoproject.org/Main_Page)
* [Fuel OpenStack](https://wiki.openstack.org/wiki/Fuel)
* [HP Helion](http://www8.hp.com/us/en/cloud/helion-overview.html)
* [Compass Openstack](https://wiki.openstack.org/wiki/Compass)
* [TripleO](https://wiki.openstack.org/wiki/TripleO)
* [PackStack](https://wiki.openstack.org/wiki/Packstack)
* [puppetlabs-openstack](https://github.com/puppetlabs/puppetlabs-openstack)
* [Ubuntu OpenStack](https://wiki.ubuntu.com/ServerTeam/OpenStackHA)

