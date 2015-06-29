# Horizon 安裝與設定
架設環境都確認完成，到```Controller```節點上透過```apt-get```安裝套件：
```sh
sudo apt-get install openstack-dashboard
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

TIME_ZONE = "TIME_ZONE"
```
> Replace TIME_ZONE with an appropriate time zone identifier. For more information, see the [list of time zones](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

重新讀取服務：
```sh
sudo service apache2 reload
```

