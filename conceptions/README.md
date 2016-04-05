# OpenStack 介紹技術
OpenStack是```美國國家航空暨太空總署```和```Rackspace```共同打造的雲端開源軟體，以```Apache```許可證授權，並且是一個自由軟體和開放原始碼項目，來打造```基礎設施即服務(Infrastructure as a Service)```。OpenStack擁有三大模組```運算模組```、```網通模組```和```儲存模組```，加上一套集中式管理的```儀表板模組```，來組合成一套OpenStack共享服務，並且以提供虛擬機方式，對外帶來運算資源，以便利彈性擴充或調度。

從2010年10月到現今已歷經12個版本，來到了```Liberty```與下一個版本```Mitaka```，專案數也從A版的```2```個發展到現今超過```10```個以上專案，許多大廠也紛紛加入該行列，打造一套自己的雲端平台。

值得一提的是 OpenStack 已在 Liberty 加入了 Big Tent 模型。讓管理人員只需要更新核心的專案，其餘專案可以隨自己需求選擇是否要更新。也將在 2016年第二季推出第一個OpenStack認證管理員（COA）認證。

![OpenStack](images/openstack_kilo_conceptual_arch.png)

# 套件介紹
以下列表為 OpenStack 目前比較成熟與較受關注的服務套件。

