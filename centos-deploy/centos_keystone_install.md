# Keystone 安裝與設定
本章節會說明與操作如何安裝```Keystone```服務到OpenStack Controller節點上，並設置相關參數與設定。若對於Keystone不瞭解的人，可以參考[Keystone 身份驗證套件章節](http://kairen.gitbooks.io/openstack/content/keystone/index.html)。

### 安裝前準備
我們需要在Database底下建立儲存Keystone資訊的資料庫，利用```mysql```指令進入：
```sh
mysql -u root -p
```
建立Keystone 資料庫與使用者：
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
> 這邊若```KEYSTONE_DBPASS```要更改的話，可以更改。

完成後，透過```quit```指令離開資料庫，並透過```openssl```建立一個隨機的admin token：
```sh
openssl rand -hex 10

# 會產生如下密碼
3193558db65508595553
```
### 安裝與設置Keystone套件
> 在```Kilo```的Keystone遺棄了```Eventlet```，改用```WSGI```server來代替。所以本教學關閉使用keystone服務。

首先透過```yum```安裝keystone套件：
```sh
yum install openstack-keystone httpd mod_wsgi python-openstackclient memcached python-memcached
```
開啟```memcached```服務，並設定Boot開啟：
```sh
systemctl enable memcached.service
systemctl start memcached.service
```
編輯```/etc/keystone/keystone.conf```，將ADMIN_TOKEN替換為上一步中，產生的隨機字串：
```
[DEFAULT]
...
admin_token = 3193558db65508595553
```
在```[database]```部分修改如下：
```
[database]
...
connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
```
在```[memcache]```部分，加入Memcache service：
```sh
[memcache]
...
servers = localhost:11211
```
在```[token]```部分修改如下：
```
[token]
...
provider = keystone.token.providers.uuid.Provider
driver = keystone.token.persistence.backends.memcache.Token
```
在```[revoke]```部分修改如下：
```
[revoke]
...
driver = keystone.contrib.revoke.backends.sql.Revoke
```
最後可以選擇是否要在```[DEFAULT]```中，開啟詳細Logs，為後期的故障排除提供幫助：
```
[DEFAULT]
...
verbose = True
```
完成後，同步資料庫：
```sh
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
### Apache HTTP server
編輯```/etc/httpd/conf/httpd.conf```檔案，修改以下：
```
ServerName controller
```
建立並編輯```/etc/httpd/conf.d/wsgi-keystone.conf```，加入以下：
```sh
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /var/www/cgi-bin/keystone/main
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LogLevel info
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /var/www/cgi-bin/keystone/admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LogLevel info
    ErrorLogFormat "%{cu}t %M"
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined
</VirtualHost>
```
建立一個資料夾，放WSGI元件使用：
```sh
 mkdir -p /var/www/cgi-bin/keystone
```
下載網路上的 WSGI 元件到剛建立的資料夾：
```
curl http://git.openstack.org/cgit/openstack/keystone/plain/httpd/keystone.py?h=stable/kilo \
  | tee /var/www/cgi-bin/keystone/main /var/www/cgi-bin/keystone/admin
```
改變擁有權與權限於WGSI資料夾：
```sh
chown -R keystone:keystone /var/www/cgi-bin/keystone
chmod 755 /var/www/cgi-bin/keystone/*
```
重新開啟HTTP服務：
```sh
systemctl enable httpd.service
systemctl start httpd.service
```

# 建立服務實體和API端點
首先透過```export```設定OS_TOKEN環境變數，輸入openssh建立的字串與API URL：
```sh
export OS_TOKEN=3193558db65508595553
export OS_URL=http://controller:35357/v2.0
```
建立服務實體和身份驗證服務：
```sh
openstack service create  --name keystone --description "OpenStack Identity" identity
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
> OpenStack 是動態產生ID 的，因此看到的輸出會跟範例中的輸出不同。

身份驗證服務管理了一個與環境相關的API 端點的目錄。服務使用這個目錄來決定如何與環境中的其他服務進行溝通。透過以下建立一個API端點：
```sh
openstack endpoint create --publicurl http://controller:5000/v2.0 --internalurl http://controller:5000/v2.0 --adminurl http://controller:35357/v2.0  --region RegionOne identity
```
會看到產生類似以下資訊：
```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| adminurl     | http://controller:35357/v2.0     |
| id           | dd69029787b04bb890d9e6ef6624f082 |
| internalurl  | http://controller:5000/v2.0      |
| publicurl    | http://controller:5000/v2.0      |
| region       | RegionOne                        |
| service_id   | 317ef3c6861d40a3b5a04b8d14b4b276 |
| service_name | keystone                         |
| service_type | identity                         |
+--------------+----------------------------------+
```

> 添加到OpenStack環境中的每個服務，都需要一個或多個服務實體和身份驗證服務的一個API端點。

# 建立Projects、Users與Roles
驗證服務會對每一個Openstack的服務提供身份驗證。驗證服務使用domains, projects (tenants), users, 和roles的組合。在環境中，為了進行管理操作，我們要建立```admin```的Project、User和Role：
```sh
# 建立 admin Project(tenant)
openstack project create --description "Admin Project" admin
# 建立 admin User
openstack user create --password ADMIN --email admin@example.com admin
# 建立 admin Role
openstack role create admin
# 將 admin role 加到 project與user
openstack role add --project admin --user admin admin
# 建立 service project
openstack project create --description "Service Project" service
```
為了驗證，我們建立一個沒有特權的使用者```Demo```：
```sh
# 建立 demo Project(tenant)
openstack project create --description "Demo Project" demo
# 建立 demo User
openstack user create --password demo --email demo@example.com demo
openstack role create user
openstack role add --project demo --user demo user
```
> 你可以重複此過程來建立其他的Projects和Users。

# 驗證操作
在安裝其他服務之前，我們要確認Keystone的是否沒問題。因為安全問題，可以選擇是否關閉臨時驗證token機制，編輯```/usr/share/keystone/keystone-dist-paste.ini```，移除以下三個區域的```admin_token_auth```參數：
```
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth [admin_token_auth] json_body ec2_extension user_crud_extension public_service

[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth [admin_token_auth] json_body ec2_extension s3_extension crud_extension admin_service

[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth [admin_token_auth] json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
```
再來取消```OS_TOKEN```與```OS_URL```環境變數：
```sh
unset OS_TOKEN OS_URL
```
透過```admin```來驗證Identity v2.0，請求一個```token```，記得輸入設定的密碼，這邊範例為```ADMIN```：
```sh
openstack --os-auth-url http://controller:35357 --os-project-name admin --os-username admin --os-auth-type password  token issue
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
接下來驗證Identity v3.0，因為v3.0增加了對包含Project與User的Domain的支援。Project與User可以在不同的Domain使用相同名稱，因此要使用v3.0 API，請求至少必須顯示包含```default domain``或者```User ID```。為了簡化驗證，這邊用```default domain``，這樣範例可以用使用者帳號名稱，而不是透過ID，指令如下：
```sh
openstack --os-auth-url http://controller:35357  --os-project-domain-id default --os-user-domain-id default  --os-project-name admin --os-username admin --os-auth-type password  token issue
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
openstack --os-auth-url http://controller:35357  --os-project-name admin --os-username admin --os-auth-type password  project list
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
openstack --os-auth-url http://controller:35357  --os-project-name admin --os-username admin --os-auth-type password  user list
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
openstack --os-auth-url http://controller:35357  --os-project-name admin --os-username admin --os-auth-type password role list
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
openstack --os-auth-url http://controller:5000  --os-project-domain-id default --os-user-domain-id default  --os-project-name demo --os-username demo --os-auth-type password  token issue
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
> 注意！這邊採用```5000``` port是因為該port允許正常的存取(非管理用) Keystone API。

透過```demo```來嘗試獲取擁有權限的操作：
```sh
openstack --os-auth-url http://controller:5000  --os-project-domain-id default --os-user-domain-id default  --os-project-name demo --os-username demo --os-auth-type password  user list
```
若請求成功，會看到錯誤資訊：
```
ERROR: openstack You are not authorized to perform the requested action: admin_required
```
若以上都正確，代表```keystone```就是正常運作。

# 建立一個 Client 端腳本
我們會分別為```admin```與```demo```建立腳本，來方便我們做操作與驗證，首先建立```admin```檔案為```admin-openrc.sh```，並加入以下參數：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN
export OS_AUTH_URL=http://controller:35357/v3
```
> 注意！```OS_PASSWORD```記得修改為設定密碼，這邊採用```ADMIN```。

再來建立```demo```為```demo-openrc.sh```，並加入以下參數：
```sh
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://controller:5000/v3
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

