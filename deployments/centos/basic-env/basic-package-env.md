# Network Time Protocol (NTP)
由於要讓各節點的時間能夠同步，我們需要安裝```ntp```套件來提供服務，這邊推薦將 NTP Server 安裝於 Controller 上，再讓其他節點進行關聯即可。

### Controller 節點設定
在 Controller 節點上，我們可以以下指令來安裝套件：
```sh
$ sudo yum install -y ntp
```
> P.S NTP Server 也可以考慮使用 chrony，透過以下方式安裝：
```sh
$ sudo yum install chrony
```
> 若要修改設定檔則編輯```/etc/chrony.conf```。基本設定上與 ntp 雷同。這邊要在設定檔多加入```allow 10.0.0.0/24```。

完成安裝後，預設 NTP 會透過公有的伺服器來同步時間，但也可以修改```/etc/ntp.conf```來選擇伺服器：
```sh
restrict 10.0.0.0 mask 255.255.255.0 nomodify notrap

server 2.tw.pool.ntp.org
server 3.asia.pool.ntp.org
server 0.asia.pool.ntp.org
```

將 ```NTP_SERVER``` 替換為主機名稱或更準確的（lower stratum） NTP 伺服器 IP 地址。這個設定支援多個 server 關鍵字。
> 如果系統有```/var/lib/ntp/ntp.conf.dhcp```存在，請將它刪除。

設定正確的 Time Zone：
```sh
$ sudo timedatectl set-timezone Asia/Taiwan
```

完成後重新啟動 NTP 服務：
```sh
sudo systemctl enable ntp.service
sudo systemctl start ntp.service
```
> Chrony 使用以下指令：
```sh
$ sudo systemctl enable chronyd.service
$ sudo systemctl start chronyd.service
```

### 其他節點設定
在其他節點一樣安裝 NTP：
```sh
$ sudo yum install -y ntp
```
> P.S NTP Server 也可以考慮使用 chrony，透過以下方式安裝：
```sh
$ sudo yum install chrony
```
> 若要修改設定檔則編輯```/etc/chrony.conf```。基本設定上與 ntp 雷同。

完成安裝後，編輯```/etc/ntp.conf```檔案，註解掉所有```server```的參數，並將其設定為 Controller IP：
```sh
server 10.0.0.11 iburst
```
> 如果系統有```/var/lib/ntp/ntp.conf.dhcp```存在，請將它刪除。

設定正確的 Time Zone：
```sh
$ sudo timedatectl set-timezone Asia/Taiwan
```

完成後重新啟動 NTP 服務：
```sh
sudo systemctl enable ntp.service
sudo systemctl start ntp.service
```
> Chrony 使用以下指令：
```sh
$ sudo systemctl enable chronyd.service
$ sudo systemctl start chronyd.service
```

### 驗證設定
當完成上述設定後，建議在進一步的安裝之前驗證NTP 是否已同步。有些節點，特別是哪些引用了 Controller 節點，需要花費一些時間去同步。

可以在 Controller 節點，下該指令：
```sh
$ ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*ntp-server1     192.0.2.11       2 u  169 1024  377    1.901   -0.611   5.483
+ntp-server2     192.0.2.12       2 u  887 1024  377    0.922   -0.246   2.864
```
> Chrony 使用以下指令：
```sh
$ chronyc sources
```

也可以透過以下指令，進一步對 Controller 做驗證：
```sh
$ ntpq -c assoc
ind assid status  conf reach auth condition  last_event cnt
===========================================================
  1 20487  961a   yes   yes  none  sys.peer    sys_peer  1
  2 20488  941a   yes   yes  none candidate    sys_peer  1
```

當都沒問題後，可以在其他節點下達該指令：
```sh
$ ntpq -c peers
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*controller      192.0.2.21       3 u   47   64   37    0.308   -0.251   0.079
```
> Chrony 使用以下指令：
```sh
$ chronyc sources
```

# 安裝 OpenStack 套件
接下來我們需在每個節點安裝 Openstack 相關套件，但由於 CentOS 的版本差異，會影響 OpenStack 支援的版本，在安裝時要特別注意是否支援，才會有對應的 Repository 可以使用。

