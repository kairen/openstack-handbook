# Murano 安裝與設定
本章節會說明與操作如何安裝```Application Catalog```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於 Murano 不瞭解的人，可以參考[Murano 應用程式目錄套件](http://murano.readthedocs.org/en/stable-kilo/install/manual.html)

# Controller節點安裝與設置
### 安裝前準備
我們需要在 Database 底下建立儲存 Murano 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Murano 資料庫與使用者：
```sql
CREATE DATABASE murano;
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'localhost'  IDENTIFIED BY ' MURANO_DBPASS';
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'%'  IDENTIFIED BY 'MURANO_DBPASS';

```
> 這邊若```MURANO_DBPASS```要更改的話，可以更改。

完成建立後，安裝 Python 相關套件：
```sh
sudo apt-get install python-pip python-dev \
libmysqlclient-dev libpq-dev \
libxml2-dev libxslt1-dev \
libffi-dev
```
並安裝虛擬環境管理套件```tox```：
```sh
sudo pip install tox
```

### Install the API service and Engine
透過 git 下載最新版本的 murano 程式：
```sh
 git clone git://git.openstack.org/openstack/murano
```
設置 murano 的設定檔案，可以透過```tox```產生：
```sh
tox -e genconfig
```