# MAAS+JuJu
首先為了方便系統操作，將```sudo```設置為不需要密碼：
```sh
echo "openstack ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/openstack && sudo chmod 440 /etc/sudoers.d/openstack
```
> ```openstack``` 為 User Name。

安裝相關套件：
```sh
sudo apt-get install software-properties-common
```

新增 MAAS 與 JuJu 的資源庫：
```sh
sudo add-apt-repository -y ppa:juju/stable
sudo add-apt-repository -y ppa:maas/stable
sudo add-apt-repository -y ppa:cloud-installer/stable
sudo apt-get update
```
接著安裝 MAAS：
```sh
sudo apt-get install -y maas
```
安裝完成後，建立一個 admin 使用者帳號：
```sh
sudo maas-region-admin createadmin
```
> 這時候就可以登入 [MAAS](http://<maas.ip>/MAAS/)。

安裝 JuJu quick start：
```sh
sudo apt-get update && sudo apt-get install juju-quickstart
```
完成後，透過以下指令來部署 JuJu GUI：
```sh
$ juju-quickstart --gui-port 4242
```

### 客製化 Images
首先安裝 Maas Image Builder 來建立映像檔：
```sh
$ sudo add-apt-repository -y ppa:blake-rouse/mib-daily
$ sudo apt-get install maas-image-builder
```
安裝完成後，我們可以使用以下指令建立映像檔，如以下為建立一個```CentOS 7```的映像檔：
```sh
$ sudo maas-image-builder -a amd64 -o centos7-amd64-root-tgz centos --edition 7
```
> 目前 CentOS 支援了```6.5```與```7.0```版本。

當映像檔建立完成後，就可以上傳到 MAAS 了，在這之前需要登入 MAAS，才有權限上傳：
```sh
$  maas login <user_name> <maas_url> <api_key>
```
> ```<user_name>```為使用者帳號名稱。
> ```<maas_url>```為 MAAS 網址，如 http://192.168.1.2/MAAS
> ```<api_key>```為帳號 API Key。

登入後，就可以使用以下指令進行上傳客製化的映像檔了：
```sh
$ maas <user_name> boot-resources create name=centos/centos7 architecture=amd64/generic content@=./build-output/centos7-amd64-root-tgz
```
> 登入時，要輸入指令需加```<user_name>```。