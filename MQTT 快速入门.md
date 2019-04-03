# MQTT 快速入门

现在我们来快速体验一下，使用MQTT进行数据的发布和订阅。

考虑到Mosquitto比较适合初学者，所以选择它来做实验。实验环境是Windows 10 64 bit，Mosquitto版本是1.5.8。

### Mosquitto安装

- 进入[Mosquitto](<https://mosquitto.org/download/>)下载页面，选择对应的操作系统下载安装。Windows用户下载安装包，一直点击下一步到安装结束，系统默认会安装到"C:\Program Files\mosquitto"，为方便使用，将这个路径加到Path环境变量中。安装成功后，会添加一个叫Mosquitto Broker的系统服务，并且会自动启动。

- 这里顺便讲一下，如果是CentOS用户，可以直接使用yum install mosquitto命令进行安装的。其它平台的安装参见下载页，有详细说明。

- 使用windows + r，执行命令services.msc，打开windows服务，找到Mosquitto Broker服务，查看服务状态，如果服务没启动，则启动服务。现在就可以连接到Broker了。


### 订阅主题

打开命令行终端，输入如下命令订阅主题topic1。

```
$ mosquitto_sub -d -t topic1 
Client mosqsub|3508-SCNWCL0121 sending CONNECT
Client mosqsub|3508-SCNWCL0121 received CONNACK (0)
Client mosqsub|3508-SCNWCL0121 sending SUBSCRIBE (Mid: 1, Topic: topic1, QoS: 0)
Client mosqsub|3508-SCNWCL0121 received SUBACK
Subscribed (mid: 1): 0
```

- 以上第一行是命令，使用mosquitto_sub命令订阅主题。

  -d 参数表示启用debug模式，这样mosquitto_sub会显示详细的连接以及数据收发过程。

  -t topic1 表示需要订阅主题topic1。

  这里有些参数没写，都使用了默认值，如host使用了localhost，port使用了1883，并且使用了"mosqsub|3508-SCNWCL0121"作为Client ID，其中3508是进程Id，SCNWCL0121是我的机器名。

  如果要指定host，port以及Client Id，可以这样使用。

  ```
  $ mosquitto_sub -d -h localhost -p 1883 -i subscriber-test -t topic1
  Client subscriber-test sending CONNECT
  Client subscriber-test received CONNACK (0)
  Client subscriber-test sending SUBSCRIBE (Mid: 1, Topic: topic1, QoS: 0)
  Client subscriber-test received SUBACK
  Subscribed (mid: 1): 0
  ```

  -h 表示host

  -p 表示port

  -i 表示客户端ID。

  其实所有的命令行使用都可以使用--help进行查阅，输入

  ```
  $ mosquitto_sub --help
  ```

  你就会看到它的详细用法，所有参数都显示出来了，很详细。

- 第二行和第三行是建立连接的过程。

  ```
  Client mosqsub|3508-SCNWCL0121 sending CONNECT
  Client mosqsub|3508-SCNWCL0121 received CONNACK (0)
  ```

  表示客户端“mosqsub|3508-SCNWCL0121”发送CONNECT，同时Broker回复CONNACK (0)。其中0是状态码，表示连接成功。如果是其它数字，则表示连接失败。失败的原因有很多，比如不支持当前协议，服务器不可用等等，具体可参见[Connect Return Code](<http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718035>)。

- 第四行到第六行是订阅主题的过程，

  ```
  Client mosqsub|3508-SCNWCL0121 sending SUBSCRIBE (Mid: 1, Topic: topic1, QoS: 0)
  Client mosqsub|3508-SCNWCL0121 received SUBACK
  Subscribed (mid: 1): 0
  ```

  sending SUBSCRIBE (Mid: 1, Topic: topic1, QoS: 0)表示发送订阅请求，

  Mid是Message Id，从1开始计算，当一个连接发送多条消息时，Mid是递增的。

  Topic:topic1表示要订阅的主题时topic1。

  QoS：0指定了QoS等级，默认是0。


### 发布消息

打开一个新的命令行终端，输入以下命令

```
$ mosquitto_pub -d -t topic1 -m "Hello MQTT"
Client mosqpub|12796-SCNWCL012 sending CONNECT
Client mosqpub|12796-SCNWCL012 received CONNACK (0)
Client mosqpub|12796-SCNWCL012 sending PUBLISH (d0, q0, r0, m1, 'topic1', ... (10 bytes))
Client mosqpub|12796-SCNWCL012 sending DISCONNECT
```

- 第一行是发布消息命令，将消息发布到主题topic1，-m 指定了发送消息的内容是"Hello MQTT"。我们同样启用了debug模式。

- 第二行和第三行是连接过程

- 第四行是发布的消息信息。

  ```
  Client mosqpub|12796-SCNWCL012 sending PUBLISH (d0, q0, r0, m1, 'topic1', ... (10 bytes))
  ```

  其中d0, q0, r0, m1,分别是发布消息指定的参数，这里使用默认参数。

  d0表示DUP为0，DUP是是否重复标记，如果是第一次发送消息，则设置为0。如果是重复投递，比如QoS设置为1，客户端发送消息超时后服务器还没有回复，客户端为确保消息能发出去，于是再发一次，这是DUP就设置为1，表明这个消息是重复发送的。

  q0表示QoS为0。

  r0表示RETAIN为0。RETAIN意思是是否要求Broker帮我保留这条消息，如果设置为1，则服务器会保留当前消息。当下一次有新的客户端连接并订阅topic1时，服务器自动发送这条保留的消息给客户端。

  m1表示消息序号，默认从1开始。

  topic1是发布到这个主题。

  ... (10 bytes)没有显示消息内容，但是显示了消息长度是10个字节。

- 最后一行是断开连接。

  ```
  Client mosqpub|12796-SCNWCL012 sending DISCONNECT
  ```


### 接收消息

发布完消息后，再回到之前订阅的终端，会显示接收到的消息。

```
Client mosqsub|11104-SCNWCL012 received PUBLISH (d0, q0, r0, m0, 'topic1', ... (10 bytes))
topic1 Hello MQTT
```

第一行显示收到PUBLISH数据包，第二行打印出接收到的数据。



### 总结

Mosquitto是很容易使用的MQTT实现，包含了服务端和客户端。在这个实验中，我们其实就执行了两条命令。

```
$ mosquitto_sub -d -t topic1 
$ mosquitto_pub -d -t topic1 -m "Hello MQTT"
```

分别表示订阅主题和发布消息，当另一个客户端发送消息成功后，订阅端会收到消息并打印出来。

在以上命令中，-d参数非常有用，是我们学习MQTT协议的利器，这里举个列子。

QoS是服务质量保证，在发布消息时，当QoS设置为0，那么客户端发送消息后，Broker是不做回复的。

QoS设置为1时，客户端发送消息后，会等待Broker确认，如果等不到PUBACK，那么过一段时间后会重新发送。这样确保Broker能收到消息。我们来对比一下。

```
$ mosquitto_pub -d -t topic1 -m "Hello MQTT"
Client mosqpub|6188-SCNWCL0121 sending CONNECT
Client mosqpub|6188-SCNWCL0121 received CONNACK (0)
Client mosqpub|6188-SCNWCL0121 sending PUBLISH (d0, q0, r0, m1, 'topic1', ... (10 bytes))
Client mosqpub|6188-SCNWCL0121 sending DISCONNECT
```

```
$ mosquitto_pub -d -q 1 -t topic1 -m "Hello MQTT"
Client mosqpub|14788-SCNWCL012 sending CONNECT
Client mosqpub|14788-SCNWCL012 received CONNACK (0)
Client mosqpub|14788-SCNWCL012 sending PUBLISH (d0, q1, r0, m1, 'topic1', ... (10 bytes))
Client mosqpub|14788-SCNWCL012 received PUBACK (Mid: 1)
Client mosqpub|14788-SCNWCL012 sending DISCONNECT
```

当我们发送消息时，如果增加参数-q 1，表示QoS设置成1。数据包就会多出一个回复。

```
Client mosqpub|14788-SCNWCL012 received PUBACK (Mid: 1)
```

QoS的实现机制比较复杂，后续我会专门写一篇文章讲[MQTT QoS](<http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html#_Toc398718099>)，有兴趣的可以自己点击链接先去看看。

