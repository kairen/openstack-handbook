# Horizon 安裝與設定
本章節會說明與操作如何安裝```Dashboard```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Horizon不瞭解的人，可以參考[Horizon 儀表板套件章節](http://kairen.gitbooks.io/openstack/content/horizon/index.html)

#### 系統需求
在安裝OpenStack Dashborad前，須滿足以下幾點：
* OpenStack Compute安裝。並啟動了 Identity service 用於使用者與專案管理。
> 需注意 Identity service 與 Compute 的 endpoints

* Identity service的使用者可以使用 sudo 權限。因為 Apache 不提供從 Root 使用者獲取內容，使用者若必須運行 dashboard ，要為 Identity service 的使用者擁有 sudo 特權。
* Python 需為 2.7 以上版本。Python 的版本必須支援 Django。Python 的版本應該可以運作在任何系統上，包括Mac OS X的安裝....等。

#### 安裝套件
假設我們的基本OpenStack環境都確認完成後，回到```Controller```節點上透過```apt-get```安裝```dashboard```套件：
```sh
sudo apt-get install openstack-dashboard
```
> Ubuntu 安裝 openstack-dashboard 時，會有```ubuntu-theme```套件，若發生問題或者不需要，可以直接刪除該套件。
```sh
sudo apt-get remove --purge openstack-dashboard-ubuntu-theme
```

#### 設定儀表板
編輯```/etc/openstack-dashboard/local_settings.py ```修改以下：
```py
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = '*'

CACHES = {
   'default': {
       'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION': '127.0.0.1:11211',
   }
}

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```
> Replace TIME_ZONE with an appropriate time zone identifier. For more information, see the [list of time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

重新讀取服務：
```sh
sudo service apache2 reload
sudo service apache2 restart
```

# 驗證操作
這個部分將描述如何進行儀表板的驗證操作，依照以下兩個簡單步驟：
1. 開啟web瀏覽器進入儀表板: http://controller/horizon。
2. 使用admin或demo的使用者登入。

![horizon](images/horizon.png)
