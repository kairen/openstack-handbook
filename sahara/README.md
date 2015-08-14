# Sahara 資料處理套件
Sahara提供一個簡單的方式提供了一個巨量資料應用(Hadoop或是Spark)OpenStack上，Sahara的目的是提供給「搭建Haddoop分散式集群」的工程師能用簡單的概念，就能在 OpenStack上面生出和管理「Haddoop 分散式集群」。

Sahara兩大特點:
* **模組化可配置** : Sahara中的大量功能和機制，都基於可選擇、可配置的模組化實現。例如 : 可以通過對engine的配置來選擇不同的Hadoop叢集配置，通過對plugin的配置來選擇不同的Hadoop版本與部屬的方式和工具等等

* **代碼使用** :Sahara也盡可能使用了OpenStack自身提供的IaaS的服務。例如 : Nova提供虛擬機資源，利用Horizon提供操作介面等等
