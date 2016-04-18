# 選項一:無驅動程式支援
這邊為了方便，直接使用同區塊儲存的 Storage 節點配置。但是 LVM 驅動程式需要一個獨立的區塊儲存裝置，以避免與區塊儲存服務發生衝突。該儲存裝置使用```/dev/sdc```，但也可以隨硬體不同而更改。

- [安裝前準備](#安裝前準備)
- [無驅動程式支援設定](#無驅動程式支援設定)

### 安裝前準備
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install lvm2 nfs-kernel-server
```

由於本教學採用 LVM（Logical Volume Manager）來提供儲存給虛擬機使用，因此這邊需要先建立 LVM Physical Volume。這邊將 Storage 節點的 ```/dev/sdb``` 硬碟當作 Physical Volume 使用：
```sh
$ sudo pvcreate /dev/sdb
Physical volume "/dev/sdb" successfully created
```
> 若有多顆硬碟則重複方式建立。
> 若該硬碟有過去使用的資料與分區的話，可以使用以下兩個指令來解決：
```sh
$ sudo fdisk /dev/sdb
$ sudo mkfs -t ext4 /dev/sdb
```

接著要建立一個 LVM Volume Group 來讓多顆硬碟當作一個邏輯儲存使用：
```sh
$ sudo vgcreate manila-volumes /dev/sdb
Volume group "cinder-volumes" successfully created
```

由於 Manila 會使用被建立成 LVM 的裝置來提供共享式檔案系統。為了確保儲存安全問題，故任何作業系統底層不應該被任意存取該 LVM volume。且預設下的 LVM 會透過工具搜尋包含 ```/dev``` 的區塊儲存裝置目錄。如果部署時是使用 LVM 來提供 Volume 的話，該工具會檢查這些 Volume，並試著快取目錄，這樣將會造成各式各樣的問題，因此要編輯 ```/etc/lvm/lvm.conf``` 來正確的提供 Volume Group 的硬碟使用。這邊設定只使用 ```/dev/sdb```：
```sh
filter = [ 'a/sdb/', 'r/.*/']
```
> 這邊以 ```a``` 開頭的表示 accept，而 ```r``` 則表示 reject。

當完成上述過程後，最後要確認建置無誤，透過以下指令來查看 Volume Group：
```sh
$ sudo  pvdisplay
--- Physical volume ---
PV Name               /dev/sdb
VG Name               cinder-volumes
PV Size               465.81 GiB / not usable 3.18 MiB
Allocatable           yes
PE Size               4.00 MiB
Total PE              59618
Free PE               59362
Allocated PE          256
PV UUID               tsYzd6-a4s8-32iL-aYdA-AMol-qtj2-4RMhvA
```

### 無驅動程式支援設定
由於預設下配置檔案分散有所不同。因此需要增加這些 [section] 與 options，而不是修改現有的 [section] 與 options。

首先編輯```/etc/manila/manila.conf```設定檔，並在```[DEFAULT]```部分設定以下內容：
```
[DEFAULT]
...
enabled_share_backends = lvm
enabled_share_protocols = NFS,CIFS
```
> 注意這邊的 backend names 是任意字串，如同 Cinder。

在```[lvm]```部分加入以下內容：
```sh
[lvm]
share_backend_name = LVM
share_driver = manila.share.drivers.lvm.LVMShareDriver
driver_handles_share_servers = False
lvm_share_volume_group = manila-volumes
lvm_share_export_ip = MANAGEMENT_IP
```
> P.S. ```MANAGEMENT_IP```這邊為```10.0.0.61```。

完成後，回到 [配置共享伺服器管理支援選項](README.md#配置共享伺服器管理支援選項) 重新啟動服務。
