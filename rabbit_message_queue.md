# Rabbit Message Queue
Rabbit Message Queue是實作了```AMQP（Advanced Message Queuing Protocol ）```的軟體套件，是導向訊息的中介軟體，RabbitMQ Server是透過```Erlang```語言撰寫而成，它所能做就是處理數位化資料的訊息接收，再把訊息發送出去。而在叢集與故障轉移是建構於開發電信平台框架上，所以支援了多程式語言的代理介面通訊的客戶端Library。

許多人會透過 RabbitMQ 訊息佇列系統來整合分散式叢集系統，讓不同的系統協同運作。我們可以在Ubuntu OS 透過內建套件來安裝 RabbitMQ：
```sh
wget http://www.rabbitmq.com/rabbitmq-signing-key-public.asc
sudo apt-key add rabbitmq-signing-key-public.asc
echo "deb http://www.rabbitmq.com/debian/ testing main" | sudo tee -a /etc/apt/sources.list

sudo apt-get update
sudo apt-get install rabbitmq-server
```

### Components
在進入開發RabbitMQ前，我們要知道組成該Message Queue套件的三個主要元件：
* **Producer**：訊息的發送端。
* **Queue**：在RabbitMQ內部中，作為暫時儲存訊息，在儲存空間足夠的狀況下，它可以保存任意數量的訊息，多個 producer 可以將訊息發送至同一個 queue 中，而不同的 consumer 也可以從同一個 queue 接收訊息。
* **Consumer**：訊息的接收端。

### Python AMPQ Lbirary
以 Python 來說可以使用下面這幾種：
* **py-amqplib**
* **txAMQP**
* **Pika**


我們可以使用```rabbitmqctl```的指令，來列出已建立的queue：
```sh
sudo rabbitmqctl list_queues
```

http://blogger.gtwang.org/2014/01/ubuntu-linux-install-rabbitmq.html

