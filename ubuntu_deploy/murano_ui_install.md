# Murano UI 安裝
在 OpenStack Kilo 版本中，Murano 的 Application Catalog 已經正式釋出正式版，且 Murano 也整合於 Dashboard 上，本章節就是安裝該 UI 套件。

### Install Murano UI
由於 Murano UI 目前還沒有相關 .deb 安裝套件，故使用 git 的 repository 安裝：
```sh
git clone https://github.com/openstack/horizon
git clone git://git.openstack.org/openstack/murano-dashboard
```
> 安裝 Horizon 參考本書 Git 安裝章節

> if not use keystone, add ```MURANO_API_URL = 'http://localhost:8082'``` to horizon local_settings.py

Install Murano UI with all dependencies in your environment：
```sh
pip install -e murano-dashboard/
```
> 也可以用 ```python setup.py install``` 安裝，差異在於一個是參考，一個是直接安裝到 /usr/bin。

進入 Horizon git 目錄，並複製以下檔案:
```sh
cp ../murano-dashboard/muranodashboard/local/_50_murano.py openstack_dashboard/local/enabled/
```
To run horizon with the newly enabled Murano UI plugin run:
```sh
./run_tests.sh --runserver 0.0.0.0:8080
```
> 若已安裝過 Horizon 於 HTTP Server 上的話，可以重啟 Apache 或 httpd 服務。
