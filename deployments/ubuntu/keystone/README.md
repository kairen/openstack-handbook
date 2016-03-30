# Keystone 安裝與設定
本章節會說明與操作如何安裝身份認證服務到 Controller 節點上，並設置相關參數與設定。若對於 Keystone 不瞭解的人，可以參考 [Keystone 身份認證套件章節](../../../conceptions/keystone/README.md)。

### 安裝前準備
我們需要在 MySQL 建立儲存 Keystone 資訊的資料庫，利用 ```mysql``` 指令進入：
```sh
mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Keystone 新帳號：
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
> 這邊若```KEYSTONE_DBPASS```要更改的話，可以更改。

完成後，透過 ```quit``` 指令離開資料庫，並透過 ```openssl``` 建立一個隨機的 admin token：
```sh
openssl rand -hex 24

# 會產生如下密碼
21d7fb48086e09f30d40be5a5e95a7196f2052b2cae6b491
```

### 安裝與設定 Keystone 套件
首先我們設定安裝完成後，不要自動開啟服務：
```sh
echo "manual" | sudo tee /etc/init/keystone.override
```
> 在 ```Kilo``` 的 Keystone 遺棄了 ```Eventlet```，改用```WSGI```來代替。所以本教學關閉使用 keystone 服務。

再來透過```apt-get```安裝 Keystone 套件：
```sh
sudo apt-get install keystone apache2 memcached libapache2-mod-wsgi python-openstackclient python-memcache
```

安裝完後，編輯```/etc/keystone/keystone.conf```，將 ADMIN_TOKEN 替換為上一面產生的隨機字串：
```
[DEFAULT]
admin_token = ADMIN_TOKEN
```

在```[database]```部分修改如下：
```sh
[database]
# connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:KEYSTONE_DBPASS@10.0.0.11/keystone
```

在```[memcache]```部分修改如下：
```
[memcache]
servers = localhost:11211
```

在```[token]```部分修改如下：
```
[token]
provider = fernet
driver = memcache
```

完成後同步資料庫：
```sh
sudo keystone-manage db_sync
```

初始化 Fernet keys：
```sh
sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
```

成功會看到類似以下內容：
```sh
2016-03-30 13:08:39.452 9126 INFO keystone.token.providers.fernet.utils [-] [fernet_tokens] key_repository does not appear to exist; attempting to create it
2016-03-30 13:08:39.453 9126 INFO keystone.token.providers.fernet.utils [-] Created a new key: /etc/keystone/fernet-keys/0
2016-03-30 13:08:39.453 9126 INFO keystone.token.providers.fernet.utils [-] Starting key rotation with 1 key files: ['/etc/keystone/fernet-keys/0']
2016-03-30 13:08:39.453 9126 INFO keystone.token.providers.fernet.utils [-] Current primary key is: 0
2016-03-30 13:08:39.454 9126 INFO keystone.token.providers.fernet.utils [-] Next primary key will be: 1
2016-03-30 13:08:39.454 9126 INFO keystone.token.providers.fernet.utils [-] Promoted key 0 to be the primary: 1
2016-03-30 13:08:39.454 9126 INFO keystone.token.providers.fernet.utils [-] Created a new key: /etc/keystone/fernet-keys/0
```

### 設置 Apache HTTP 伺服器
編輯```/etc/apache2/apache2.conf```的 ServerName 設為以下：
```
ServerName 10.0.0.11
```

再來建立 WSGI 檔案 ```/etc/apache2/sites-available/wsgi-keystone.conf``` 給 Apache 使用，並加入以下內容：
```
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
```

完成後，開啟身份驗證服務虛擬主機：
```sh
sudo ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```

重新啟動 HTTP 服務：
```sh
sudo service apache2 restart
```

刪除預設產生的 SQLite 資料庫：
```sh
sudo  rm -f /var/lib/keystone/keystone.db
```

# 建立服務實體和 API 端點
首先透過```export```設定 OS_TOKEN 環境變數，輸入 Keystone 使用的 ADMIN_TOKEN 與 API URL：
```sh
export OS_TOKEN=21d7fb48086e09f30d40be5a5e95a7196f2052b2cae6b491
export OS_URL=http://10.0.0.11:35357/v2.0
```
建立服務實體和身份驗證服務：
```sh
openstack service create --name keystone \
--description "OpenStack Identity" identity
```
會看到產生類似以下資訊：
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 4ddaae90388b4ebc9d252ec2252d8d10 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```
> OpenStack 是動態產生 ID 的，因此看到的輸出會跟範例中的輸出不同。

身份驗證服務管理了一個與環境相關的 API 端點的目錄。服務使用這個目錄來決定如何與環境中的其他服務進行溝通。透過以下建立一個 API 端點：
```sh
openstack endpoint create \
--publicurl http://10.0.0.11:5000/v3 \
--internalurl http://10.0.0.11:5000/v3 \
--adminurl http://10.0.0.11:35357/v3 \
--region RegionOne identity
```
會看到產生類似以下資訊：
```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://10.0.0.11:35357/v3        |
| id           | 0af1316372844e579bc84cf71f66fff0 |
| internalurl  | http://10.0.0.11:5000/v3         |
| publicurl    | http://10.0.0.11:5000/v3         |
| region       | RegionOne                        |
| service_id   | 721282b3c5514bcab6147a29fbaf28e7 |
| service_name | keystone                         |
| service_type | identity                         |
+--------------+----------------------------------+
```

> 添加到 OpenStack 環境中的每個服務，都需要一個或多個服務實體和身份驗證服務的一個 API 端點。

