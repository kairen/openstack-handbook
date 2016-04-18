# NFS Driver 設定
首先到 ```NFS Server``` 節點，新增分享的檔案系統目錄：
```sh
$ sudo mkdir -p /var/nfs/volumes
```

然後編輯```/etc/exports```檔案，來設定分享的目錄：
```
/var/nfs/volumes 10.0.0.0/24(rw,sync,no_root_squash,no_subtree_check)
```

完成後重新啟動 NFS Server，如以下指令：
```sh
$ sudo /etc/init.d/nfs-kernel-server restart
```

接著到 ```Cinder volume``` 節點，首先安裝相依套件：
```sh
$ sudo apt-get -y install nfs-common
```

然後要建立檔名為```/etc/cinder/nfs_shares```的檔案，用來給 Cinder 使用的 NFS conf：
```
10.0.0.61:/var/nfs/volumes
```
> 這邊```10.0.0.61```為 NFS Server，後面代表要掛載的目錄。

接著編輯```/etc/cinder/cinder.conf```設定檔，在```[DEFAULT]```部分加入以下內容：
```
[DEFAULT]
...
enabled_backends = nfs
```

在```[nfs]```部分加入以下內容：
```
[nfs]
nfs_shares_config = /etc/cinder/nfs_shares
volume_driver = cinder.volume.drivers.nfs.NfsDriver
volume_backend_name = nfs-backend
nfs_sparsed_volumes = True
```

完成後重新啟動 Cinder volume 服務：
```sh
$ sudo service cinder-volume restart
```

然後透過以下指令檢查是否正常被掛載：
```sh
$ df -hT
10.0.0.61:/var/nfs/volumes nfs4      230G  5.2G  213G   3% /var/lib/cinder/mnt/5131a84be0970fe412fdb53aa092f403
```

接著會因為掛載目錄有權限問題，故修改目錄權限給 cinder 使用：
```sh
$ sudo chown -R cinder:cinder /var/lib/cinder/
```

完成後即可測試，當建立一個 volume 後，可以用以下方式查看區塊裝置實體位置：
```sh
$ ls /var/lib/cinder/mnt/5131a84be0970fe412fdb53aa092f403
volume-0a8349e8-d53b-447d-8ed6-f08b4ddf04ee
```

# Backend Types
由於可能會建立來至不同地方或者不同儲存裝置所提供的 NFS Server，這時區分種類就非常的有用，可以透過以下方式來建立對應 Type 的後台儲存：
```sh
$ cinder type-create TYPE
$ cinder type-key TYPE set volume_backend_name=BACKEND_NAME
```
> 其中 ```TYPE``` 為一個後台儲存型態，這邊為 ```nfs```。```BACKEND_NAME```則是 cinder group 中的名稱，這邊為```nfs-backend```。
