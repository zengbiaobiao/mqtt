# MQTT 协议 -- Connect

### 概述

今天来学习MQTT协议中关于connect部分，connect是很重要的部分，因为它是Client 与MQTT Broker通信的基础，并且提供了很多很有用的特性，很多场景中都可以用到这些特性。

还是协议结合着理论结合着来讲吧，否则担心小伙伴们看了睡觉。~~~

前面已经讲过了，MQTT是一种基于发布订阅的消息传输协议，所以MQTT发布客户端可以发布消息到1个或多个订阅客户端。这个模式很像电视或者收音机的广播，电台发布节目，千家万户的观众接收节目。所有的消息都是发给MQTT Broker，MQTT Broker再将消息转发给它的订阅者。在这个过程中，需要注意以下几点：

- 所有客户端都有一个唯一Id，这个Id只是一个标记，不是客户地址。发布端发布消息时，只能发给某个topic，而不能将消息发给某个地址或Id。
- 客户端的Id不能重复。如果有一个客户端Client A连到MQTT Broker，之后又有一个客户端Client B连到MQTT Broker，此时MQTT Broker会断开Client A的连接。因为MQTT 客户端有自动连接功能，Client A断开连接之后，尝试重新连接到MQTT Broker，此时Client B就会断开，Client B又进行重连，然后两个Client就会进入断开-连接断开的死循环。
- MQTT Broker负责接收消息，并进行过滤，将消息转发给订阅了相关主题的Client。
- Publisher和Subscriber没有直接的关联。他们都只与MQTT Broker进行连接。
- 客户端既可以发送消息，也可以接收消息。
- 通常情况下，MQTT Broker是不存储消息的。

现在应该对MQTT Client和Broker有一个比较清楚的认识了。我们来讨论MQTT Connect的格式吧。

---

### Fixed Header

MQTT的固定头部包含了首字节和可变长度。其中首字节的高4位(bit7~bit4)用于表示报文类型，1表示connect。其它标记字节(bit3~bit0)都为0，如下表。

| Bit       | 7         | 6      | 5    | 4    | 3    | 2    | 1    | 0    |
| --------- | --------- | ------ | ---- | ---- | ---- | ---- | ---- | ---- |
| Byte 1    | 0         | 0      | 0    | 1    | 0    | 0    | 0    | 0    |
| Byte 2... | Remaining | Length |      |      |      |      |      |      |



---

### Variable Header

在Connect报文中，可变头部包含8个字节，如下表：

| Byte    | Description    | bit7      | bit6     | bit5        | bi4      | bit3     | bit2      | bit1          | bit0     |
| ------- | -------------- | --------- | -------- | ----------- | -------- | -------- | --------- | ------------- | -------- |
| Byte 1  | Length MSB (0) | 0         | 0        | 0           | 0        | 0        | 0         | 0             | 0        |
| Byte 2  | Length LSB (4) | 0         | 0        | 0           | 0        | 0        | 1         | 0             | 0        |
| Byte 3  | 'M'            | 0         | 1        | 0           | 0        | 1        | 1         | 0             | 1        |
| Byte 4  | 'Q'            | 0         | 1        | 0           | 1        | 0        | 0         | 0             | 1        |
| Byte 5  | 'T'            | 0         | 1        | 0           | 1        | 0        | 1         | 0             | 0        |
| Byte 6  | 'T'            | 0         | 1        | 0           | 1        | 0        | 1         | 0             | 0        |
| Byte 7  | Level(4)       | 0         | 0        | 0           | 0        | 0        | 1         | 0             | 0        |
| Byte 8  | Connect Flag   | User Name | Password | Will Retain | Will QoS | Will QoS | Will Flag | Clean Session | Reserved |
| Byte 9  | Keep Alive MSB | 0         | 0        | 0           | 0        | 0        | 0         | 0             | 0        |
| Byte 10 | Keep Alive MSB | 0         | 1        | 1           | 0        | 0        | 0         | 0             | 0        |

可变头不的内容包含Protocol Name, Protocol Level, Connect Flags以及Keep Alive时间。下面分别介绍：

**Protocol Name:** 字节1-6，这部分内容是固定的，其中字节1和字节2表示协议名称长度，其内容是0x04。字节3-字节6表示协议名称"MQTT"的UTF-8编码。

**Protocol Level:** 字节7，表示协议等级，MQTT 3.1.1协议版本的协议等级是4。

**Connect Flag:** 字节8，连接标记，每一位都表示一个标记，Bit0是保留标记。从Bit1~Bit7，分别表示Clean Session, Will Flag等内容。这些标记确定了Payload是否包含对应的信息。比如Bit7和Bit6的值都为1，那么表示此次连接的Payload中包含User Name和Password。后边会分别介绍各个标记的作用。

**Keep Alive:** 字节9和字节10，客户端与服务器心跳间隔，高字节在前，低字节在后。单位是S，默认是60S。关于Keep Alive，需要注意的事项包括：

- 当客户端和服务器之间没有消息传输时，客户端会每隔60S(keep alive值)向MQTT Broker发送PINGREQ数据报文。服务器需要回复PINGRESP数据报文。
- 如果客户端在发送PINGREQ数据包一段时间后没有收到PINGRESP数据包，客户端会断开连接。
- 如果Keep Alive的值设置为大于0(假设60S)，在没有数据交互的情况下，服务器如果在超过1.5被Keep Alive时间(90S)后没有收到PINGREQ数据包，则服务器会断开与当前客户端的连接。
- Keep Alive可以设置为0，那么客户端不会发送PINGREQ数据包，服务器也不会因为没有收到PINGREQ而断开客户端连接。

我们来测试一下Keep Alive，观察PINGREQ和PINGRESP数据包。在命令行终端输入以下命令，会得到相应结果：

```bash
$ mosquitto_sub -t topic001 -k 5 -d
Client mosqsub|16532-SCNWCL012 sending CONNECT
Client mosqsub|16532-SCNWCL012 received CONNACK (0)
Client mosqsub|16532-SCNWCL012 sending SUBSCRIBE (Mid: 1, Topic: topic001, QoS: 0)
Client mosqsub|16532-SCNWCL012 received SUBACK
Subscribed (mid: 1): 0
Client mosqsub|16532-SCNWCL012 sending PINGREQ
Client mosqsub|16532-SCNWCL012 received PINGRESP
Client mosqsub|16532-SCNWCL012 sending PINGREQ
Client mosqsub|16532-SCNWCL012 received PINGRESP
Client mosqsub|16532-SCNWCL012 sending PINGREQ
Client mosqsub|16532-SCNWCL012 received PINGRESP
```

上面命令中，-t topic001表示订阅topic为topic001的主题，使用 -k 5表示设置keep alive时间间隔为5S，-d表示启用Debug模式。从输出结果可以看到每隔5S，客户端会发送PINGREQ，并收到从服务器返回的PINGRESP。

### Connect Flag

因为Connect Flag的内容比较多，所以单独用一小节来介绍。

##### Reserved

第8字节的Bit0。保留位，必须为0。

##### Clean Session

第8字节的Bit1，

