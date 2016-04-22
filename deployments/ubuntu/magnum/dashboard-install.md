# Magnum Dashboard
在 OpenStack Liberty 版本中，Magnum 已成為正式釋出第一個版本，因此 Magnum 也開始整合於 OpenStack Dashboard 上，本章節就是安裝該 UI 套件。

### 安裝 Magnum Dashboard
由於 Magnum UI 目前還沒有相關 .deb 安裝套件，故使用 git 的 repository 安裝：
```sh
$ git clone https://github.com/openstack/horizon
$ git clone https://github.com/openstack/magnum-ui
```
> 安裝 Horizon 參考本書 Git 安裝章節

安裝 Magnum Dashboard 的相依套件與環境：
```sh
$ sudo pip install -e magnum-ui/
```
> 也可以用 ```python setup.py install``` 安裝，差異在於一個是參考，一個是直接安裝到 /usr/bin。

將 Magnum dashboard 相關程式檔案複製到 Horizon:
```sh
cp ../magnum-ui/enabled/_50_project_containers_panelgroup.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_51_project_containers_bays_panel.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_52_project_containers_baymodels_panel.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_53_project_containers_containers_panel.py openstack_dashboard/local/enabled
```
> 若是```liberty```的話，請使用以下指令：
```sh
$ cp ../magnum-ui/enabled/_50_add_containers_dashboard.py openstack_dashboard/local/enabled
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
> 若已安裝過 Horizon 於 HTTP Server 上的話，可以重啟 Apache 或 httpd 服務。
