# Nova Docker 部署
本節將安裝 compute 節點使用 ```DockerDriver```。

### Install Docker on compute node
```sh
$ sudo apt-get install apt-transport-https
$ sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
$ sudo bash -c "echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list"
$ sudo apt-get update
$ sudo apt-get install -y lxc-docker
$ sudo usermod -G docker nova
$ sudo chmod 666 /var/run/docker.sock
$ sudo chmod 777 /var/run/libvirt/libvirt-sock
```

### Install nova-docker drive on compute node
```sh
$ sudo apt-get install python-pip python-dev
$ git clone -b stable/kilo https://github.com/stackforge/nova-docker.git
$ cd nova-docker
$ sudo python setup.py install
$ sudo pip list | grep nova-docker
$ sudo cp nova-docker/etc/nova/rootwrap.d/docker.filters \
  /etc/nova/rootwrap.d/
```

### Nova configuration in the compute node
```sh
$ sudo vim /etc/nova/nova-compute.conf
----------
[DEFAULT]
compute_driver = novadocker.virt.docker.DockerDriver

# compute_driver = libvirt.LibvirtDriver
# [libvirt]
# virt_type=kvm
----------
```

```sh
$ sudo service nova-compute restart
$ sudo service docker restart

# Glance configuration in the controller node
$ sudo vim /etc/glance/glance-api.conf
----------
[DEFAULT]
container_formats = ami,ari,aki,bare,ovf,docker
----------
```

```sh
$ sudo service glance-api restart
$ sudo service glance-registry restart
```

### Create docker images and export to glance
```sh
$ docker pull nginx
$ docker save nginx > nginx.tar
$ glance image-create --file nginx.tar --container-format docker --disk-format raw --name nginx
$ nova boot --image nginx --flavor m1.tiny --nic net-id=aa85fed0-c500-48de-b2f2-bbc9c4ce3022 --availability_zone=nova:compute3 nginx 
```