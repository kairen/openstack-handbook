# Magnum 安裝與設定
本章節會說明與操作如何安裝```Container as a Service```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於 Magnum 不瞭解的人，可以參考 [Magnum 容器服務套件章節](http://kairen.gitbooks.io/openstack/content/magnum/index.html)

### 安裝前準備
我們需要在 Database 底下建立儲存 Magnum 資料庫，利用```mysql```指令進入：
```sql
CREATE DATABASE magnum;
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'localhost'  IDENTIFIED BY 'MAGNUM_DBPASS';
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%'  IDENTIFIED BY 'MAGNUM_DBPASS';
```
> 這邊若```MAGNUM_DBPASS```要更改的話，可以更改。

接著要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Magnum User
openstack user create --password MAGNUM_PASS --email magnum@example.com magnum

# 建立 Magnum Role
openstack role add --project service --user magnum admin

# 建立 Magnum service
openstack service create --name magnum  --description "magnum Container Service" container

# 建立 Magnum URL
openstack endpoint create \
--publicurl http://10.0.0.11:9511/v1 \
--internalurl http://10.0.0.11:9511/v1 \
--adminurl http://10.0.0.11:9511/v1 \
--region RegionOne container
```
> 這邊若```MAGNUM_PASS```要更改的話，可以更改。

建立 magnum 使用者與相關檔案放置用目錄：
```sh
for SERVICE in magnum
do
useradd --home-dir "/var/lib/$SERVICE" \
    --create-home \
    --system \
    --shell /bin/false \
    $SERVICE

#Create essential dirs

mkdir -p /var/log/$SERVICE
mkdir -p /etc/$SERVICE

#Set ownership of the dirs

chown -R $SERVICE:$SERVICE /var/log/$SERVICE
chown -R $SERVICE:$SERVICE /var/lib/$SERVICE
chown $SERVICE:$SERVICE /etc/$SERVICE
done
```

### 安裝 Container service
當 endpoint 與 mysql 建立完成，就可以開始進行安裝，由於 Magnum 目前還沒有相關 .deb 安裝套件，故使用 git 的 repository 安裝：
```sh
cd ~/
git clone https://git.openstack.org/openstack/magnum
cd magnum
sudo pip install -e .
```
建立 ```/etc/magnum``` 放置設定檔：
```sh
sudo mkdir -p /etc/magnum
```

產生 magnum 的設定檔案，可以透過```tox```產生檔案：
```sh
tox -e genconfig
```

複製相關設定與參數檔案於 ```/etc/magnum```：
```sh
sudo cp etc/magnum/magnum.conf.sample /etc/magnum/magnum.conf
sudo cp etc/magnum/policy.json /etc/magnum/policy.json
sudo chown magnum:magnum -R /etc/magnum
```
配置設定檔案```/etc/magnum/magnum.conf```的```[DEFAULT]```：
```sh
[DEFAULT]
logging_exception_prefix = %(color)s%(asctime)s.%(msecs)03d TRACE %(name)s %(instance)s
logging_debug_format_suffix = from (pid=%(process)d) %(funcName)s %(pathname)s:%(lineno)d
logging_default_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [-%(color)s] %(instance)s%(color)s%(message)s
logging_context_format_string = %(asctime)s.%(msecs)03d %(color)s%(levelname)s %(name)s [%(request_id)s %(user_name)s %(project_name)s%(color)s] %(instance)s%(color)s%(message)s
debug = True
```

在 ```[database]```加入連接的資料庫：
```sh
[database]
connection = mysql+pymysql://magnum:MAGNUM_DBPASS@10.0.0.11/magnum?charset=utf8
```
在```[oslo_messaging_rabbit]```加入 AMQP 帳號與密碼等：
```sh
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
在 ```[keystone_authtoken]```加入 keystone 驗證：
```sh
[keystone_authtoken]
auth_version = v3
memcache_servers = localhost:11211
auth_uri = http://10.0.0.11:5000/v3
project_domain_id = default
project_name = service
user_domain_id = default
username = magnum
password = MAGNUM_PASS

auth_url = http://10.0.0.11:35357
auth_type = password
admin_tenant_name = service
admin_user = magnum
admin_password = MAGNUM_PASS
```
在 ```[api]```加入 bind ip 與 port：
```sh
[api]
port = 9511
host = 10.0.0.11
```
在```[oslo_policy]```加入 policy 檔案位置：
```sh
[oslo_policy]
policy_file = /etc/magnum/policy.json
```

在```[oslo_concurrency]```加入 Lock path：
```sh
[oslo_concurrency]
lock_path = /var/lib/magnum/tmp
```

在```[certificates]```加入憑證放置位置：
```sh
[certificates]
cert_manager_type = local
storage_path = /var/lib/magnum/certificates/
```
> 若出錯，注意```/var/lib/magnum/certificates/```是否有建立，並設定正確權限。

完成後，透過以下指令初始化與同步資料庫：
```sh
magnum-db-manage upgrade
```

新增 ```magnum-api``` upstart 檔案```/etc/init/magnum-api.conf```：
```sh
cat > /etc/init/magnum-api.conf << EOF
description "OpenStack Magnum API"
author "kyle Bai <kyle.b@inwinStack.com>"

start on runlevel [2345]
stop on runlevel [!2345]

exec start-stop-daemon --start --chuid magnum --exec /usr/local/bin/magnum-api -- --config-file=/etc/magnum/magnum.conf
EOF
```
新增 ```magnum-conductor``` upstart 檔案```/etc/init/magnum-conductor.conf```：
```sh
cat > /etc/init/magnum-conductor.conf << EOF
description "OpenStack Magnum Conductor"
author "kyle Bai <kyle.b@inwinStack.com>"

start on runlevel [2345]
stop on runlevel [!2345]

exec start-stop-daemon --start --chuid magnum --exec /usr/local/bin/magnum-conductor -- --config-file=/etc/magnum/magnum.conf
EOF
```

完成 upstart 檔案建立後，使用```update-rc.d```指令設定開機啟動：
```sh
sudo update-rc.d magnum-api defaults
sudo update-rc.d magnum-conductor defaults
```
啟動服務：
```sh
sudo start magnum-api
sudo start magnum-conductor
```
> 也可以在 ```/etc/init.d/``` 下建立相關檔案，來使用服務 daemon。

### 驗證 Container 服務
首先上傳 magnum 使用的通用映像檔```fedora-21-atomic```，透過```wget```下載檔案：
```sh
wget https://fedorapeople.org/groups/magnum/fedora-21-atomic-5.qcow2
```
透過```glance```指令上傳映像檔至 image service：
```sh
glance image-create --name fedora-21-atomic-7  \
--disk-format=qcow2 --container-format=bare \
--property os_distro='fedora-atomic' \
--file=fedora-21-atomic-7.qcow2 \
--visibility public --progress
```

若 Mesos 則使用映像檔```ubuntu-mesos```，透過```wget```下載檔案：
```sh
wget https://fedorapeople.org/groups/magnum/ubuntu-14.04.3-mesos-0.25.0.qcow2
```
透過```glance```指令上傳映像檔至 image service：
```sh
glance image-create --name  ubuntu-mesos  \
--disk-format=qcow2 --container-format=bare \
--property os_distro='ubuntu' \
--file=ubuntu-mesos.qcow2 \
--visibility public --progress
```

#### Kubernetes 
採用 Kubernetes 可以使用以下指令建立 baymodel：
```sh
magnum baymodel-create --name k8sbaymodel \
--image-id fedora-21-atomic-5 \
--keypair-id MyKey \
--external-network-id ext-net \
--dns-nameserver 8.8.8.8 \
--flavor-id m1.small \
--docker-volume-size 5 \
--network-driver flannel \
--coe kubernetes
```
若成功會類似如下所示資訊呈現：
```sh
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| fixed_network       | None                                 |
| uuid                | 8bbec1f8-6983-495e-899b-9a8c567c20bc |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kairen                               |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | 5                                    |
| external_network_id | 9cb7c9fb-de9f-4b28-b7b4-4b496aabe2f7 |
| cluster_distro      | fedora-atomic                        |
| image_id            | 28d6814e-e6f5-4cdf-9f30-366c1100443e |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | k8sbaymodels-bai                     |
| created_at          | 2015-11-17T02:58:34+00:00            |
| network_driver      | flannel                              |
| ssh_authorized_key  | None                                 |
| coe                 | kubernetes                           |
| flavor_id           | m1.medium                            |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```
完成後，就可以建立實例的 bay 來部署叢集：
```sh
magnum bay-create --name k8sbay --baymodel k8sbaymodels-bai --node-count 1
```
> 若要更新節點數可以用以下指令：
```sh
magnum bay-update k8sbay replace node_count=2
```

使用 kubernetes 範例，下載 Google k8s 資源庫：
```sh
wget https://github.com/kubernetes/kubernetes/releases/download/v1.0.1/kubernetes.tar.gz
tar -xvzf kubernetes.tar.gz
```
> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。


#### Mesos 
採用 Mesos 可以使用以下指令建立 baymodel：
```sh
magnum baymodel-create --name mesosbaymodel --image-id ubuntu-mesos \
--keypair-id kairen \
--external-network-id ext-net \
--dns-nameserver 8.8.8.8 \
--flavor-id m1.small \
--coe mesos
```

若成功的話，會顯示類似以下的內容：
```sh
+---------------------+--------------------------------------+
| Property            | Value                                |
+---------------------+--------------------------------------+
| http_proxy          | None                                 |
| updated_at          | None                                 |
| master_flavor_id    | None                                 |
| fixed_network       | None                                 |
| uuid                | c1c6330a-2d51-4052-b82f-097c381b5544 |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kairen                               |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | None                                 |
| external_network_id | 9cb7c9fb-de9f-4b28-b7b4-4b496aabe2f7 |
| cluster_distro      | ubuntu                               |
| image_id            | ubuntu-mesos                         |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | mesosbaymodel                        |
| created_at          | 2015-11-17T04:38:09+00:00            |
| network_driver      | None                                 |
| ssh_authorized_key  | None                                 |
| coe                 | mesos                                |
| flavor_id           | m1.small                             |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```

完成後，就可以建立實例的 bay 來部署叢集：

```sh
magnum bay-create --name mesosbay --baymodel mesosbaymodel --node-count 2
```
> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。


#### Docker Swarm
採用 Docker Swarm 可以使用以下指令建立 baymodel：
```
magnum baymodel-create --name swarmbaymodel \
--image-id fedora-21-atomic-5 \
--keypair-id kairen \
--external-network-id ext-net \
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
| fixed_network       | None                                 |
| uuid                | 7f1c01ea-4006-4e67-9804-a3b36754e781 |
| no_proxy            | None                                 |
| https_proxy         | None                                 |
| tls_disabled        | False                                |
| keypair_id          | kairen                               |
| public              | False                                |
| labels              | {}                                   |
| docker_volume_size  | None                                 |
| external_network_id | 9cb7c9fb-de9f-4b28-b7b4-4b496aabe2f7 |
| cluster_distro      | fedora-atomic                        |
| image_id            | fedora-21-atomic-5                   |
| registry_enabled    | False                                |
| apiserver_port      | None                                 |
| name                | swarmbaymodel                        |
| created_at          | 2015-11-17T04:59:38+00:00            |
| network_driver      | None                                 |
| ssh_authorized_key  | None                                 |
| coe                 | swarm                                |
| flavor_id           | m1.small                             |
| dns_nameserver      | 8.8.8.8                              |
+---------------------+--------------------------------------+
```
完成後，就可以建立實例的 bay 來部署叢集：
```sh
magnum bay-create --name swarmbay --baymodel swarmbaymodel --node-count 2
```
> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。

### 其他參考
* [Magnum Liberty](http://blog.yaoyumeng.com/2015/10/30/OpenStack-Liberty%E7%89%88%E6%9C%AC%E4%B8%8B%E6%9E%81%E9%80%9F%E4%BD%93%E9%AA%8CMagnum/)
* [Magnum 架構](http://www.csdn.net/article/2015-11-07/2826146)