- [OpenStack Core Service](#)
    - [Keystone 身分識別套件](#keystone-身分識別套件)
    - [Glance 映象檔管理套件](#glance-映象檔管理套件)
    - [Nova 運算套件](#nova-運算套件)
    - [Neutron 網通套件](#neutron-網通套件)
    - [Horizon 儀表板套件](#horizon-儀表板套件)
    - [Cinder 區塊儲存套件](#cinder-區塊儲存套件)
    - [Swift 物件儲存套件](#wwift-物件儲存套件)
- [OpenStack Big Tent Service](#)
    - [Heat 編排模板套件](#keystone-身分識別套件)
    - [Ceilometer 資料監控計量套件](#keystone-身分識別套件)
    - [Sahara 資料處理套件](#keystone-身分識別套件)
    - [Trove 資料庫服務套件](#keystone-身分識別套件)
    - [Ironic 裸機部署套件](#keystone-身分識別套件)
    - [Zaqar 雲端訊息佇列服務](#keystone-身分識別套件)
    - [Barbican 金鑰管理服務](#keystone-身分識別套件)
    - [Designate DNS管理服務](#keystone-身分識別套件)
    - [Manila 共享式檔案系統服務](#keystone-身分識別套件)
    - [Magnum 容器即服務](#keystone-身分識別套件)
    - [Murano 應用程式目錄服務](#keystone-身分識別套件)
    - [Monasca](#keystone-身分識別套件)
    - [Senlin](#keystone-身分識別套件)

### Keystone 身分識別套件 (Identity service)
Keystone套件作為OpenStack的```身份驗證```服務，具有中央目錄能查看哪位使用者可存取哪些服務，並且提供了多種驗證方式，包括使用者帳號密碼、Token以及類似AWS的登入機制。另外，Keystone可以整合現有的中央控管系統，像是LDAP（輕型目錄訪問協議）。
> 類似 Amazon AWS 的 IAM。

### Glance 映象檔管理套件 (Image Service)
Glance套件提供了硬碟或伺服器的Image```尋找```、```註冊```以及```服務交付```等功能。儲存的Image可作為新伺服器部署所需的範本，加快服務上線速度。若是有多臺伺服器需要配置新服務，就不需要額外花費時間單獨設定，也可做為備份時所用。
> 類似 Amazon AWS 的 VM Import／Export。

### Nova 運算套件 (Compute)
Nova 主要擔任著```部署```與```管理```虛擬機角色。Nova提供了一套API來開發額外的應用程式，IT人員可以透過網頁介面來查看與管理資源狀態，且可以控制啟動、停止、調整虛擬機。

IT人員可將Nova套件部署在多家廠商的虛擬化平臺上，目前來說以```KVM```和```Xen```虛擬化平臺最為穩定。除了支援不同的虛擬化平臺之外，在硬體架構的部份，OpenStack支援```x86```架構、```ARM```架構等。另外Nova套件還支援Linux羽量級的虛擬化技術```LXC```，能夠在切割虛擬機時，分出更多的虛擬化執行環境。

此外，Nova套件還具有管理LAN網路的功能，可程式化的分配IP位址與VLAN，快速部署網路與資安功能。Nova套件還可將某幾臺虛擬機器設為群組，和不同群組作隔離，並有基於角色的訪問控制（RBAC）功能，可根據使用者的角色確保可存取的資源為何。
> 類似 Amazon AWS 的 EC2。

### Neutron 網通套件 (Networking)
Neutron套件為其它OpenStack服務提供```網路連接即服務（Network-Connectivity-as-a-Service）```功能。比如OpenStack運算，為租戶提供API定義網路和使用。基於插件式的架構，使其支援眾多的網路供應商和技術，，IT人員可分配IP位址、靜態IP或是動態IP。且IT人員也可以使用SDN技術，像是OpenFlow協定來打造更大規模或是多租戶的網路環境。

此外，允許部署和管理其他網路服務，像是入侵偵測系統（IDS）、負載平衡、防火牆、VPN等。
> 類似 Amazon AWS 的 VPC。

### Horizon 儀表板套件 (Dashboard)
Horizon套件提供IT人員一套```圖形化的網頁介面```，讓IT人員可以綜觀雲端服務目前的規模與狀態，並且能夠統一存取、部署與管理所有雲端服務所使用到的資源。

Horizon套件是個可擴展的網頁式Application。所以Horizon套件可以整合第三方的服務或是產品，像是計費、監控或是額外的管理工具。
> 類似 Amazon AWS 的 Console。

### Cinder 區塊儲存套件 (Block Storage)
Cinder套件允許區塊儲存設備能夠整合商業化的企業儲存平臺，像是NetApp、Nexenta、SolidFire等。區塊儲存系統可讓IT人員設置伺服器和區塊儲存設備的各項指令，包括建立、連接和分離等，並整合了運算套件，可讓IT人員查看儲存設備的容量使用狀態。

Cinder套件並提供```快照管理功能```，可保護虛擬機器上的資料，作為系統回復時所用，快照甚至可用來建立一個新的區塊儲存容量。
> 類似 Amazon AWS 的 EBS。

### Swift 物件儲存套件 (Object Storage)
Swift套件提供可擴展的```分散式儲存平臺```，以防止```單點故障```的情況發生。使用者可透過API進行存取，可存放非結構化的資料，像是圖片、網頁、網誌等，並可作為應用程式資料備份、歸檔以及保留之用。

透過Swift套件，可讓業界標準的設備存放PB等級的資料量。而且，當新增伺服器後，儲存群集可輕易的橫向擴充。

此外，因為Swift套件是透過軟體的邏輯，確保資料被複製與分布在不同設備上，這可讓企業使用較便宜的設備，節省成本。
> 類似 Amazon AWS 的 S3。

### Heat 編排模板套件 (Orchestration)
Heat主要提供一個以```模板（Templeate）```為基礎的架構來描述雲端的應用，模板中可以讓使用者建立如虛擬映像實體（Instance）、浮動IP位址、安全群組（Security Group）或是使用者等OpenStack各種資源，也就是說，Heat讓使用者可以設定一個雲端應用模板，來串連建立設定相關所需的OpenStack服務資源，而不必一個個分別去建立設定。

### Ceilometer 資料監控計量套件(Telemetry)
Ceilometer提供OpenStack雲端服務可藉由```監控```與```量測```OpenStack的使用，來收集CPU與網路的使用資料，以提供```收費計價（Billing）```、```評測（Benchmarking）```等使用，或是使用這些資料當作評估系統延展性以及進行系統相關統計之用。

### Sahara 資料處理套件 (Data Processing)
Sahara 目的是提供給搭建```Haddoop 分散式叢集```的工程師能用簡單的概念，
就能在 OpenStack 上面部署和管理「Haddoop 分散式叢集」。Sahara 也提供了```MapR Distribution```、```Spark```、```Cloudera```、```Hortonworks```插件，替IT人員打造一系列```Hadoop ecosystem```。

### Trove 資料庫服務套件 (Database as a Service)
Trove主要負責```銜接```與```簡化```實際資料庫的使用，提供OpenStack各個服務一個具延展性且可靠的```雲端資料庫服務（Cloud Database-as-a-Service）```，Database服務包含了銜接傳統關聯式資料庫與新興非關聯式資料庫。

### Ironic 裸機部署套件 (Bare Metal )
Ironic裸機部署功能，在Kilo版中釋出，IT人員可以在實體伺服器自動化部署OpenStack，等於能用管理虛擬機器的方式，來管理實體伺服器，有助於一次部署大量OpenStack主機來滿足大型IaaS環境的需要。

### Zaqar 雲端訊息佇列服務(Message service)
Zaqar 是對 Web 開發人員提供了```多租戶（Multi tenant）```的雲端訊息服務。它結合開創了 Amazon 的 SQS 產品與附加的語義來支援事件的廣播想法。

本服務擁有一個完全基於 RESTful 的 API，開發人員可使用他們的 Saas 與行動應用程式的各種元件之間的訊息發送，透過使用多種通訊模式。這個 API 是一種高效的訊息傳送引擎設計，充分的考慮可擴展性與安全性。

然而其他 OpenStack 的套件可以與 Zaqar 的表面事件 End users 進行整合以及與訪客的Agent運作於 『Over-cloud』層。雲端公司可以利用Zaqar提供如同 SQS 與 SNS給他們的客戶。

### Barbican 金鑰管理服務(Key management)
Barbican 是一個以 REST API 設計來進行安全儲存、配置以及機密的管理，如密碼、加密金鑰以及 X.509 憑證。其目的是為了適用於所有環境，包含大型短暫性雲端。

### Designate DNS管理服務 (DNS)
Designate 提供了 DNSaaS 服務於 OpenStack 上，包含以下幾項功能：
* 使用 REST API 管理 domain/record
* 多租戶
* 整合 Keystone 驗證
* 以框架來整合 Nova 與 Neutrion 的通知（自動產生記錄）
* 支援立即可用的 PowerDNS 與 Bind9

### Manila 共享式檔案系統服務 (Shared Filesystems)
Manila 提供 OpenStack 共享的檔案系統，核心概念有共享目錄、ACL、共享網路、快照與後端驅動程式，目前支援有 GPFS、GlusterFS、EMCVNX等。在雲端平台上，所有服務必須要考慮多租戶資源隔離，目前 Manila 的多租戶資源隔離依賴於 Neutron 的私有網路隔離。

### Magnum 容器即服務 (Containers service)
Magnum 是一個 OpenStack API 服務，是由 OpenStack Containers Team 開發作為``` Container orchestration``` 的引擎，諸如 Docker、Kubernetes 這一類別可以在 Openstack 上作為資源。

Magnum 使用 Heat 來編排一個 OS Image，其中包含 Docker 以及 Kubernetes，並執行 Image 於任何的虛擬機或 bare metal 叢集配置。

### Murano 應用程式目錄服務(Application catalog)
Murano 專案引入一個 Application catalog 於 OpenStack上，使應用程式開發人員與雲端管理人員，可以發布各種已就緒的雲端應於可瀏覽的分類目錄。

雲端使用者、包括沒經驗的人可以通過統一的框架與 API 實現應用程式的快速部署與應用程式的生命週期管理，來降低應用程式對底層平台（IaaS層）的依賴。

### Monasca

### Senlin
