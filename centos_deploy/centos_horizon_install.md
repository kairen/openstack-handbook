# Horizon 安裝與設定
本章節會說明與操作如何安裝```Dashboard```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Horizon不瞭解的人，可以參考[Horizon 儀表板套件章節](http://kairen.gitbooks.io/openstack/content/horizon/index.html)

#### 系統需求
在安裝OpenStack Dashborad前，須滿足以下幾點：
* OpenStack Compute安裝。並啟動了 Identity service 用於使用者與專案管理。
> 需注意 Identity service 與 Compute 的 endpoints

* Identity service的使用者可以使用 sudo 權限。因為 Apache 不提供從 Root 使用者獲取內容，使用者若必須運行 dashboard ，要為 Identity service 的使用者擁有 sudo 特權。
* Python 需為 2.7 以上版本。Python 的版本必須支援 Django。Python 的版本應該可以運作在任何系統上，包括Mac OS X的安裝....等。


#### 安裝與設定套件
假設OpenStack基本環境已佈建完成，且Horizon節點符和最低需求，可以透過```yum```安裝：
```sh
yum install -y openstack-dashboard httpd mod_wsgi memcached python-memcached
```

安裝完成後，編輯```/etc/openstack-dashboard/local_settings```檔案並加入設定，首先找到```OPENSTACK_HOST```取代為以下：
```sh
OPENSTACK_HOST = "controller"
```
允許所有主機可以存取 dashboard ：
```sh
ALLOWED_HOSTS = '*'
```
> 可視情況決定是否給任何主機存取

設置 memcached session storage service：
```sh
CACHES = {
   'default': {
       'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION': '127.0.0.1:11211',
   }
}
```
> 請註解掉其他的 session storage configuration

配置 ```user``` 作為透過 dashboard 建立的使用者的預設規則：
```
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```
最後看是否要跟換時區：
```sh
TIME_ZONE = "TIME_ZONE"
```
> 相關的時區資訊可以看 [list of time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)

#### 完成安裝
我們需在 RHEL 與 CentOS 中，配置 SElinux 來允許 Web 伺服器連接到 OpenStack 服務：
```sh
setsebool -P httpd_can_network_connect on
```
由於套件的 bug，dashboard 的 CSS 會載入失敗。透過以下指令來解決問題：
```sh
chown -R apache:apache /usr/share/openstack-dashboard/static
```
> 更多的資訊，可以看 [bug report](https://bugzilla.redhat.com/show_bug.cgi?id=1150678)

最後重新開啟服務與設定boot開啟：
```sh
systemctl enable httpd.service memcached.service
systemctl start httpd.service memcached.service
```

# 驗證操作
這部分會說明如何驗證 dashboard 套件：
1. 透過瀏覽器開啟 http://controller/dashboard 驗證。
2. 登入 ```admin``` 或者 ```demo``` 進行認證。