首先透過以下指令加入 Repository 來獲取套件：
```sh
$ sudo yum install centos-release-openstack-mitaka
```
> 若要安裝 ``` pre-release``` 測試版本可以使用以下指令：
```sh
$ sudo yum install yum-plugin-priorities
$ cd /etc/yum.repos.d/
$ curl -O http://trunk.rdoproject.org/centos7/delorean-deps.repo
$ curl -O http://trunk.rdoproject.org/centos7/current-passed-ci/delorean.repo
```

> 若要安裝 ```liberty```，修改為```centos-release-openstack-liberty```。

更新 Repository 與系統核心套件：
```sh
$ sudo yum upgrade
```
> 如果 Upgrade 包含了新的核心套件的話，請重新開機。

由於 RHEL 與 CentOS 預設會使用 SELinux，故要安裝 openstack-selinux 來自動管理安全策略：
```sh
$ sudo yum install openstack-selinux
```

# SQL database 安裝
大部份的 OpenStack 套件服務都是使用 SQL 資料庫來儲存訊息，該資料庫一般運作於```Controller```上。以下我們使用了 MariaDB 或 MySQL 來當作各套件的資訊儲存。OpenStack 也支援了其他資料庫，諸如：PostgreSQL。這邊透過以下指令來安裝 MariaDB 套件：
```sh
$ sudo yum install mariadb mariadb-server python2-PyMySQL
```
> 記住 Python MySQL 和 MariaDB 是相容的。

完成安裝後，需要建立並編輯``` /etc/my.cnf.d/openstack.cnf```來設定資料庫。在```[mysqld]```部分加入以下修改：
```sh
[mysqld]
bind-address = 10.0.0.11

default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
character-set-server = utf8
```
> 這邊的檔案 ```mysqld_openstack.cnf``` 是可以修改名稱的。

完成後重新啟動 MySQL 服務：
```sh
sudo systemctl enable mariadb.service
sudo systemctl start mariadb.service
```

並對資料庫進行安全設定：
```sh
$ sudo mysql_secure_installation
```
> 因為資料庫預設沒有 root 密碼所以看到
```sh
Enter current password for root (enter for none):
```
直接按 Enter 就可以進行資料庫密碼變更

> 若要備份資料庫可以用以下指令：
```sh
$ mysqldump --user=root -p --all-databases > mysql.sql
```
> 若要恢復資料庫可以用以下指令：
```sh
$ mysql -u root -p < mysql.sql
```

除了更換密碼外，其餘每個項目都輸入```yes```，並設置對應資訊。

# Message queue 安裝
OpenStack 使用 Message Queue 來對整個叢集提供協調與狀態訊息收集。Openstack 支援的 Message Queue 包含以下[RabbitMQ](http://www.rabbitmq.com/)、[Qpid](http://qpid.apache.org/)、[ZeroMQ](http://zeromq.org/)。但是大多數的釋出版本支援特殊的 Message Queue 服務，這邊我們使用了```RabbitMQ```來實現，並安裝於```Controller```節點上，透過以下指令安裝套件：
```sh
$ sudo yum install rabbitmq-server
```

完成安裝後，啟動 RabbitMQ 服務：
```sh
sudo systemctl enable rabbitmq-server.service
sudo systemctl start rabbitmq-server.service
```

新增一個名稱為 ```openstack``` 的 User:
```sh
$ sudo rabbitmqctl add_user openstack RABBIT_PASS
Creating user "openstack" ...
...done.
```
> 可以更改 RABBIT_PASS，來改變密碼。

為剛建立的 User 設定存取權限：
```sh
$ sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
Setting permissions for user "openstack" in vhost "/" ...
...done.
```

# 提醒
接下來會依序針對 OpenStack 的基礎套件進行安裝與設定教學，若發現有設定格式中有 ```...``` 代表上面有預設設定的其他參數，若沒提示```要註解掉```，請不要修改：
```
[DEFAULT]
...
admin_token = 74e00617afa2008fcf25
```
> P.S. 若在設定時，找不到對應的```[section]```部分，請自行新增。
