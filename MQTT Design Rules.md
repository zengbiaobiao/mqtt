# MQTT Design Rules

### Overview

This document defines the rules you should obey when you design your application related to MQTT. 

### Connection

When you connect to a MQTT server, you should use TLS cert, set the Connection Identifier, Clean Session and so on.

**Connection Identifier**

1\. The connection identifier must be unique, if there are two clients connect to MQTT server with the same identifier, it will fall into a connection and disconnection loop.

2\. Don't use random Connection Identifier. The identifier should be meaningful and won't change after reconnection.

**TLS Authentication**

In product environment, the port 1883 will be disabled, so you should use a cert to connect to the MQTT broker.

**Will Topic**

1\. The escalator gateway client should send a online message to SMS when it connects to MQTT server successfully. It should also set a Will message so that the SMS can receive an offline message when the gateway goes offline.

```JSON
// offline message
{  
   "OnlineStatus":"False"
}
```

```JSON
// online message
{  
   "OnlineStatus":"True"
}
```

2\. The QoS of Will message should be set to 1 so that it can be delivered to SMS.



**Clean Session**

The Clean Session should be set to false so that it can support QoS 1 and QoS 2.



### Topic & Payload

Topic design should follow below conventions:

1\. Topic should be meaningful and short, it's better to include some meta data, such as equipment No. e.g, "escalator/54000999/status" means this topic is related the escalator status message, its equipment No is 54000999.

2\. Only use letters or/and digits as the topic name. Although MQTT server support special character and space, it increase the complexity for the server to handle the delivery. 

3\. Don't start with "/", it adds an additional level for the topic and increase the complexity for MQTT server. e.g. "home/device/light/1" is recommended, "/home/device/light/1" is not recommended.

4\. Append action to the last of the topic. e.g, "home/bedroom/light/1/rgb/set" means to set the rgb color for the first light in bedroom.

The message rules includes:

1\. Considering the topic contains some meta data such the equipment No, the message type, the message can only contain data. If the topic name can't include all necessary meta data, then the message content should includes them.

2\. All messages are transferred by JSON format except a specific explanation.

3\. Generally, there are duplex topics for one type of message, e.g, "escalator/54000999/status" and "escalator/54000999/status/request". The first one is which the gateway MQTT client will publish to, the second is what should be subscribed.



