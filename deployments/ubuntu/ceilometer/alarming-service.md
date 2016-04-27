# Alarming Service
這部分將介紹如何安裝 Ceilometer 的警報服務於 ```Controller```節點上，該專案名稱為 aodh。

- [Alarming 安裝前準備](#alarming-安裝前準備)
- [Alarming 套件安裝與設定](#alarming-套件安裝與設定)
- [驗證服務](#驗證服務)

### Alarming 安裝前準備
在開始安裝前，要預先建立一個資料庫給 aodh 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 aodh 資料庫：
```sql
CREATE DATABASE aodh;
GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'localhost' IDENTIFIED BY 'AODH_DBPASS';
GRANT ALL PRIVILEGES ON aodh.* TO 'aodh'@'%' IDENTIFIED BY 'AODH_DBPASS';
```
> 這邊```AODH_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 aodh 的使用者、Service 以及 API Endpoint：
```sh
# 建立 aodh user
$ openstack user create --domain default --password AODH_PASS --email aodh@example.com aodh

# 新增 aodh 到 Admin Role
$ openstack role add --project service --user aodh admin

# 建立 aodh service
$ openstack service create --name aodh \
--description "Telemetry" alarming

# 建立 aodh v1 public endpoints
$ openstack endpoint create --region RegionOne \
alarming public http://10.0.0.11:8042

# 建立 aodh v1 internal endpoints
$ openstack endpoint create --region RegionOne \
alarming internal http://10.0.0.11:8042

# 建立 aodh v1 admin endpoints
$ openstack endpoint create --region RegionOne \
alarming admin http://10.0.0.11:8042
```

### Alarming 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install aodh-api aodh-evaluator aodh-notifier \
aodh-listener aodh-expirer python-ceilometerclient
```

安裝完成後，編輯```/etc/aodh/aodh.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
auth_strategy = keystone
```

在```[database]```部分修改使用以下方式：
```
[database]
# connection=mongodb://localhost:27017/aodh
connection = mysql+pymysql://aodh:AODH_DBPASS@10.0.0.11/aodh
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
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = aodh
password = AODH_PASS
```
> 這邊```AODH_PASS```可以隨需求修改。

在```[service_credentials]```部分加入以下內容：
```
[service_credentials]
os_auth_url = http://10.0.0.11:5000/v2.0
os_username = aodh
os_tenant_name = service
os_password = AODH_PASS
interface = internalURL
region_name = RegionOne
```
> 這邊```AODH_PASS```可以隨需求修改。

最後編輯```/etc/aodh/api_paste.ini```，在```[filter:authtoken]```修改以下內容：
```
[filter:authtoken]
...
oslo_config_project = aodh
```

完成以上後即可同步資料庫來建立資料表：
```sh
$ sudo aodh-dbsync
```

完成後即可重新啟動服務：
```sh
sudo service aodh-api restart
sudo service aodh-evaluator restart
sudo service aodh-notifier restart
sudo service aodh-listener restart
```

# 驗證服務
導入該環境參數，來透過 Ceilometer client 查看服務狀態：
```sh
$ . admin-openrc
```

這邊可以透過 Ceilometer client 建立一個 alarm，如以下方式：
```sh
$ INSTANCE_ID=<YOUR_INSTANCE_ID>
$ ceilometer alarm-threshold-create --name cpu_hi \
--description 'instance running hot' \
--meter-name cpu_util --threshold 10.0 \
--comparison-operator gt --statistic avg \
--period 600 --evaluation-periods 3 \
--alarm-action 'log://' \
--query resource_id=${INSTANCE_ID}
```
> 更多資訊可以查看 [Alarms](http://docs.openstack.org/admin-guide/telemetry-alarms.html)。

接著就就可以透過 Ceilometer client 查看所有 alarm：
```sh
$ ceilometer alarm-list
+--------------------------------------+--------+-------------------+----------+---------+------------+--------------------------------------+------------------+
| Alarm ID                             | Name   | State             | Severity | Enabled | Continuous | Alarm condition                      | Time constraints |
+--------------------------------------+--------+-------------------+----------+---------+------------+--------------------------------------+------------------+
| 3fe8cde8-627b-47a1-b5af-8f923478a893 | cpu_hi | insufficient data | low      | True    | False      | avg(cpu_util) > 10.0 during 3 x 600s | None             |
+--------------------------------------+--------+-------------------+----------+---------+------------+--------------------------------------+------------------+
```
