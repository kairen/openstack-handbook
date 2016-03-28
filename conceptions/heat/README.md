# Heat 協調整合套件

Heat是OpenStack負責編排計畫的主要項目，他可以基於模板來實現雲環境資源初始化，也可以解決自動收縮與附載平衡等特性。同時也支援AWS CloudFormation樣板。
Heat提供了OpenStack原生REST API和CloudFromation相容的API查詢。

* Heat提供一個基本的流程執行適當的OpenStack API調用來產生雲端應用
* Heat結合OpenStack核心部分成為一個樣板(Template)系統，這些樣板允許創建大多數的OpenStack類型，(像是instances, floating ips, volumes, security groups, users等等)還有一些進階功能，像是高可用性(HA)，instance的自動擴展(instance autoscaling)和nested stacks.em.
* 允許管理者對Heat添加自行定義的插件
