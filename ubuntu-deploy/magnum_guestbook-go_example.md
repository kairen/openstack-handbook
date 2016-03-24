# Magnum Geustbook-Go Example
以下為透過 Magnum 來建立一個簡單的 kubernetes 範例，我們採用 [Geustbook-Go](https://github.com/kubernetes/kubernetes/tree/master/examples/guestbook-go) 來進行教學，首先下載範例檔案：
```sh
$ git clone https://github.com/kubernetes/kubernetes.git
$ cd kubernetes/examples/guestbook
```

### 步驟一: 建立 Redis master pod
我們使用```redis-master-controller.json```檔案來建立 Redis master：
```sh
$ magnum rc-create --manifest  ./redis-master-controller.yaml --bay k8sbay
```