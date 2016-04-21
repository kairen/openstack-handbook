# Magnum 安裝與設定
本章節會說明與操作如何安裝容器服務到 Controller 節點上，並設置相關參數與設定。。若對於 Murano 不瞭解的人，可以參考[Magnum 容器即服務](../../../conceptions/magnum/README.md)

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)
    - [佈署 Kubernetes](#佈署-kubernetes)
    - [佈署 Mesos](#佈署-mesos)
    - [佈署 Docker Swarm](#佈署-docker-swarm)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Magnum 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Magnum 資料庫：
```sql
CREATE DATABASE magnum;
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'localhost'  IDENTIFIED BY 'MAGNUM_DBPASS';
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%'  IDENTIFIED BY 'MAGNUM_DBPASS';
```
> 這邊```MAGNUM_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Magnum 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Magnum User
$ openstack user create --domain default --password MAGNUM_PASS --email magnum@example.com magnum

# 建立 Magnum Role
$ openstack role add --project service --user magnum admin

# 建立 Magnum service
$ openstack service create --name magnum \
--description "Container Service" container

# 建立 Magnum public endpoints
$ openstack endpoint create --region RegionOne \
container public http://10.0.0.11:9511/v1

# 建立 Magnum internal endpoints
$ openstack endpoint create --region RegionOne \
container internal http://10.0.0.11:9511/v1

# 建立 Magnum admin endpoints
$ openstack endpoint create --region RegionOne \
container admin http://10.0.0.11:9511/v1
```
> 這邊```MAGNUM_PASS```可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install magnum-api magnum-conductor
```

安裝完成後，編輯```/etc/magnum/magnum.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
debug = True

rpc_backend = rabbit
auth_strategy = keystone
```

接下來，在```[database]```部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://magnum:MAGNUM_DBPASS@10.0.0.11/magnum
```
> 這邊```MAGNUM_DBPASS```可以隨需求修改。

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
username = magnum
password = MAGNUM_PASS
```
> 這邊```MAGNUM_PASS```可以隨需求修改。

在```[engine]```部分加入以下內容：
```
[api]
port = 9511
host = 10.0.0.11
```

在```[oslo_policy]```部分加入以下內容：
```
[oslo_policy]
policy_file = /etc/magnum/policy.json
```

在```[oslo_concurrency]```部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/magnum/tmp
```

在```[certificates]```部分加入以下內容：
```
[certificates]
cert_manager_type = local
storage_path = /var/lib/magnum/certificates/
```

在```[cinder_client]```部分加入以下內容：
```
[cinder_client]
region_name = RegionOne
```

完成所有設定後，即可同步資料庫來建立 Magnum 資料表：
```sh
$ sudo magnum-db-manage upgrade
```

重新啟動所有 Magnum 服務：
```sh
sudo service magnum-api restart
sudo service magnum-conductor restart
```

### 驗證服務
首先回到 ```Controller``` 節點並接著導入 ```admin``` 帳號來驗證服務：
```sh
$ . admin-openrc
```

透過 Magnum client 來查看服務列表，如以下方式：
```sh
$ magnum service-list
+----+------------+------------------+-------+
| id | host       | binary           | state |
+----+------------+------------------+-------+
| 1  | controller | magnum-conductor | up    |
+----+------------+------------------+-------+
```

接著下載將使用的 Magnum service 映像檔，以下是提供 k8s 與 docker swarm 使用：
```sh
$ wget https://fedorapeople.org/groups/magnum/fedora-23-atomic-20160405.qcow2
```
> 其他版本可以參考 [Magnum image elements](https://fedorapeople.org/groups/magnum/)。

透過 Glance client 來上傳映像檔到 OpenStack：
```sh
$ glance image-create --name fedora-23-atomic \
--disk-format=qcow2 --container-format=bare \
--property os_distro='fedora-atomic' \
--file fedora-23-atomic-20160405.qcow2 \
--visibility public --progress
```

若 Mesos 則下載該映像檔：
```sh
$ wget https://fedorapeople.org/groups/magnum/ubuntu-14.04.3-mesos-0.25.0.qcow2
```
> 其他版本可以參考 [Magnum image elements](https://fedorapeople.org/groups/magnum/)。

透過 Glance client 來上傳映像檔到 OpenStack：
```sh
$ glance image-create --name ubuntu-mesos \
--disk-format=qcow2 --container-format=bare \
--property os_distro='ubuntu' \
--file ubuntu-14.04.3-mesos-0.25.0.qcow2 \
--visibility public --progress
```

### 佈署 Kubernetes
首先透過 Magnum client 建立 baymodel，來提供虛擬機的規格：
```
$ magnum baymodel-create --name k8smodel \
--image-id fedora-23-atomic \
--keypair-id KEY_ID \
--external-network-id EXTERNAL_NETWORK \
--dns-nameserver 8.8.8.8 \
--flavor-id m1.medium \
--docker-volume-size 5 \
--network-driver flannel \
--coe kubernetes
```
> 這邊```KEY_ID```請取代成自己個 Key Pair Name。```EXTERNAL_NETWORK```取代成 Provider Network Name。

成功的話，會看到類似以下結果：
```
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| uuid                | 7c0042f1-0ddc-4b7a-a246-1d6f67a8204e |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kylebai                              |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | 5                                    |
| server_type         | vm                                   |
| external_network_id | ext-net                              |
| cluster_distro      | fedora-atomic                        |
| image_id            | fedora-23-atomic                     |
| volume_driver       | None                                 |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | k8smodel                             |
| created_at          | 2016-04-21T10:55:11+00:00            |
| network_driver      | flannel                              |
| fixed_network       | None                                 |
| coe                 | kubernetes                           |
| flavor_id           | m1.medium                            |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```

完成後，就可以建立實例的 bay 來部署叢集：
```sh
$ magnum bay-create --name k8sbay \
--baymodel k8smodel --node-count 1
```
> 若要更新節點數可以用該指令：
```sh
$ magnum bay-update k8sbay replace node_count=2
```

> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。

### 佈署 Mesos
採用 Mesos 可以使用以下指令建立 baymodel：
```
$ magnum baymodel-create --name mesosbaymodel \
--image-id ubuntu-mesos \
--keypair-id KEY_ID \
--external-network-id EXTERNAL_NETWORK \
--dns-nameserver 8.8.8.8 \
--flavor-id m1.small \
--coe mesos
```
> 這邊```KEY_ID```請取代成自己個 Key Pair Name。```EXTERNAL_NETWORK```取代成 Provider Network Name。

成功的話，會看到類似以下結果：
```
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| uuid                | 88134520-7780-43e6-9a6b-62b0d0c7df2b |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kylebai                              |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | None                                 |
| server_type         | vm                                   |
| external_network_id | ext-net                              |
| cluster_distro      | ubuntu                               |
| image_id            | ubuntu-mesos                         |
| volume_driver       | None                                 |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | mesosbaymodel                        |
| created_at          | 2016-04-21T10:58:49+00:00            |
| network_driver      | docker                               |
| fixed_network       | None                                 |
| coe                 | mesos                                |
| flavor_id           | m1.small                             |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```

完成後，就可以建立實例的 bay 來部署叢集：
```sh
$ magnum bay-create --name mesosbay \
--baymodel mesosbaymodel --node-count 2
```
> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。

### 佈署 Docker Swarm
採用 Docker Swarm 可以使用以下指令建立 baymodel：
```
$ magnum baymodel-create --name swarmbaymodel \
--image-id fedora-23-atomic \
--keypair-id KEY_ID \
--external-network-id EXTERNAL_NETWORK \
--dns-nameserver 8.8.8.8 \
--flavor-id m1.small \
--coe swarm
```

若成功的話，會顯示類似如下資訊：
```sh
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| uuid                | 6308107c-6cd1-4421-b054-caa06abf3a86 |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kylebai                              |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | None                                 |
| server_type         | vm                                   |
| external_network_id | ext-net                              |
| cluster_distro      | fedora-atomic                        |
| image_id            | fedora-23-atomic                     |
| volume_driver       | None                                 |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | swarmbaymodel                        |
| created_at          | 2016-04-21T11:00:55+00:00            |
| network_driver      | docker                               |
| fixed_network       | None                                 |
| coe                 | swarm                                |
| flavor_id           | m1.small                             |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```

完成後，就可以建立實例的 bay 來部署叢集：
```sh
$ magnum bay-create --name swarmbay \
--baymodel swarmbaymodel --node-count 2
```
> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。
