# Ceilometer 安裝與設定
本章節會說明與操作如何安裝遙測服務到 Controller 節點上，並設置相關參數與設定。若對於 Ceilometer 不瞭解的人，可以參考[Ceilometer 遙測服務章節](../../../conceptions/ceilometer/README.md)

- [Controller Node](#controller-node)
    - [安裝前準備](#安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [啟用 Image Service Meters](#啟用-image-service-meters)
- [啟用 Compute Service Meters](#啟用-compute-service-meters)
- [啟用 Block Storage Meters](#啟用-block-storage-meters)
- [啟用 Object Storage Meters](#啟用-object-storage-meters)
- [驗證服務](#驗證服務)

# Controller Node
在 Controller 節點我們需要安裝 Ceilometer 的 API Server 以及用於儲存資料的 MongoDB。

### 安裝前準備
由於 Ceilometer 的資料庫使採用 ```MongoDB``` 來做儲存，可以透過以下方式進行安裝：
```sh
$ sudo apt-get install mongodb-server mongodb-clients python-pymongo
```

安裝完成後，編輯```/etc/mongodb.conf```設定檔，並加入以下內容：
```sh
bind_ip = 10.0.0.11
smallfiles = true
```
> 預設下 MongoDB 會建立一個 /var/lib/mongodb/journal 目錄來儲存 journal 檔案（每個檔案 1GB 大小），若要縮小 journal 檔案大小可以設定 ```smallfiles```為```true```。

若有修改 journal 的設定，請務必停止服務並刪除 journal 初始化檔案，再重新啟動服務：
```sh
sudo service mongodb stop
sudo rm /var/lib/mongodb/journal/prealloc.*
sudo service mongodb start
```
> 你也可以關閉 journaling，詳細資訊可以看 [MongoDB manual](http://docs.mongodb.org/manual/).

確認修改後，透過以下命令用來更新現有帳號資料或建立 Ceilometer 資料庫：
```sh
$ mongo --host 10.0.0.11 --eval '
  db = db.getSiblingDB("ceilometer");
  db.addUser({user: "ceilometer",
  pwd: "CEILOMETER_DBPASS",
  roles: [ "readWrite", "dbAdmin" ]})'
```

建立成功會看到類似以下訊息：
```sh
MongoDB shell version: 2.4.9
connecting to: controller:27017/test
{
	"user" : "ceilometer",
	"pwd" : "447c1db3b92df9035684b39ad9fa2185",
	"roles" : [
		"readWrite",
		"dbAdmin"
	],
	"_id" : ObjectId("55964bbc2d42ee01755e590a")
}
```
> 這邊```CEILOMETER_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Ceilometer 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Ceilometer User
$ openstack user create --domain default --password CEILOMETER_PASS --email ceilometer@example.com ceilometer

# 建立 Ceilometer Role
$ openstack role add --project service --user ceilometer admin

# 建立 Ceilometer Service
$ openstack service create --name ceilometer  --description "Telemetry" metering

# 建立 Ceilometer public endpoints
$ openstack endpoint create --region RegionOne \
metering public http://10.0.0.11:8777

# 建立 Ceilometer internal endpoints
$ openstack endpoint create --region RegionOne \
metering internal http://10.0.0.11:8777

# 建立 Ceilometer admin endpoints
$ openstack endpoint create --region RegionOne \
metering admin http://10.0.0.11:8777
```
> 這邊```CEILOMETER_PASS```可以隨需求修改。

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install ceilometer-api ceilometer-collector \
ceilometer-agent-central ceilometer-agent-notification \
python-ceilometerclient
```

安裝完成後，編輯```/etc/ceilometer/ceilometer.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
```

在``` [database]```部分修改使用以下方式：
```
[database]
connection = mongodb://ceilometer:CEILOMETER_DBPASS@10.0.0.11:27017/ceilometer
```
> 這邊```CEILOMETER_DBPASS```可以隨需求修改。

在```[oslo_messaging_rabbit]```部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```

在```[keystone_authtoken]```部分加入以下內容：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = ceilometer
password = CEILOMETER_PASS
```
> 這邊```CEILOMETER_PASS```可以隨需求修改。

在```[service_credentials] ```部分加入以下內容：
```
[service_credentials]
os_auth_url = http://10.0.0.11:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = CEILOMETER_PASS
interface = internalURL
region_name = RegionOne
```
> 這邊```CEILOMETER_PASS```可以隨需求修改。

完成所有設定後就可以重新啟動所有 Ceilometer 服務：
```sh
sudo service ceilometer-agent-central restart
sudo service ceilometer-agent-notification restart
sudo service ceilometer-api restart
sudo service ceilometer-collector restart
```

# 啟用 Image Service Meters
Ceilometer 使用 Notifications 來收集映像檔服務的 Meters。到 ```Controller（Glance）``` 節點上編輯```/etc/glance/glance-api.conf```、```/etc/glance/glance-registry.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
```

在```[oslo_messaging_notifications]```部分加入以下內容：
```sh
[oslo_messaging_notifications]
driver = messagingv2
```

在```[oslo_messaging_rabbit]```部分加入以下內容：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

完成後，就可以重新啟動所有 Glance 服務：
```sh
sudo service glance-registry restart
sudo service glance-api restart
```

# 啟用 Compute Service Meters
Ceilometer 使用 notifications 與 agent 的組合來收集 Compute 的 meters。到所有 ```Compute``` 節點進行以下步驟：
Telemetry 透過結合使用通知與proxy來收集Computer測量值。若要監控每個Compute節點，需在每個節點進行以下步驟：

在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install ceilometer-agent-compute
```

安裝完成後，編輯```/etc/ceilometer/ceilometer.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
```

在```[oslo_messaging_rabbit]```部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊```RABBIT_PASS```可以隨需求修改。

在```[keystone_authtoken]```部分加入以下內容：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = ceilometer
password = CEILOMETER_PASS
```
> 這邊```CEILOMETER_PASS```可以隨需求修改。

在```[service_credentials]```部分加入以下內容：
```
[service_credentials]
os_auth_url = http://10.0.0.11:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = CEILOMETER_PASS
interface = internalURL
region_name = RegionOne
```

接著編輯```/etc/nova/nova.conf ```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state
notification_driver = messagingv2
```

完成後，就可以重新啟動所有相關服務：
```sh
sudo service ceilometer-agent-compute restart
sudo service nova-compute restart
```

# 啟用 Block Storage Meters
Ceilometer 使用 Notifications 來收集區塊儲存服務的 meters。我們須在 ```Controller```與 ```Storage``` 節點上執行以下步驟。

首先編輯所有節點的```/etc/cinder/cinder.conf```設定檔，在```[oslo_messaging_notifications]```部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messagingv2
```

在 ```Controller``` 節點，重新啟動以下服務：
```sh
sudo service cinder-api restart
sudo service cinder-scheduler restart
```

在 ```Storage``` 節點，重新啟動以下服務：
```sh
$ sudo service cinder-volume restart
```

# 啟用 Object Storage Meters
Ceilometer 使用 Pooling 與 Notifications 組合來收集物件儲存服務的 meters。由於 Ceilometer 服務需要使用到 ResellerAdmin 來取得物件儲存的 meters。首先在 ```Controller``` 執行以下步驟：
```sh
$ . admin-openrc
```

透過 Keystone client 建立 ResellerAdmin role：
```sh
$ openstack role create ResellerAdmin
```

然後將 ResellerAdmin role 加入到 Server project，並賦予給 ceilometer 使用者：
```sh
$ openstack role add --project service --user ceilometer ResellerAdmin
```

完成後，在進行設定前必須先安裝相關套件：
```sh
$ sudo apt-get install python-ceilometermiddleware
```

接著在```Controller```節點上編輯```/etc/swift/proxy-server.conf```設定檔，在```[filter:keystoneauth]```部分加入以下內容：
```
[filter:keystoneauth]
...
operator_roles = admin, user, ResellerAdmin
```

在```[pipeline:main]```部分加入以下內容：
```
[pipeline:main]
...
pipeline = catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging ceilometer proxy-server
```

在```[filter:ceilometer]```部分加入以下內容：
```
[filter:ceilometer]
...
paste.filter_factory = ceilometermiddleware.swift:filter_factory
control_exchange = swift
url = rabbit://openstack:RABBIT_PASS@10.0.0.11:5672/
driver = messagingv2
topic = notifications
log_level = WARN
```
> 這邊```RABBIT_PASS```可以隨需求修改。

完成後就可以重新啟動 Swift Proxy 服務：
```sh
$ sudo swift-init all restart
```

# 驗證服務
導入該環境參數，來透過 Ceilometer client 查看服務狀態：
```sh
$ . admin-openrc
```

這邊可以透過 Ceilometer client 來查看所有 meter，如以下方式：
```sh
$ ceilometer meter-list
```

透過 Ceilometer client 來查看哪個期間的資料，如以下方式：
```sh
$ ceilometer statistics -m image.download -p 60
```
