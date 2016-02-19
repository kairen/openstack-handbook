# OpenStack MySQL High Availability
OpenStack High Availability 有兩種常見部署模式，一個為 ```Active/Active``` 與 ```Active/Passive```，前者為每個節點主動提供服務，後者則是主只有節點提供服務，另外節點當作備援使用。MySQL 的 HA 可以用以下兩種方式達成。

## MariaDB Galera 叢集
### 叢集環境說明
本教學環境採用三台節點進行安裝，分別為：
* **Controller1**：10.0.0.11。
* **Controller2**：10.0.0.12。
* **Controller3**：10.0.0.13。

安裝 cluster 相關套件，以下動作要在 cluster 中的每一個 node 上面執行：
```sh
$ sudo apt-get -y install software-properties-common python-software-properties
$ sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
$ sudo add-apt-repository 'deb http://nyc2.mirrors.digitalocean.com/mariadb/repo/10.0/ubuntu trusty main'
$ sudo apt-get update
$ sudo apt-get -y install python-mysqldb galera mariadb-galera-server
```

### 建立同步資料庫用的使用者帳號
登入每一台 MariaDB 中，建立使用者並設定權限：
```sql
GRANT USAGE ON *.* to openstack@'%' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL PRIVILEGES on *.* to openstack@'%';
GRANT USAGE ON *.* to openstack@'localhost' IDENTIFIED BY 'p@ssw0rd';
GRANT ALL PRIVILEGES on *.* to openstack@'localhost';
```

## 設定 cluster
以下動作要在 cluster 中的每一個 node 上面執行，新增設定檔```/etc/mysql/conf.d/cluster.cnf```，並設定內容如下：
```sh
[mysqld]
binlog_format = ROW
default-storage-engine = innodb
innodb_autoinc_lock_mode = 2
query_cache_size = 0
query_cache_type = 0
bind-address = 0.0.0.0

# Galera Settings
wsrep_node_address = "controller1"
wsrep_provider = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name = "openstack-mysql-cluster"
wsrep_cluster_address = "gcomm://10.0.0.11:4567,10.0.0.12:4567,10.0.0.13:4567"
wsrep_sst_auth = openstack:p@ssw0rd
wsrep_sst_method = rsync
```
> ```wsrep_node_address```跟```wsrep_node_name``` 需要隨節點更改。

為了確保 debian-sys-maint 的密碼是相同的，我們必須將第一個 node 的 /etc/mysql/debian.cnf 複製到其他 node：
```sh
$ sudo cat /etc/mysql/debian.cnf | ssh controller3 "sudo tee /etc/mysql/debian.cnf"
```

### Start the Cluster
在每個節點關閉 mysql：
```sh
$ sudo service mysql stop
```
確認所有節點都關閉後，到主節點開啟服務：
```sh
$ sudo service mysql start --wsrep-new-cluster
```

到其他節點開啟服務：
```sh
$ sudo service mysql start
```

## 驗證
隨便選一台機器，登入 MariaDB(以 root 身分)，並下達以下命令：
```sql
show status like 'wsrep%';
```


## 參考文獻
* http://godleon.github.io/blog/2015/05/27/Build-MariaDB-Galera-Cluster-in-Ubuntu/
* http://scar.tw/article/2015/03/11/mariadb-galera-cluster-10-0-setting/