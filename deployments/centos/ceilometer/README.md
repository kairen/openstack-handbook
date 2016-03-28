# Ceilometer 安裝與設定
本章節會說明與操作如何安裝```Telemetry```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Ceilometer不瞭解的人，可以參考[Ceilometer 資料監控套件章節](http://kairen.gitbooks.io/openstack/content/ceilometer/index.html)

# Controller節點安裝與設置
這部分將說明如何在Controller上安裝與設定Telemetry服務（即ceilometer），模組採用分離的代理來從Openstack服務中收集測量值。

### 安裝前準備
由於 Ceilometer 資訊的資料庫，利用```MongoDB```收集資料，我們要透```yum```指令安裝相關套件：
```sh
yum install mongodb-server mongodb
```
安裝完成後，編輯``` /etc/mongod.conf```，設定ip位置：
```sh
bind_ip = 10.0.0.11
smallfiles = true
```
> 預設情況下，MongoDB 會在/var/lib/mongodb/journal 目錄下建立一些1GB 的journal 檔案。如果想要縮小每個journal 檔案為```128MB```，並限制總數的journal空間設為```512M```，讓設定```smallfiles```為```true```。

重新開啟服務，並設定boot開啟：：
```sh
systemctl enable mongod.service
systemctl start mongod.service
```
> 你也可以關閉journaling，詳細資訊可以看[MongoDB manual](http://docs.mongodb.org/manual/).

建立 Ceilometer 資料庫：
```sh
mongo --host controller --eval '
  db = db.getSiblingDB("ceilometer");
  db.createUser({user: "ceilometer",
  pwd: "CEILOMETER_DBPASS",
  roles: [ "readWrite", "dbAdmin" ]})'
```
建立成功會看到類似以下訊息：
```sh
MongoDB shell version: 2.6.x
connecting to: controller:27017/test
Successfully added user: { "user" : "ceilometer", "roles" : [ "readWrite", "dbAdmin" ] }
```
> 可自行選擇更改```CEILOMETER_DBPASS```密碼

導入```admin```來建立服務驗證：
```sh
# 建立 Ceilometer user
openstack user create --password CEILOMETER_PASS --email ceilometer@example.com ceilometer

# 建立 Ceilometer Role
openstack role add --project service --user ceilometer admin

# 建立 Ceilometer Service
openstack service create --name ceilometer  --description "Telemetry" metering

# 建立 Ceilometer Endpoints API
openstack endpoint create --publicurl http://controller:8777 --internalurl http://controller:8777 --adminurl http://controller:8777 --region RegionOne metering
```
> 可自行選擇更改```CEILOMETER_PASS```密碼

### 安裝與設置Ceilometer套件
首先我們要透過```yum```安裝相關套件：
```sh
yum install openstack-ceilometer-api openstack-ceilometer-collector \
  openstack-ceilometer-notification openstack-ceilometer-central openstack-ceilometer-alarm \
  python-ceilometerclient
```
然後透過```openssl```產生一個亂數：
```sh
openssl rand -hex 10
# 4d246a04c645a2fa04d9
```
編輯```/etc/ceilometer/ceilometer.conf```，在``` [database]```部分加入以下：
```
[database]
connection = mongodb://ceilometer:CEILOMETER_DBPASS@controller:27017/ceilometer
```
> 可自行選擇更改```CEILOMETER_DBPASS```密碼

在```[DEFAULT]```加入RabbitMQ與Keystone存取：
```
rpc_backend = rabbit
auth_strategy = keystone
```
在```[oslo_messaging_rabbit]```部分，修改為以下：
```
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
在```[keystone_authtoken]```部分，修改設定keystone存取以及註解所有auth_host、auth_port 和auth_protocol，因為Keystone預設已包含：
```
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = ceilometer
admin_password = CEILOMETER_PASS
```
> 可自行選擇更改```CEILOMETER_PASS```密碼

在```[service_credentials] ```部分，修改以下：
```
[service_credentials]
os_auth_url = http://controller:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = CEILOMETER_PASS
os_endpoint_type = internalURL
os_region_name = RegionOne
```
> 可自行選擇更改```CEILOMETER_PASS```密碼

在```[publisher]```部分，修改以下：
```
[publisher]
telemetry_secret = TELEMETRY_SECRET
```
> 將```TELEMETRY_SECRET```更改為剛剛產生的亂數字串 ```4d246a04c645a2fa04d9```

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成後重啟服務，並設定boot開啟：
```sh
systemctl enable openstack-ceilometer-api.service openstack-ceilometer-notification.service \
openstack-ceilometer-central.service openstack-ceilometer-collector.service \
openstack-ceilometer-alarm-evaluator.service openstack-ceilometer-alarm-notifier.service

systemctl start openstack-ceilometer-api.service openstack-ceilometer-notification.service \
openstack-ceilometer-central.service openstack-ceilometer-collector.service \
openstack-ceilometer-alarm-evaluator.service openstack-ceilometer-alarm-notifier.service

```

# 設定 Compute 監控服務
Telemetry 透過結合使用通知與proxy來收集Computer測量值。若要監控每個Compute節點，需在每個節點進行以下步驟：

### 安裝套件
透過```yum```安裝相關套件：
```sh
yum install openstack-ceilometer-compute python-ceilometerclient python-pecan
```
安裝完成後，編輯```/etc/ceilometer/ceilometer.conf```，在```[publisher] ```修改一下：
```
[publisher]
telemetry_secret = TELEMETRY_SECRET
```
> 將```TELEMETRY_SECRET```更改為剛剛產生的亂數字串 ```4d246a04c645a2fa04d9```

在```[DEFAULT]```部分，設定RabbitMQ與Keystone存取：
```
[DEFAULT]
rpc_backend = rabbit
```
在```[oslo_messaging_rabbit]```部分，設定RabbitMQ資訊：
```
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 可自行選擇更改```RABBIT_PASS```密碼

在```[keystone_authtoken]```部分，設定Keystone資訊，以及註解所有auth_host、auth_port 和auth_protocol，因為Keystone預設已包含：
```
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = ceilometer
admin_password = CEILOMETER_PASS
```
> 可自行選擇更改```CEILOMETER_PASS```密碼

在```[service_credentials]```部分，設定以下：
```
[service_credentials]
os_auth_url = http://controller:5000/v2.0
os_username = ceilometer
os_tenant_name = service
os_password = CEILOMETER_PASS
os_endpoint_type = internalURL
os_region_name = RegionOne
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
### 配置Notification
編輯```/etc/nova/nova.conf ```設定通知於```[DEFAULT]```部分：
```
[DEFAULT]
...
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state
notification_driver = messagingv2
```
完成後重啟服務，並設定boot開啟：
```sh
systemctl enable openstack-ceilometer-compute.service
systemctl start openstack-ceilometer-compute.service

systemctl restart openstack-nova-compute.service
```

# 設定 Image 監控服務
為了取得image-oriented時間與範例，需要配置Image service來發送訊息給Message bus，在```Controller```上執行這些步驟，編輯```/etc/glance/glance-api.conf```與```/etc/glance/glance-registry.conf```，在```[DEFAULT]```修改以下：
```
[DEFAULT]
...
notification_driver = messagingv2
rpc_backend = rabbit
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 可自行選擇更改```RABBIT_PASS```密碼

重啟服務：
```sh
systemctl restart openstack-glance-api.service openstack-glance-registry.service
```

# 設定  Block Storage 監控服務
為了檢索到Volume方面的事件與採樣，必須設定區塊儲存服務，以發送通知到Message bus。請在```Controller```與```Block Storage```節點上執行這些步驟。
編輯```/etc/cinder/cinder.conf```，在```[DEFAULT]```部分加入以下：
```
[DEFAULT]
...
control_exchange = cinder
notification_driver = messagingv2
```
若是```Controller``` 節點，重新啟動以下：
```sh
systemctl restart openstack-cinder-api.service openstack-cinder-scheduler.service
```
若是```Storage``` 節點，重新啟動以下：
```sh
systemctl restart openstack-cinder-volume.service
```

# 設定 Object Storage 監控服務
為了檢索到Storage的事件與採樣，設定物件儲存服務，以便發送通知到Message bus。Telemetry 服務需要使用 Reseller Admin 存取物件儲存服務，請在```Controller```完成以下步驟：
```sh
source admin-openrc.sh
```
建立 ResellerAdmin 角色：
```sh
openstack role create ResellerAdmin
```
將ResellerAdmin 角色添加到service 租戶，並賦予給ceilometer 使用者：
```sh
openstack role add --project service --user ceilometer ResellerAdmin
```
### 設定通知
在```Controller```節點上，編輯```/etc/swift/proxy-server.conf```，在```[filter:keystoneauth]```增加ResellerAdmin：
```
[filter:keystoneauth]
...
operator_roles = admin,user,ResellerAdmin
```
在``` [pipeline:main]```部分，加入ceilometer：
```
[pipeline:main]
...
pipeline = authtoken cache healthcheck keystoneauth proxy-logging ceilometer proxy-server
```
在 ```[filter:ceilometer]``` 部分，設定提醒：
```
[filter:ceilometer]
...
paste.filter_factory = ceilometermiddleware.swift:filter_factory
control_exchange = swift
url = rabbit://openstack:RABBIT_PASS@controller:5672/
driver = messagingv2
topic = notifications
log_level = WARN
```
> 可自行選擇更改```RABBIT_PASS```密碼

增加Swift的系統使用者到Ceilometer的系統群組中，來允許透過物件儲存服務存取 Telemetry 的設定檔：
```sh
usermod -a -G ceilometer swift
```
安裝```ceilometermiddleware```套件：
```sh
pip install ceilometermiddleware
```
重啟服務：
```sh
systemctl restart openstack-swift-proxy.service
```

# 驗證操作
首先我們要建立一個沒有```OS_PROJECT_DOMAIN_ID```與```OS_USER_DOMAIN_ID```的環境參數檔案```admin-ceilometer-openrc.sh```，並加入以下：
```sh
unset OS_PROJECT_DOMAIN_ID
unset OS_USER_DOMAIN_ID
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN
export OS_AUTH_URL=http://controller:35357
export OS_IMAGE_API_VERSION=2
export OS_VOLUME_API_VERSION=2
```
首先source admin使用者，進行驗證：
```sh
source admin-ceilometer-openrc.sh
```
透過ceilometer指令列出所有meter：
```sh
ceilometer meter-list
```
透過以下指令列出哪個期間的資料：
```sh
ceilometer statistics -m image.download -p 60
```
