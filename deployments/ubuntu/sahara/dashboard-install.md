# Sahara Dashboard
在 OpenStack Sahara 已經經過許多版本的感善，目前 Sahara 也早已整合進 OpenStack Dashboard 上，本章節就是安裝該 Dashboard 套件。

### 安裝 Sahara Dashboard
假設 OpenStack Sahara 的環境已經部署完成，且正常運作的話。即可以開始進行 Dashboard 部署，這邊為了取得最新版本的 UI 以及方便更新，故採用 Git source 進行安裝：
```sh
$ git clone https://github.com/openstack/horizon
$ git clone https://github.com/openstack/sahara-dashboard.git
```
> 安裝 Horizon 參考本書 Git 安裝章節

安裝 Sahara Dashboard 的相依套件與環境：
```sh
$ sudo pip install -e sahara-dashboard/
```
> 也可以用 ```python setup.py install``` 安裝，差異在於一個是參考，一個是直接安裝到 /usr/bin。

將 Sahara dashboard 相關程式檔案複製到 Horizon:
```sh
$ cd horizon/
$ cp ../sahara-dashboard/sahara_dashboard/enabled/_*.py openstack_dashboard/local/enabled
```

完成後讓 Django 進行 collectstatic 與 compress：
```sh
$ ./manage.py collectstatic
$ ./manage.py compress
```

完成後可以透過 Django 來執行測試:
```sh
$ ./run_tests.sh --runserver 0.0.0.0:8080
```
> 若已安裝過 Horizon 於 HTTP Server 上的話，可以重啟 Apache2 或 httpd 服務。
