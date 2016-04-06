# Ironic 安裝與設定
本章節會說明與操作如何安裝裸機服務到 Controller 與 Compute 節點上，並修改相關設定檔。若對於 Ironic 不瞭解的人，可以參考 [Ironic 裸機服務套件](../../../conceptions/ironic/README.md)。

- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [Share Node](#share-node)
    - [Share 套件安裝與設定](#share-套件安裝與設定)
- [驗證服務](#驗證服務)

# Controller Node

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Glance 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Ironic 資料庫：
```sql
CREATE DATABASE ironic;
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'localhost'  IDENTIFIED BY 'IRONIC_DBPASS';
GRANT ALL PRIVILEGES ON ironic.* TO 'ironic'@'%' IDENTIFIED BY 'IRONIC_DBPASS';
```
> 這邊```IRONIC_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Ironic 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Ironic User
$ openstack user create --domain default --password IRONIC_PASS --email ironic@example.com ironic

# 新增 Ironic 到 Admin Role
$ openstack role add --project service --user ironic admin

# 建立 Ironic service
$ openstack service create --name ironic \
--description "OpenStack Bare Metal Provisioning Service" baremetal

# 建立 Ironic public endpoints
$ openstack endpoint create --region RegionOne \
baremetal public http://10.0.0.11:6385

# 建立 Ironic internal endpoints
$ openstack endpoint create --region RegionOne \
baremetal internal http://10.0.0.11:6385

# 建立 Ironic admin endpoints
$ openstack endpoint create --region RegionOne \
baremetal admin http://10.0.0.11:6385
```
