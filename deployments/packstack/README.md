# PackStack all-in-one 安裝
本節將介紹如何透過 PackStack 安裝 OpenStack 單一節點核心套件，由於 PackStack 只能在 CentOS、RHEL 與 Fedora 進行使用，故這邊要準備安裝上述幾項的作業系統主機，設備建議規格如下所示：

| Role       	| RAM         	| Disk            	| CPUs       	|
|------------	|-------------	|-----------------	|------------	|
| all-in-one 	| 8 GB 記憶體 	| 250 GB 儲存空間 	| 四核處理器 	|

首先確認系統是 English 語系，編輯```/etc/environment```設定檔，加入以下內容：
```
LANG=en_US.utf-8
LC_ALL=en_US.utf-8
```

接著要取得最新版本的 OpenStack 套件來源，若是 RHEL 使用以下指令：
```sh
$ sudo yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
$ sudo yum update -y
$ sudo yum install -y openstack-packstack
$ packstack --allinone
```

若是 CentOS 則使用以下指令：
```sh
$ sudo yum install -y centos-release-openstack-mitaka
$ sudo yum update -y
$ sudo yum install -y openstack-packstack
$ packstack --allinone
```

完成後就可以開啟 [Dashboard](http://10.0.0.11/dashboard)。
> Username 為 ```admin```，Passwd 可以在```/root```目錄底下的```keystonerc_admin```檔案找到。
