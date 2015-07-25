# Heat 安裝與設定
本章節會說明與操作如何安裝```Orchestration```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Heat不瞭解的人，可以參考[Heat 協調整合套件章節](heat.html)

# Controller節點安裝與設置
### 安裝前準備
我們需要在Database底下建立儲存 Heat 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Heat 資料庫與使用者：
```sql
CREATE DATABASE heat;
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' IDENTIFIED BY 'HEAT_DBPASS';
GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%'  IDENTIFIED BY 'HEAT_DBPASS';
```
> 這邊若```HEAT_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫。之後我們要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Heat User
openstack user create --password HEAT_PASS --email heat@example.com heat
# 建立 Heat Role
openstack role add --project service --user heat admin
# 建立 Heat stack owner
openstack role create heat_stack_owner
# 建立 Owner Role
openstack role add --project demo --user demo heat_stack_owner
# 建立 Heat stack user
openstack role create heat_stack_user
# 建立 Heat service
openstack service create --name heat --description "Orchestration" orchestration
# 建立 Heat-cfn 服務
openstack service create --name heat-cfn  --description "Orchestration" cloudformation
# 建立 Heat orchestration endpoints
openstack endpoint create  --publicurl http://controller:8004/v1/%\(tenant_id\)s  --internalurl http://controller:8004/v1/%\(tenant_id\)s --adminurl http://controller:8004/v1/%\(tenant_id\)s  --region RegionOne  orchestration
# 建立 Heat cloudformation endpoinst
openstack endpoint create  --publicurl http://controller:8000/v1  --internalurl http://controller:8000/v1 --adminurl http://controller:8000/v1  --region RegionOne cloudformation
```
> 這邊若```HEAT_PASS```要更改的話，可以更改。

### 安裝與設置Heat套件
首先透過```apt-get```安裝相關套件：
```sh
sudo apt-get install heat-api heat-api-cfn heat-engine python-heatclient
```
編輯```/etc/heat/heat.conf```，在```[database]```部分加入：
```
[database]
...
connection = mysql://heat:HEAT_DBPASS@controller/heat
```
> 這邊若```HEAT_DBPASS```有更改的話，請記得更改。

在```[DEFAULT]```部分，設定RabbitMQ、Keystone存取、metadata與url、heat 認證服務：
```
[DEFAULT]
...
rpc_backend = rabbit
heat_metadata_server_url = http://controller:8000
heat_waitcondition_server_url = http://controller:8000/v1/waitcondition

stack_domain_admin = heat_domain_admin
stack_domain_admin_password = HEAT_PASS
stack_user_domain_name = heat_user_domain
```
> 這邊若```HEAT_PASS```有更改的話，請記得更改。

在```[oslo_messaging_rabbit]```部分，設定RabbitMQ：
```
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊若```RABBIT_PASS```有更改的話，請記得更改。

在```[keystone_authtoken]```與```[ec2authtoken]```部分，設定keystone存取以及註解所有auth_host、auth_port 和auth_protocol，因為Keystone預設已包含：
```
[keystone_authtoken]
auth_uri = http://controller:5000/v2.0
identity_uri = http://controller:35357
admin_tenant_name = service
admin_user = heat
admin_password = HEAT_PASS

[ec2authtoken]
auth_uri = http://controller:5000/v2.0
```
> 這邊若```HEAT_PASS```有更改的話，請記得更改。

最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成後建立Heat Domain：
```sh
heat-keystone-setup-domain  --stack-user-domain-name heat_user_domain --stack-domain-admin heat_domain_admin --stack-domain-admin-password HEAT_DOMAIN_PASS
```
會取得以下資訊，並更新```/etc/heat/heat.conf```的```[DEFAULT]```以下資訊：
```
stack_user_domain_id=1a6d106bf43641f2bdcb7ded3a49e6a2
stack_domain_admin=heat_domain_admin
stack_domain_admin_password=HEAT_DOMAIN_PASS
```
以上都完成後，同步資料庫：
```sh
sudo heat-manage db_sync
```
重啟服務，並刪除預設的SQLite資料庫：
```sh
sudo service heat-api restart
sudo service heat-api-cfn restart
sudo service heat-engine restart

sudo rm -f /var/lib/heat/heat.sqlite
```
# 驗證操作
這部分我們將驗證 Orchestrationg 是否正確運作。首先導入```admin```環境變數檔案：
```
source admin-openrc.sh
```
建立一個檔案```test-stack.yml```，並加入以下：
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
使用```heat stack-create```指令建立板模：
```sh
NET_ID=$(nova net-list | awk '/ demo-net / { print $2 }')
heat stack-create -f test-stack.yml -P "ImageID=cirros-0.3.4-x86_64;NetID=$NET_ID" testStack
```
成功會看到類似以下資訊：
```
+--------------------------------------+------------+--------------------+----------------------+
| id                                   | stack_name | stack_status       | creation_time        |
+--------------------------------------+------------+--------------------+----------------------+
| 477d96b4-d547-4069-938d-32ee990834af | testStack  | CREATE_IN_PROGRESS | 2014-04-06T15:11:01Z |
+--------------------------------------+------------+--------------------+----------------------+
```
透過```heat stack-list```來查看列表：
```sh
heat stack-list
```
成功會看到類似以下資訊：
```
+--------------------------------------+------------+-----------------+----------------------+
| id                                   | stack_name | stack_status    | creation_time        |
+--------------------------------------+------------+-----------------+----------------------+
| b1e5e05e-8932-4b2e-beba-97a5ba36a3f3 | testStack  | CREATE_COMPLETE | 2015-07-03T07:08:28Z |
+--------------------------------------+------------+-----------------+----------------------+
```
