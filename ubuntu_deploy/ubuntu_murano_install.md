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