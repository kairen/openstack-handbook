# Sahara 安裝與設定
本章節會說明與操作如何安裝資料處理服務到 Controller 節點上，並設置相關參數與設定。若對於 Sahara 不瞭解的人，可以參考[Sahara 資料處理服務](../../../conceptions/sahara/README.md)

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)
    - [建立 Sahara 服務映像檔](#建立-sahara-服務映像檔)
    - [建立 Sahara Plugin](#建立-sahara-plugin)
    - [建立 Node Group Templates](#建立-node-group-templates)
    - [建立 Cluster Templates](#建立-cluster-templates)
    - [建立 Cluster](#建立-cluster)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Sahara 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Sahara 資料庫：
```sql
CREATE DATABASE sahara;
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'localhost'  IDENTIFIED BY ' SAHARA_DBPASS';
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'%'  IDENTIFIED BY 'SAHARA_DBPASS';
```
> 這邊```SAHARA_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Sahara 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Sahara User
$ openstack user create --domain default \
--password SAHARA_PASS --email sahara@example.com sahara

# 建立 Sahara Role
$ openstack role add --project service --user sahara admin

# 建立 Sahara Service
$ openstack service create --name sahara \
--description "Data Processing Service" data_processing

# 建立 Sahara public endpoints
$ openstack endpoint create --region RegionOne \
data_processing public http://10.0.0.11:8386/v1.1/%\(tenant_id\)s

# 建立 Sahara internal endpoints
$ openstack endpoint create --region RegionOne \
data_processing internal http://10.0.0.11:8386/v1.1/%\(tenant_id\)s

# 建立 Sahara admin endpoints
$ openstack endpoint create --region RegionOne \
data_processing admin http://10.0.0.11:8386/v1.1/%\(tenant_id\)s
```
> 這邊```SAHARA_PASS```可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install sahara-api sahara-common sahara-engine python-sahara
```

安裝完成後，編輯```/etc/sahara/sahara.conf```，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
infrastructure_engine = heat
use_neutron = true
use_floating_ip = true
use_syslog = False
plugins = vanilla,hdp,cdh,mapr,spark,storm,fake
debug = True
verbose = True
rpc_backend = rabbit
```

接下來，在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://sahara:SAHARA_DBPASS@10.0.0.11/sahara
```
> 這邊```SAHARA_DBPASS```可以隨需求修改。

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
project_domain_name = default
project_name = service
user_domain_name = default
admin_tenant_name = default
auth_type = password
admin_user = sahara
admin_password = SAHARA_PASS
username = sahara
password = SAHARA_PASS
```
> 這邊```SAHARA_PASS```可以隨需求修改。

在```[oslo_policy]```部分加入以下內容：
```
[oslo_policy]
policy_file = /etc/sahara/policy.json
```

完成所有設定後，即可同步資料庫來建立 Sahara 資料表：
```sh
$ sudo sahara-db-manage upgrade head
```

重新啟動所有 Sahara 服務：
```sh
sudo service sahara-api restart
sudo service sahara-engine restart
```

# 驗證服務
首先回到 ```Controller``` 節點並接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

透過 Sahara client 來查看服務列表，如以下方式：
```sh
$ sahara cluster-list
+------+----+--------+------------+
| name | id | status | node_count |
+------+----+--------+------------+
+------+----+--------+------------+
```

### 建立 Sahara 服務映像檔
在建立一個叢集之前，我們要先提供 Sahara 服務使用的映像檔，這邊採用 Sahara image elements 進行建置，首先下載該套件：
```sh
$ git clone https://github.com/openstack/sahara-image-elements
$ cd sahara-image-elements
```

然後安裝與更新使用的工具：
```sh
$ sudo pip install --upgrade  tox
```

最後透過 tox 來提供虛擬環境，並建置一個映像檔：
```sh
$ sudo tox -e venv -- sahara-image-create -p cloudera -i ubuntu -v 5.3
```
> ```-p``` 為插件種類，有以下：
[vanilla|spark|hdp|cloudera|storm|mapr|plain]

> ```-i``` 為作業系統，有以下：
[ubuntu|fedora|centos|centos7]

> ```-v``` 為版本號，如以下範例：
[1|2|2.6|4|5.0|5.3|5.4]

> 更多的參數可以到 [sahara-image-elements](https://github.com/openstack/sahara-image-elements/blob/master/diskimage-create/README.rst) 查看。

> 建置 Cloudera Plugin 可以參考 [cdh-on-openstack-with-sahara](http://blog.cloudera.com/blog/2015/05/how-to-get-started-with-cdh-on-openstack-with-sahara/)。

### 建立 Sahara Plugin
這邊為了提升佈署的效率，直接抓取官方建置好的映像檔來提供服務，首先下載映像檔：
```sh
$ wget http://sahara-files.mirantis.com/images/upstream/mitaka/sahara-mitaka-vanilla-hadoop-2.7.1-ubuntu.qcow2
```
> 更多的映像檔可以參考 [Sahara Images](http://sahara-files.mirantis.com/images/upstream/mitaka/)。

使用 Glance client 來上傳 Manila Service 映像檔：
```sh
$ glance image-create --name "sahara-vanilla-ubuntu" \
--file sahara-mitaka-vanilla-hadoop-2.7.1-ubuntu.qcow2 \
--disk-format qcow2 --container-format bare \
--visibility public --progress
```

接著透過 OpenStack client 註冊映像檔：
```sh
$ openstack dataprocessing image register sahara-vanilla-ubuntu --username ubuntu
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| Description | None                                 |
| Id          | fc64b0bf-5c1f-4fbc-a76f-70df53267d6e |
| Name        | sahara-vanilla-ubuntu                |
| Status      | ACTIVE                               |
| Tags        |                                      |
| Username    | ubuntu                               |
+-------------+--------------------------------------+
```

然後標示 Tag 來區別版本：
```sh
$ openstack dataprocessing image tags add sahara-vanilla-ubuntu --tags vanilla <plugin_version>
+-------------+--------------------------------------+
| Field       | Value                                |
+-------------+--------------------------------------+
| Description | None                                 |
| Id          | fc64b0bf-5c1f-4fbc-a76f-70df53267d6e |
| Name        | sahara-vanilla-ubuntu                |
| Status      | ACTIVE                               |
| Tags        | 2.7.1, vanilla                       |
| Username    | ubuntu                               |
+-------------+--------------------------------------+
```
> 這邊```<plugin_version>```為 2.7.1。

### 建立 Node Group Templates
節點群組是定義一些節點角色的板模配置。這邊會說明如何建立節點板模，並指配對應的角色功能。首先我們先建立名稱為```vanilla-master.json```的 Hadoop Master 角色板模：
```json
{
    "plugin_name": "vanilla",
    "hadoop_version": "2.7.1",
    "node_processes": [
        "namenode",
        "resourcemanager"
    ],
    "name": "vanilla-master",
    "floating_ip_pool": "4e737d99-0427-4efd-a33d-d58d44b0fcf8",
    "flavor_id": "2",
    "auto_security_group": true
}
```
> 這邊```hadoop_version```為上面建立的版本。

> 這邊```floating_ip_pool```為 Provides Network ID。

接著建立名稱為```vanilla-worker.json```的 Hadoop Slave 角色板模：
```json
{
    "plugin_name": "vanilla",
    "hadoop_version": "2.7.1",
    "node_processes": [
        "nodemanager",
        "datanode"
    ],
    "name": "vanilla-worker",
    "floating_ip_pool": "4e737d99-0427-4efd-a33d-d58d44b0fcf8",
    "flavor_id": "2",
    "auto_security_group": true
}
```
> 這邊```hadoop_version```為上面建立的版本。

> 這邊```floating_ip_pool```為 Provides Network ID。

完成上述檔案後，透過 OpenStack client 來建立 Master 板模實例：
```sh
$ openstack dataprocessing node group template create --json vanilla-master.json
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| Auto security group | True                                 |
| Availability zone   | None                                 |
| Flavor id           | 2                                    |
| Floating ip pool    | 4e737d99-0427-4efd-a33d-d58d44b0fcf8 |
| Id                  | 7909f06b-5551-4e97-acb2-eb0fee6947e1 |
| Is default          | False                                |
| Is protected        | False                                |
| Is proxy gateway    | False                                |
| Is public           | False                                |
| Name                | vanilla-master                       |
| Node processes      | namenode, resourcemanager            |
| Plugin name         | vanilla                              |
| Security groups     | None                                 |
| Use autoconfig      | True                                 |
| Version             | 2.7.1                                |
| Volumes per node    | 0                                    |
+---------------------+--------------------------------------+
```

再透過 OpenStack client 來建立 Slave 板模實例：
```sh
$ openstack dataprocessing node group template create --json vanilla-worker.json
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| Auto security group | True                                 |
| Availability zone   | None                                 |
| Flavor id           | 2                                    |
| Floating ip pool    | 4e737d99-0427-4efd-a33d-d58d44b0fcf8 |
| Id                  | 22374a26-be19-4e9c-9f36-3295b300e604 |
| Is default          | False                                |
| Is protected        | False                                |
| Is proxy gateway    | False                                |
| Is public           | False                                |
| Name                | vanilla-worker                       |
| Node processes      | datanode, nodemanager                |
| Plugin name         | vanilla                              |
| Security groups     | None                                 |
| Use autoconfig      | True                                 |
| Version             | 2.7.1                                |
| Volumes per node    | 0                                    |
+---------------------+--------------------------------------+
```

完成後可以透過指令查看是否有正常建立：
```sh
$ openstack dataprocessing node group template list
+----------------+--------------------------------------+-------------+---------+
| Name           | Id                                   | Plugin name | Version |
+----------------+--------------------------------------+-------------+---------+
| vanilla-worker | 22374a26-be19-4e9c-9f36-3295b300e604 | vanilla     | 2.7.1   |
| vanilla-master | 7909f06b-5551-4e97-acb2-eb0fee6947e1 | vanilla     | 2.7.1   |
+----------------+--------------------------------------+-------------+---------+
```

### 建立 Cluster Template
接著我們要建立一個用來描述叢集的節點群組板模，首先建立一個名稱為```vanilla-cluster-template.json```的叢集板模：
```json
{
    "plugin_name": "vanilla",
    "hadoop_version": "2.7.1",
    "node_groups": [
        {
            "name": "worker",
            "count": 2,
            "node_group_template_id": "22374a26-be19-4e9c-9f36-3295b300e604"
        },
        {
            "name": "master",
            "count": 1,
            "node_group_template_id": "7909f06b-5551-4e97-acb2-eb0fee6947e1"
        }
    ],
    "name": "vanilla-hadoop-cluster",
    "cluster_configs": {}
}
```
> 這邊```hadoop_version```為上面建立的版本。

> 這邊```node_group_template_id```為 Node Group ID。

完成上述檔案後，透過 OpenStack client 來建立 Cluster 板模實例：
```sh
$ openstack dataprocessing cluster template create --json vanilla-cluster-template.json
+----------------+--------------------------------------+
| Field          | Value                                |
+----------------+--------------------------------------+
| Anti affinity  |                                      |
| Description    | None                                 |
| Id             | 2297881e-1590-4ae2-ad2a-65130bfb9fa5 |
| Is default     | False                                |
| Is protected   | False                                |
| Is public      | False                                |
| Name           | vanilla-hadoop-cluster               |
| Node groups    | worker:2, master:1                   |
| Plugin name    | vanilla                              |
| Use autoconfig | True                                 |
| Version        | 2.7.1                                |
+----------------+--------------------------------------+
```

完成後可以透過指令查看是否有正常建立：
```sh
$ openstack dataprocessing cluster template list
+------------------------+--------------------------------------+-------------+---------+
| Name                   | Id                                   | Plugin name | Version |
+------------------------+--------------------------------------+-------------+---------+
| vanilla-hadoop-cluster | 2297881e-1590-4ae2-ad2a-65130bfb9fa5 | vanilla     | 2.7.1   |
+------------------------+--------------------------------------+-------------+---------+
```

### 建立 Cluster
現在上述的步驟都完成後，就可以建立一個叢集實例了。首先建立一個名稱為```vanilla-cluster-instance.json```的叢集實例資訊配置檔：
```json
{
    "name": "vanila-cluster",
    "plugin_name": "vanilla",
    "hadoop_version": "2.7.1",
    "cluster_template_id": "2297881e-1590-4ae2-ad2a-65130bfb9fa5",
    "user_keypair_id": "kylebai",
    "default_image_id": "fc64b0bf-5c1f-4fbc-a76f-70df53267d6e",
    "neutron_management_network": "1d280d77-d37d-44d6-b020-53ebd896baa4"
}
```
> 這邊```hadoop_version```為上面建立的版本。

> 這邊```cluster_template_id```為上面建立的 Template ID。

> 這邊```user_keypair_id```為自己的 SSH Keypair ID。

> 這邊```default_image_id```為 Hadoop 映像檔 ID。

> 這邊```neutron_management_network```為自己的 Tanent Network ID。

完成上述檔案後，透過 OpenStack client 來建立 Cluster 板模實例：
```sh
$ openstack dataprocessing cluster create --json vanilla-cluster-instance.json
+----------------------------+--------------------------------------+
| Field                      | Value                                |
+----------------------------+--------------------------------------+
| Anti affinity              |                                      |
| Cluster template id        | 2297881e-1590-4ae2-ad2a-65130bfb9fa5 |
| Description                | None                                 |
| Id                         | 0fe3deea-e452-427d-bfb2-0d3a3e9bb33b |
| Image                      | fc64b0bf-5c1f-4fbc-a76f-70df53267d6e |
| Is protected               | False                                |
| Is public                  | False                                |
| Is transient               | False                                |
| Name                       | vanila-cluster                       |
| Neutron management network | 1d280d77-d37d-44d6-b020-53ebd896baa4 |
| Node groups                | master:1, worker:2                   |
| Plugin name                | vanilla                              |
| Status                     | Validating                           |
| Use autoconfig             | True                                 |
| User keypair id            | kylebai                              |
| Version                    | 2.7.1                                |
+----------------------------+--------------------------------------+
```
