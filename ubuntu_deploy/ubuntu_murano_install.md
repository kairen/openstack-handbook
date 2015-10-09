# Murano 安裝與設定
本章節會說明與操作如何安裝```Application Catalog```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於 Murano 不瞭解的人，可以參考[Murano 應用程式目錄套件]()

# Controller節點安裝與設置
### 安裝前準備
我們需要在Database底下建立儲存 Sahara 資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立 Sahara 資料庫與使用者：
```sql
CREATE DATABASE sahara;
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'localhost'  IDENTIFIED BY ' SAHARA_DBPASS';
GRANT ALL PRIVILEGES ON sahara.* TO 'sahara'@'%'  IDENTIFIED BY 'SAHARA_DBPASS';

```
> 這邊若```SAHARA_DBPASS```要更改的話，可以更改。