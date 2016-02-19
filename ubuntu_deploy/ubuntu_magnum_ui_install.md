# Magnum-UI 安裝
在 OpenStack Liberty 版本中，Magnum 的 Container 已經正式釋出 Container as a Service，且 Magnum 也開始整合於 Dashboard 上，本章節就是安裝該 UI 套件。

### Install Magnum UI
由於 Magnum UI 目前還沒有相關 .deb 安裝套件，故使用 git 的 repository 安裝：
```sh
git clone https://github.com/openstack/horizon
git clone https://github.com/openstack/magnum-ui
```
> 安裝 Horizon 參考本書 Git 安裝章節


Install Magnum UI with all dependencies in your environment：
```sh
pip install -e magnum-ui/
```
> 也可以用 ```python setup.py install``` 安裝，差異在於一個是參考，一個是直接安裝到 /usr/bin。

And enable it in Horizon:
```sh
cp ../magnum-ui/enabled/_50_project_containers_panelgroup.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_51_project_containers_bays_panel.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_52_project_containers_baymodels_panel.py openstack_dashboard/local/enabled
cp ../magnum-ui/enabled/_53_project_containers_containers_panel.py openstack_dashboard/local/enabled
```
> 若是```liberty```的話，請使用以下指令：
```sh
cp ../magnum-ui/enabled/_50_add_containers_dashboard.py openstack_dashboard/local/enabled
```

To run horizon with the newly enabled Magnum UI plugin run:
```sh
./run_tests.sh --runserver 0.0.0.0:8080
```
> 若已安裝過 Horizon 於 HTTP Server 上的話，可以重啟 Apache 或 httpd 服務。
