# DevStack Trove
要透過 DevStack 安裝 Trove 服務，我們可以透過 trove-integration 這個專案來進行佈署，首先將專案抓到本地端：
```sh
$ git clone https://github.com/openstack/trove-integration.git
```

然後設定系統使用者不需要輸入密碼即可使用 sudo 權限：
```sh
$ echo "ubuntu ALL = (root) NOPASSWD:ALL" \
| sudo tee /etc/sudoers.d/ubuntu && sudo chmod 440 /etc/sudoers.d/ubuntu
```

進入安裝腳本目錄：
```sh
$ cd trove-integration/scripts/
```

然後透過以下指令進行安裝相依套件與佈署 Trove 服務：
```sh
$ ./redstack install
```

若過程沒有錯誤即可建置 Trove 服務使用的映像檔，並上傳到 Glance 服務：
```sh
$ ./redstack build-image mysql
```
