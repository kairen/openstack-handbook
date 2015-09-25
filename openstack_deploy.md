# OpenStack 部署與安裝
本章節將部署過程進行整理，其部署包含Devstack、PackStack、Manual等。


```sh
virt-install --connect qemu:///system \
  --name windows7 --ram 2048 --vcpus 2 \
  --network network=default,model=virtio \
  --disk path=windows7.qcow2,format=qcow2,device=disk,bus=virtio \
  --cdrom /home/k8s/build_images/windows7.iso \
  --disk path=/home/k8s/build_images/virtio-win-0.1.105.iso,device=cdrom \
  --vnc --os-type windows --os-variant win7
```