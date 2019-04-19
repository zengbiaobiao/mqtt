# MQTT Message Design Spec

### Overview

MQTT协议是一套轻量级的专注于IoT(Internet of Thing)，M2M(machine to machine)以及移动设备的消息传输协议，MQTT有效的解决了低带宽，高延迟，以及网络不可靠等条件下的数据传输问题，并在业界得到了广泛应用。

EMQTT是一个使用Erlang/OTP写的，实现了MQTT协议的服务端框架。EMQTT拥有丰富的特性和很好的性能，配置简单，提供良好的dashboard进行消息监管，强大的插件功能，以及良好的集群支持，IoEE中产品大部分设备端的数据都将基于EMQTT传输。







### Connection

在创建MQTT连接到EMQTT server时，需要设置Connection Identifier，TLS证书，Will Topic，Clean Session等信息，下边分别介绍。

**Connection Identifier**

虽然connection identifier不是必须的，但是考虑到在发布QoS>0的消息时，必须要有connection id，并且设置clean session为false，所以我们规定需要每一个连接设置一个id。设置规则如下：

1. connection identifier必须唯一，如果有两个客户连接使用了相同的id，那个两个客户端连接或轮番连接并踢掉对方。
2. 必须一致，id不能在重连或应用重启之后发生变化。如果有QoS>0的消息发送到离线设备时，服务器会根据connection id保留消息，并在设备上线时再次发送。如果connection id发生变化，那么设备就收不到离线消息了。
3. id命名规则没有特别要求，但是推荐使用有意义id，如<连接类型>_<设备id>。比如如果是设备状态连接，那么使用equipment_status_10000127，如果是传感器连接，那么使用equipment_sensor_10000127。

**TLS证书**

在IoEE EMQTT的P环境中，会关闭普通连接1883端口，只启用ssl连接8883端口。证书会发给开发人员，需要在连接中配置TLS证书才能连接到服务器。

**Will Topic**

对于需要监听连接在线状态的设备，当设备连接成功时，需要发送设备在线消息到指定的topic，同时需要设置will topic和will message，当设备离线时，EMQTT会发送will message到will topic。will message的OoS应该设置为1，确保消息不丢失。

比如对于设备监控连接，需要设置will topic为equipment/10000127/status，will message为如下内容：

```
// 设备离线状态数据
{  
   "OnlineStatus":"False"
}
```

并且当连接成功时，发送以下消息到equipment/10000127/status。

```
// 设备在线状态数据
{  
   "OnlineStatus":"True"
}
```

这样服务器就能收到设备在线和离线消息。

**Clean Session**

需要设置连接的Clean Session为false，这样才能支持QoS为1或2的消息发布和订阅。







### Topic & Payload

EMQTT是一个基于pub/sub模式的消息中间件，所有的消息都与topic关联，所以消息的定义需要包含topic的定义。topic定义应遵循如下原则：

1. 简洁明了，并自带元数据信息。这样就能根据topic就能看出它的用途，以及关联哪些设备。
2. 只使用英文字母和数据，不要使用特殊字符，空格或汉字。虽然topic name支持空格，特殊字符，但是这会给MQTT server处理增加复杂度，可能还有造成兼容性问题。
3. 不要以/开头，比如/home/device/light，等于在最前面有一个空字符串层级，这完全没有必要而且增加了broker之类的处理, home/device/light 才是合理的。。
4. 将命令放在最后，如home/bedroom/bedlight/rgb/set，这个是为卧室的灯设置颜色。

在MQTT协议中，message通过payload进行传输，message的定义根据不同的应用进行定义，message应遵循如下原则：

1. 因为topic本身包含一些元数据信息，所以在有些场景中，message可以只包含数据内容，而不用附加一些元数据信息(如messageType，equipmentNo等)，如果topic无法(或者不适合)包含所有的元数据信息，则需要根据业务需求在message中添加元数据。
2. 除非特殊说明，否则所有的message应当以Json格式进行传输。
3. 从E&E  Gateway返回的message，如果数据来源于E&E Adapter，那么message的字段名称/字段值应与Adapter中的字段名称/字段值保持一致。









### Equipment Monitoring

对于设备监控，应包含两个topic，一个用于从服务器发送监控请求给设备，另一个用于接收设备实时状态数据。

**监控请求topic**

当需要监控设备时，需要发送监控请求消息到监控请求topic，E&E Broker中Gateway应当订阅监控请求topic主题，当收到StartMonitoring request时开始发送监控数据到监控状态topic，当收到StopMonitoring request时停止发送数据到监控状态topic。监控请求topic格式如下：

[equipment type]/[equipment no.]/status/request 

equipment type: 设备类型，包括elevator，escalator，sidewalk

equipment no.，设备编号，如10000127

合理的监控请求topic如下：

elevator/54000999/status/request

escalator/10000127/status/request

**监控请求message**

```
{  
   "RequestType":"StartMonitoring"
}
```

RequestType: StartMonitoring or StopMonitoring，表示开始或停止设备监控



**监控状态topic**

E&E Broker Gateway将从Adapter得到的数据发给监控状态topic，其格式定义如下：

[equipment type]/[equipment no.]/status

各Level名称解释请参考监控请求

**监控状态message**

E&E Broker Gateway发送到EMQTT server的状态数据，可以一次返回一个字段或多个字段。参考的消息格式如下：

```
// 扶梯状态数据
{  
   "ControlMode":"Remote",
   "Direction":"Down",
   "TechnicianOnSite":"True"
}
```



```
// 电梯状态数据，字段名称仅供参考，应保持与Adapter中定义的字段名称一致
{  
   "Direction":"Up",
   "CurrentFloor":12,
   "UpCall":[  
      9,
      10,
      12
   ]
}
```







### Equipment Error and Warning

**Error/Warning topic**

当设备产生故障或警告时，设备提交故障或警告信息到EMQTT server。SIP(Schindler IoEE Platform)会订阅这些错误或警告消息，进行相应的处理。

错误或警告信息由设备主动提交到EMQTT server，所以只需要定义单向的topic，其格式定义如下：

[equipment type]/[equipment no.]/[error type]

error type: 为error或warning

合理的topic如下：

equipment/10000128/error

escalator/10000129/warning

**Error/Warning message**

错误或警告信息以数组的格式返回，可以一次返回一条或多条记录，如果只有一条记录，也应该将错误或警告记录放在数组中。参考的Error或Warning消息格式如下：

```
// elevator error

[  
   {  
      "ErrorCode":"0x23",      
      "ErrorDesc":"Breakdawn",
      "ErrorTime":"1530583048058" //考虑到设备掉线时消息发送可能有延迟，需要添加ErrorTime
   },
   {  
      "ErrorCode":"0x24",
      "ErrorDesc":"Can't close the door",
      "ErrorTime":"1530583048058"
   }
]
```

```
[  
   {  
      "ErrorCode":"0x08",
      "ErrorDesc":"Overheat",
      "ErrorTime":"1530583048058"
   }
]
```







### Sensor Message

Sensor的topic定义如下：

[equipment type]/[equipment no.]/sensor/[sensor name]/data/request

[equipment type]/[equipment no.]/sensor/[sensor name]/data

sensor name: 每一台设备上可能装有多个传感器，使用sensor name进行区分，如humidity，weight等。

大部分传感器可能是单向数据传输，由传感器发送数据到SIP2.0，所以只要使用data topic即可。

当需要双向数据传输时，SIP2.0先发送数据请求到request topic，然后传感器发送数据到SIP2.0。数据格式参考Equipment Monitoring中的监控请求数据格式。



