# Ironic 安裝與設定
本章節會說明與操作如何安裝```Baremetal Service```服務到 OpenStack Controller 節點上，並設置相關參數與設定。若對於 Ironic 不瞭解的人，可以參考 [Ironic 裸機服務套件]()。

### 安裝前準備
我們需要在 Database 底下建立儲存 Ironic 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Ironic 資料庫與使用者：
```sql
CREATE DATABASE ironic;
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost'  IDENTIFIED BY 'IRONIC_DBPASSWORD';
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%'  IDENTIFIED BY 'IRONIC_DBPASSWORD';

```
> 這邊若```IRONIC_DBPASSWORD```要更改的話，可以更改。

接著要導入Keystone的```admin```帳號，來建立服務：
```sh
source admin-openrc.sh
```
透過以下指令建立服務驗證：
```sh
# 建立 Ironic User
openstack user create --password IRONIC_PASSWORD --email ironic@example.com ironic
# 建立 Ironic Role
openstack role add --project service --user ironic admin
# 建立 Ironic service
openstack service create --name ironic  --description "Baremetal Service" baremetal
# 建立 Ironic URL
openstack endpoint create  --publicurl http://controller:6385  --internalurl http://controller:6385  --adminurl http://controller:6385  --region RegionOne baremetal
```
> 這邊若```IRONIC_PASSWORD```要更改的話，可以更改。

建立 ironic 使用者與相關檔案放置用目錄：
```sh
for SERVICE in ironic
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

### 安裝 Baremetal service
當 endpoint 與 mysql 建立完成，就可以開始進行安裝，這邊為了使用最新版本故使用 git 的 repository 安裝：
```sh
cd ~/
git clone https://github.com/openstack/ironic.git
cd ironic
sudo pip install -e .
```
> 若使用 .deb 安裝的話，可以執行以下指令：
> 
```
sudo apt-get install ironic-api ironic-conductor python-ironicclient
```


建立 ```/etc/ironic``` 放置設定檔：
```sh
sudo mkdir -p /etc/ironic
```
複製相關設定與參數檔案於 ```/etc/ironic```：
```sh
sudo cp etc/ironic/ironic.conf.sample /etc/ironic/ironic.conf
sudo cp etc/ironic/policy.json /etc/ironic/policy.json
```

編輯設定檔```/etc/ironic/ironic.conf```，加入以下：
```sh
[database]
connection = mysql://ironic:IRONIC_DBPASSWORD@DB_IP/ironic?charset=utf8
[DEFAULT]
rabbit_host=RABBIT_HOST
[DEFAULT]
auth_strategy=keystone
[keystone_authtoken]
auth_host=IDENTITY_IP
#auth_port=35357
#auth_protocol=http
auth_uri=http://IDENTITY_IP:5000/
admin_user=ironic
admin_password=IRONIC_PASSWORD
admin_tenant_name=service
[neutron]
url=http://NEUTRON_IP:9696
[glance]
glance_host=GLANCE_IP
```

完成後，進行同步動作：
```sh
sudo ironic-dbsync --config-file /etc/ironic/ironic.conf
```
重啟服務：
```sh
service ironic-api restart
service ironic-conductor restart
```
設定 ```nova-compute```的配置檔``` /etc/nova/nova.conf```：
```sh
[default]
compute_driver=ironic.nova.virt.ironic.IronicDriver
scheduler_host_manager=ironic.nova.scheduler.ironic_host_manager.IronicHostManager
ram_allocation_ratio=1.0
compute_manager=ironic.nova.compute.manager.ClusteredComputeManager
[ironic]
admin_username=ironic
admin_password=IRONIC_PASSWORD
admin_url=http://IDENTITY_IP:35357/v2.0
admin_tenant_name=service
api_endpoint=http://IRONIC_NODE:6385/v1
```
重新啟動```nova-scheduler```：
```sh
sudo service nova-scheduler restart
```

重新啟動```nova-compute```：
```sh
service nova-compute restart
```

設定 PXE：
```sh
sudo mkdir -p /tftproot
sudo chown -R ironic:LIBVIRT_GROUP -p /tftproot
mkdir -p /tftproot/pxelinux.cfg
sudo cp /usr/share/syslinux/pxelinux.0 /tftproot
```

## 參考文獻
* https://m.oschina.net/blog/288497