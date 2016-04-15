# Trove 安裝與設定
本章節會說明與操作如何安裝資料庫即服務到 Controller 節點上，並設置相關參數與設定。若對於 Trove 不瞭解的人，可以參考 [Trove 資料庫即服務章節](../../../conceptions/trove/README.md)。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Trove 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Trove 資料庫：
```sql
CREATE DATABASE trove;
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost'  IDENTIFIED BY 'TROVE_DBPASS';
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'TROVE_DBPASS';
```
> 這邊```TROVE_DBPASS```可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 ```admin``` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Trove 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Trove user
$ openstack user create --domain default --password TROVE_PASS --email trove@example.com trove

# 新增 Trove 到 Admin Role
$ openstack role add --project service --user trove admin

# 建立 Trove service
$ openstack service create --name trove  --description "OpenStack Database service" database

# 建立 Trove public endpoints
$ openstack endpoint create --region RegionOne \
database public http://10.0.0.11:8779/v1.0/%\(tenant_id\)s

# 建立 Trove internal endpoints
$ openstack endpoint create --region RegionOne \
database internal http://10.0.0.11:8779/v1.0/%\(tenant_id\)s

# 建立 Trove admin endpoints
$ openstack endpoint create --region RegionOne \
database admin http://10.0.0.11:8779/v1.0/%\(tenant_id\)s
```
> 這邊若 ```TROVE_PASS``` 要更改的話，可以更改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y trove-api trove-taskmanager trove-conductor \
python-trove python-troveclient
```
