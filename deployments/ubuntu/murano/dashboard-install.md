# Murano UI 安裝
在 OpenStack Kilo 版本中，Murano 的 Application Catalog 已經正式釋出正式版，且 Murano 也整合於 Dashboard 上，本章節就是安裝該 UI 套件。

### Install Murano UI
由於要取得最新版本，故使用 git 的 repository 安裝：
```sh
$ git clone https://github.com/openstack/horizon
$ git clone https://github.com/openstack/murano-dashboard.git
```
> 安裝 Horizon 參考本書 Git 安裝章節

> 若使用 .deb 安裝的話，可以執行以下指令：
>
```
$ sudo apt-get install murano-dashboard
```

> 如果沒有使用 keystone, 加入 ```MURANO_API_URL = 'http://localhost:8082'``` 到 horizon local_settings.py檔案。

安裝 Murano UI 相依套件到環境上：
```sh
$ sudo pip install -e murano-dashboard/
```
> 也可以用 ```python setup.py install``` 安裝，差異在於一個是參考，一個是直接安裝到 /usr/bin。

進入 Horizon git 目錄，並複製以下檔案:
```sh
$ cp ../murano-dashboard/muranodashboard/local/_50_murano.py openstack_dashboard/local/enabled/
```

完成後讓 Django 進行 collectstatic 與 compress：
```sh
$ ./manage.py collectstatic
$ ./manage.py compress
```

透過```./run_tests.sh```來執行測試用 Dashboard：
```sh
$ ./run_tests.sh --runserver 0.0.0.0:8080
```
> 若已安裝過 Horizon 於 HTTP Server 上的話，可以重啟 Apache 或 httpd 服務。