# 建立 Projects、Users 與 Roles
驗證服務會對每一個 Openstack 的服務提供身份驗證。驗證服務使用 Domains, Projects (tenants), Users 與 Roles 的組合。在環境中，為了進行管理操作，我們要建立```admin```的 Project、User 和 Role：
```sh
# 建立 admin Project(tenant)
openstack project create --description "Admin Project" admin

# 建立 admin User
openstack user create --password passwd --email admin@example.com admin

# 建立 admin Role
openstack role create admin

# 將 admin Role 加到 Project 與 User
openstack role add --project admin --user admin admin

# 建立 service Project
openstack project create --description "Service Project" service
```
為了驗證，我們建立一個沒有特權的使用者```demo```：
```sh
# 建立 demo Project(tenant)
openstack project create --description "Demo Project" demo

# 建立 demo User
openstack user create --password demo --email demo@example.com demo
openstack role create user
openstack role add --project demo --user demo user
```
> 你可以重複此過程來建立其他的 Projects 和 Users。

# 驗證操作
在安裝其他服務之前，我們要確認 Keystone 的是否沒問題。首先取消```OS_TOKEN```與```OS_URL```環境變數：
```sh
unset OS_TOKEN OS_URL
```

透過```admin```來驗證 Identity v2.0，請求一個```token```，記得輸入設定的密碼，這邊範例為```passwd```：
```sh
openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
token issue
```

成功後，會看到以下資訊：
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-23T15:36:06Z             |
| id         | 995e854225fc45efac11e3bdc705a675 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```

接下來驗證 Identity v3.0，因為 v3.0 增加了包含對 Project 與 User 的 Domain 的支援。Project 與 User 可以在不同的 Domain 使用相同名稱，因此要使用 v3.0 API，請求至少必須顯示包含```default domain```或者```User ID```。為了簡化驗證，這邊用```default domain```，這樣範例可以用使用者帳號名稱，而不是透過ID，指令如下：
```sh
openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
token issue
```

成功的話，會如上資訊一樣。
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-23T15:48:32.147152Z      |
| id         | 67ef08f69381404a8ccaf2a4725b58d5 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```

接下來透過```admin```來列出所有的```project (tenant)```，來驗證```admin```權限：
```sh
openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
project list
```

成功的話，會看到類似以下資訊：
```
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 675b421d5f794e10b924bffcfba6b3ab | demo    |
| b1214dcf4156464d8c6df2c20fd51e73 | service |
| cf8f9b8b009b429488049bb2332dc311 | admin   |
+----------------------------------+---------+
```

也可以透過```admin```來列出所有```user```：
```sh
openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
user list
```

成功會看到類似以下資訊：
```
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| a2cdc03624c04bb0bd7437f6e9f7913e | admin |
| f884a294541346d48b7e071b64011c97 | demo  |
+----------------------------------+-------+
```

然後列出所有```role```：
```sh
openstack --os-auth-url http://10.0.0.11:35357 \
--os-project-name admin \
--os-username admin \
--os-auth-type password \
role list
```

成功會看到以下資訊，這會隨著```role```的不同，而改變：
```
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 5ba17be99c624ba99046c8fddef812c2 | admin |
| d3fd2f8c04a64e6e9fe56bff410e5fe2 | user  |
+----------------------------------+-------+
```

接下來我們透過```demo```來驗證權限，先利用v3.0來獲取```token```：
```sh
openstack --os-auth-url http://10.0.0.11:5000 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name demo \
--os-username demo \
--os-auth-type password \
token issue
```

成功會看到回傳以下資訊：
```
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-23T15:55:20.322970Z      |
| id         | eabfa1621ea74747942feac618267d41 |
| project_id | 675b421d5f794e10b924bffcfba6b3ab |
| user_id    | f884a294541346d48b7e071b64011c97 |
+------------+----------------------------------+
```
> 注意！這邊採用```5000``` port 是因為該 port 允許正常的存取(非管理用) Keystone API。

透過```demo```來嘗試獲取擁有權限的操作：
```sh
openstack --os-auth-url http://10.0.0.11:5000 \
--os-project-domain-id default \
--os-user-domain-id default \
--os-project-name demo \
--os-username demo \
--os-auth-type password \
user list
```
若請求成功，會看到錯誤資訊：
```
ERROR: openstack You are not authorized to perform the requested action: admin_required
```
若以上都正確，代表```keystone```應該是正常的被執行了。

# 建立一個 Client 端腳本
我們會分別為```admin```與```demo```建立腳本，來方便我們做操作與驗證，首先建立```admin```檔案為```admin-openrc```，並加入以下參數：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=passwd
export OS_AUTH_URL=http://10.0.0.11:35357/v3
export OS_IDENTITY_API_VERSION=3
```
> 注意！```OS_PASSWORD```記得修改為設定密碼，這邊採用```passwd```。

再來建立```demo```為```demo-openrc```，並加入以下參數：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://10.0.0.11:5000/v3
export OS_IDENTITY_API_VERSION=3
```
> 注意！```OS_PASSWORD```記得修改為設定密碼，這邊採用```demo```。

完成後，可以透過```source```來加入環境變數：
```sh
source admin-openrc.sh
```

試看看直接請求```keystone```指令：
```sh
openstack token issue
```

成功的話，會看到以下資訊：
```sh
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-06-23T16:05:15.848947Z      |
| id         | 241fe6e046df47d6bd1a8f15c7f4f9b5 |
| project_id | cf8f9b8b009b429488049bb2332dc311 |
| user_id    | a2cdc03624c04bb0bd7437f6e9f7913e |
+------------+----------------------------------+
```
