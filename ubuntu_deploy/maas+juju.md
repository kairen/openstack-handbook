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


接著進入 ```Images``` 頁面，並選擇 import ```14.04 LTS amd64``` 映像檔：
![](images/import.png)

完成後，到```Networks```頁面，會看到安裝 maas 自動建立的網路，這時點選編輯網路來新增```default gateway```與```DNS server```：
![](images/network.png)

安裝 JuJu quick start：
```sh
sudo apt-get update && sudo apt-get install juju-quickstart
```