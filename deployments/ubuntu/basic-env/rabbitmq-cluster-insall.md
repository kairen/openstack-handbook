# RabbitMQ Cluster
RabbitMQ 是實現 AMQP（高級消息隊列協議）的消息中間件的一種，最初起源於金融系統，用於在分佈式系統中存儲轉發消息，在易用性、擴展性、高可用性等方面表現不俗。

RabbitMQ主要是為了實現系統之間的雙向解耦而實現的。當生產者大量產生數據時，消費者無法快速消費，那麼需要一個中間層。保存這個數據。

### 叢集環境說明
本教學環境採用三台節點進行安裝，分別為：
* **Controller1**：10.0.0.11。
* **Controller2**：10.0.0.12。
* **Controller3**：10.0.0.13。

首先在各節點安裝 RabbitMQ server 套件：
```sh
$ sudo apt-get -y install rabbitmq-server
```
> 然後可選擇是否要在主節點啟用 RabbitMQ Management Web Console：
>
```sh
$ sudo rabbitmq-plugins enable rabbitmq_management
```
> 開啟遠端使用者可登入 web console：
>
```sh
$ sudo sh -c "echo '[{rabbit, [{loopback_users, []}]}].' > /etc/rabbitmq/rabbitmq.config"
$ sudo service rabbitmq-server restart
```
完成後進入```http://<ip>:15672 ```，輸入```guest/guest```。


同步 ERLang Cookie，在同步 ERLang Cookie 之前，必須先將```slave nodes```的 RabiitMQ 服務停止：
```sh
$ sudo service rabbitmq-server stop
```

備份```slave nods```上的 ERLang Cookie：
```sh
$ sudo cp /var/lib/rabbitmq/.erlang.cookie /var/lib/rabbitmq/.erlang.cookie.bak
```
將 master node 上的 ERLang Cookie 複製到 slave nodes 上：
```sh
$ sudo cat /var/lib/rabbitmq/.erlang.cookie | ssh controller2 sudo tee /var/lib/rabbitmq/.erlang.cookie
```
重新啟動```slave nodes```上的 rabbitmq-server 服務
```sh
$ sudo service rabbitmq-server start
```

設定 RabbitMQ Cluster，在 Master 上重新設定 RabbitMQ Service：
```sh
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl start_app
```

將 Slave nodes 加入 cluster 中：
```sh
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl join_cluster --ram rabbit@controller2
$ sudo rabbitmqctl start_app
```
> 改變 node 的型態可以使用以下指令：
```sh
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl change_cluster_node_type disc
$ sudo rabbitmqctl start_app
```

確認 cluster 狀態：
```sh
$ sudo rabbitmqctl cluster_status
[{nodes,[{disc,[rabbit@controller1]},
         {ram,[rabbit@controller3,rabbit@controller2]}]},
 {running_nodes,[rabbit@controller2,rabbit@controller3,rabbit@controller1]},
 {cluster_name,<<"rabbit@controller1">>},
 {partitions,[]}]
```

設定 queue HA policy，我們可以在任一個 node 上輸入以下命令：：
```sh
$ sudo rabbitmqctl set_policy ha-all '^(?!amq\.).*' '{"ha-mode": "all"}'
```

若要重新設定 RabbitMQ，可以使用以下指令：
```sh
$ sudo rabbitmqctl stop_app
$ sudo rabbitmqctl reset
$ sudo rabbitmqctl start_app
```
