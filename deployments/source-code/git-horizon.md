# Horizon 安裝與設定
首先透過 ```apt-get``` 下載相關套件：
```sh
sudo apt-get install -y python-setuptools python-virtualenv python-dev gettext git gcc libpq-dev python-pip python-tox libffi-dev
```
透過git clone 來下載 ```OpenStack GitHub``` 的 Horizon 資源庫：
```sh
git clone https://github.com/openstack/horizon.git /opt/horizon stable/liberty
```
設定目錄權限，這邊 user 為```openstack```：
```sh
sudo chown -R ${USER}:${USER} /opt/horizon
```
> 若權限還有問題，可採用```sudo chmod 775 -R /opt/horizon```。

編譯 i18n 訊息 Catalogs：
```sh
./run_tests.sh --compilemessages
```
> 這個指令使用  Python virtualenv 進行編譯，會產生一個```.venv```目錄。結束後可以刪除。

透過 ```pip``` 套件進行安裝 Horizon：
```sh
sudo pip install .
```
複製 ```openstack_dashboard/local/local_settings.py``` 設定檔：
```sh
cp openstack_dashboard/local/local_settings.py.example openstack_dashboard/local/local_settings.py
```
加入與修改 ```openstack_dashboard/local/local_settings.py```的以下變數：
```sh
COMPRESS_OFFLINE = True
OPENSTACK_HOST = "10.0.0.11"
ALLOWED_HOSTS = '*'

CACHES = {
   'default': {
       'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
       'LOCATION': '127.0.0.1:11211',
   }
}

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
```
> 更多的部署與設定可以參考 [Deploying Horizon](http://docs.openstack.org/developer/horizon/topics/deployment.html)與[Settings and Configuration](http://docs.openstack.org/developer/horizon/topics/settings.html)。

壓縮 Django：
```sh
./manage.py collectstatic
./manage.py compress
```

設定一個支援 WSGI 的 Web server，首先安裝相關套件：
```sh
sudo apt-get install apache2 libapache2-mod-wsgi
```

可以採用內建的```openstack_dashboard/wsgi/django.wsgi```檔案，也可以自行建立：
```sh
./manage.py make_web_conf --wsgi
```

我們需為 Apache2 提供一個 WSGI 設定檔案 ```/etc/apache2/sites-available/horizon.conf```，可以採用以下指令產生：
```sh
./manage.py make_web_conf --apache | sudo tee /etc/apache2/sites-available/horizon.conf
```

修改與設定 Apache2 的 sites-available 下的 ```/etc/apache2/sites-available/horizon.conf```：
```
<VirtualHost *:80>
    DocumentRoot /opt/horizon/

    LogLevel warn
    ErrorLog /var/log/apache2/horizon-error.log
    CustomLog /var/log/apache2/horizon-access.log combined

    WSGIDaemonProcess horizon user=ubuntu group=ubuntu processes=3 threads=10 home=/opt/horizon display-name=%{GROUP}
    WSGIApplicationGroup %{GLOBAL}

    SetEnv APACHE_RUN_USER ubuntu
    SetEnv APACHE_RUN_GROUP ubuntu
    WSGIProcessGroup horizon

    WSGIScriptAlias / /opt/horizon/openstack_dashboard/wsgi/django.wsgi

    <Location "/">
        Require all granted
    </Location>

    Alias /static /opt/horizon/static
    <Location "/static">
        SetHandler None
    </Location>
</Virtualhost>
```
> 該檔案可以自行設定，也可以參考 [DevStack](http://git.openstack.org/cgit/openstack-dev/devstack/tree/files/apache-horizon.template) 的範例。

最後，啟用配置與重啟 apache2 服務：
```sh
 sudo a2ensite horizon
 sudo service apache2 restart
```

# 驗證操作
這個部分將描述如何進行儀表板的驗證操作，依照以下兩個簡單步驟：
1. 開啟web瀏覽器進入儀表板: http://controller。
2. 使用admin或demo的使用者登入。

![horizon](images/horizon.png)
