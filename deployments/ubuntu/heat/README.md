# Heat 安裝與設定
本章節會說明與操作如何安裝協作服務到 Controller 節點上，並設置相關參數與設定。若對於 Heat 不瞭解的人，可以參考[Heat 協作整合服務章節](../../../conceptions/heat/README.md)。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Heat 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Heat 資料庫：
```sql
CREATE DATABASE heat;
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'HEAT_DBPASS';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%'  IDENTIFIED BY 'HEAT_DBPASS';
```
> 這邊```HEAT_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Heat 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Heat User
$ openstack user create --domain default --password HEAT_PASS --email heat@example.com heat

# 新增 Heat 到 Admin Role
$ openstack role add --project service --user heat admin

# 建立 Heat service
$ openstack service create --name heat --description "Orchestration" orchestration

# 建立 Heat-cfn 服務
$ openstack service create --name heat-cfn  --description "Orchestration" cloudformation

# 建立 Heat public endpoints
$ openstack endpoint create --region RegionOne \
orchestration public http://10.0.0.11:8004/v1/%\(tenant_id\)s

# 建立 Heat internal endpoints
$ openstack endpoint create --region RegionOne \
orchestration internal http://10.0.0.11:8004/v1/%\(tenant_id\)s

# 建立 Heat admin endpoints
$ openstack endpoint create --region RegionOne \
orchestration admin http://10.0.0.11:8004/v1/%\(tenant_id\)s

# 建立 Heat cloudformation public endpoints
$ openstack endpoint create --region RegionOne \
cloudformation public http://10.0.0.11:8000/v1

# 建立 Heat cloudformation internal endpoints
$ openstack endpoint create --region RegionOne \
cloudformation internal http://10.0.0.11:8000/v1

# 建立 Heat cloudformation admin endpoints
$ openstack endpoint create --region RegionOne \
cloudformation admin http://10.0.0.11:8000/v1

# 建立 Heat domain
$ openstack domain create --description "Stack projects and users" heat

# 建立 Heat domain admin user
$ openstack user create --domain heat --password HEAT_DOMAIN_PASS \
--email heat_domain_admin@example.com \
heat_domain_admin

# 新增 Heat domain admin 到 Admin role
$ openstack role add --domain heat --user heat_domain_admin admin

# 建立 Heat stack owner role
$ openstack role create heat_stack_owner

# 新增 dome 到 heat_stack_owner role
$ openstack role add --project demo --user demo heat_stack_owner

# 建立 Heat stack user role
$ openstack role create heat_stack_user
```
> 這邊```HEAT_PASS```與```HEAT_DOMAIN_PASS```可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install heat-api heat-api-cfn heat-engine python-heatclient
```
> 若有使用```magnum```的話，請務必再安裝```heat-api-cloudwatch```。

安裝完成後，編輯```/etc/heat/heat.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
heat_metadata_server_url = http://10.0.0.11:8000
heat_waitcondition_server_url = http://10.0.0.11:8000/v1/waitcondition

heat_watch_server_url = http://10.0.0.11:8003

stack_domain_admin = heat_domain_admin
stack_domain_admin_password = HEAT_DOMAIN_PASS
stack_user_domain_name = heat
```
> 這邊```HEAT_PASS```與```HEAT_DOMAIN_PASS```可以隨需求修改。

在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://heat:HEAT_DBPASS@10.0.0.11/heat
```
> 這邊```HEAT_DBPASS```可以隨需求修改。

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
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = heat
password = HEAT_PASS
```
> 這邊```HEAT_PASS```可以隨需求修改。

在```[trustee]```部分加入以下內容：
```sh
[trustee]
auth_url = http://10.0.0.11:35357
auth_plugin = password
user_domain_name = default
username = heat
password = HEAT_PASS
```
> 這邊```HEAT_PASS```可以隨需求修改。

在最底部加入以下內容：
```sh
[clients_keystone]
auth_uri = http://10.0.0.11:5000

[ec2authtoken]
auth_uri = http://10.0.0.11:5000
```

完成以上檔案設定後，即可同步資料庫來建立資料表：
```sh
$ sudo heat-manage db_sync
```

完成後，重新啟動 Heat 的服務：
```sh
sudo service heat-api restart
sudo service heat-api-cfn restart
sudo service heat-engine restart
```

最後刪除預設的 SQLite 資料庫：
```sh
$ sudo rm -f /var/lib/heat/heat.sqlite
```

### 驗證服務
首先在 ```Controller``` 節點導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

建立一個```test-stack.yml```板模檔案，在裡面加入以下內容：
```
heat_template_version: 2014-10-16
description: A simple server.

parameters:
  ImageID:
    type: string
    description: Image use to boot a server
  NetID:
    type: string
    description: Network ID for the server

resources:
  server:
    type: OS::Nova::Server
    properties:
      image: { get_param: ImageID }
      flavor: m1.tiny
      networks:
      - network: { get_param: NetID }

outputs:
  private_ip:
    description: IP address of the server in the private network
    value: { get_attr: [ server, first_address ] }
```

完成後，可以透過 Heat client 程式來使用板模，指令如下：
```sh
$ NET_ID=$(nova net-list | awk '/ admin-net / { print $2 }')
$ heat stack-create -f test-stack.yml -P "ImageID=cirros-0.3.4-x86_64;NetID=$NET_ID" testStack
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| 477d96b4-d547-4069-938d-32ee990834af | testStack  | CREATE_IN_PROGRESS | 2014-04-06T15:11:01Z |
+--------------------------------------+------------+--------------------+----------------------+
```
> P.S 這邊```NET_ID```要注意執行指令時，抓取的網路是否存在。

透過 Heat client 程式來查看列表，指令如下：
```sh
$ heat stack-list
+--------------------------------------+------------+-----------------+----------------------+
| id                                   | stack_name | stack_status    | creation_time        |
+--------------------------------------+------------+-----------------+----------------------+
| b1e5e05e-8932-4b2e-beba-97a5ba36a3f3 | testStack  | CREATE_COMPLETE | 2015-07-03T07:08:28Z |
+--------------------------------------+------------+-----------------+----------------------+
```

透過 Heat client 程式來查看服務列表，指令如下：
```sh
$ heat service-list
+------------+-------------+--------------------------------------+------------+--------+----------------------------+--------+
| hostname   | binary      | engine_id                            | host       | topic  | updated_at                 | status |
+------------+-------------+--------------------------------------+------------+--------+----------------------------+--------+
| controller | heat-engine | 3e85d1ab-a543-41aa-aa97-378c381fb958 | controller | engine | 2015-10-13T14:16:06.000000 | up     |
| controller | heat-engine | 45dbdcf6-5660-4d5f-973a-c4fc819da678 | controller | engine | 2015-10-13T14:16:06.000000 | up     |
| controller | heat-engine | 51162b63-ecb8-4c6c-98c6-993af899c4f7 | controller | engine | 2015-10-13T14:16:06.000000 | up     |
| controller | heat-engine | 8d7edc6d-77a6-460d-bd2a-984d76954646 | controller | engine | 2015-10-13T14:16:06.000000 | up     |
+------------+-------------+--------------------------------------+------------+--------+----------------------------+--------+
```
