# Ceilometer 資料監控套件
OpenStack 作為一個開放原始碼的 IaaS 平台，發展越來越快速，因此公司基於 OpenStack 部署自己的公有雲設施時，```計量```與```監控```這兩個基礎服務必不可少。

OpenStack 中由 ```Ceilometer``` 專案 (官方正式名稱為 ```Telemetry```) 提供計量與監控的功能。Ceilometer 的目標是：
> 為 OpenStack 環境提供一個```獲取```和```保存```各種量測值(Measurement Metrics)的統一框架。

透過 Ceilometer 管理員和用戶可以獲取和保存包括```計量 (Metering)``` 和```監控 (Monitoring)``` 的各種```測量值 (Metrics)```，可以對這些測量值依照時間進行排序等等操作，也可以根據這些測量值進行警報的功能，同時這些保存下來的測量值也可以被其他第三方工具來做更進一步的分析。

# Ceilometer 歷史
| 時間 | 事件 |
| :--: | ---- |
| 2012年4月 | Ceilometer 專案開始。 |
| 2012年10月 | 於 ```Folsom``` 版本發布第一個 Ceilometer 版本 v1.0。<br/>主要實現一些重要數據的計量，包括 ```Compute```、```Network```、```Memory```、```CPU```、```Image```、```Volume``` 等等，並提供了 ```REST API```。|
| 2013年2月 | ```Grizzly``` 版本中 Ceilometer 從```孵化(Incubation)```狀態畢業，成為 OpenStack 的```整合專案(Integrated Project)```。<br/> |
| Grizzly | 增加對 ```Swift``` 的支援以及 ```SQLAlchemy``` 作為 Storage Backend，開發了 ```Multi Publisher```，並發布 ```v2``` 版本的 API。 |
| Havana | 增加了 ```HBase``` 作為 Storage Backend，```Alarm``` 功能也基本完成，並且增加了 ```UDP Publisher``` 作為取代 RPC 發送消息的第二選擇，擁有更好的效率。還增加了 ```DB2``` 作為 Storage Backend。 |
| IceHouse | 主要是分離 ```Collector```，增加 vmware vcenter server 支持等。 |
| Juno | 主要是進行優化，並添加 ```Temptest```。 |

# Ceilometer 任務
Ceilometer 最初目的只是：
> 收集資訊來對用戶進行收費

隨著專案的發展，OpenStack
社群發現很多專案都許耀收集很多不同類型的測量值，所以 Ceilometer 專案又增加了第二個目標：
> 成為 OpenStack 系統裡一個標準的收集測量值的框架

後來由於 ```Heat 專案```的需求，Ceilometer 又增加了利用以保存的測量值進行```警報```的功能。
