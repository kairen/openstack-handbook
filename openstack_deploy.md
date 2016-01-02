# OpenStack 部署與安裝
本章節將部署過程進行整理，其部署包含Devstack、PackStack、Manual等。


```sh
virt-install --virt-type kvm --name centos-6.7 --ram 1024 \
--disk centos-6.7.qcow2,format=qcow2 \
--network network=default \
--graphics spice,listen=0.0.0.0 \
--os-type=linux --os-variant=rhel6 \
--extra-args="console=tty0 console=ttyS0,115200n8 serial" \
--location=CentOS-6.7-x86_64-netinstall.iso
```

