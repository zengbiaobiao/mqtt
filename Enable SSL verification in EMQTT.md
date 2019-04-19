# Enable SSL Verification in EMQTT

### Enable SSL Verification in EMQTT

SSL is a security protocol for data transporting. In your production environment, you should enable SSL verification for your EMQTT server. We can make it by following steps:

1\. Enter emqttd cert foler /etc/emqttd/certs, create server root certificate

using openssl to generate the server root certificate

```shell
# create root certificate key
openssl genrsa -out MyRootCA.key 2048

# create root certificate
openssl req -x509 -new -nodes -key MyRootCA.key -sha256 -days 3650 -out MyRootCA.pem
```

​

2\. Create your server certificate

Using the root certificate to sing a server certificate

```shell
# generate server certificate key
openssl genrsa -out MyEMQ1.key 2048

# create a certificate signing request
openssl req -new -key ./MyClient1.key -out MyClient1.csr

# using the root certificate to issue a server certificate
openssl x509 -req -in ./MyEMQ1.csr -CA MyRootCA.pem -CAkey MyRootCA.key 
```

​

3\. Configure you server certificate

Edit the /etc/emqtt/emq.conf file, set below properties in the config file.

```properties
#private key for emq cert:
listener.ssl.external.keyfile = etc/certs/MyEMQ1.key
#emq cert:
listener.ssl.external.certfile = etc/certs/MyEMQ1.pem
#CA cert:
listener.ssl.external.cacertfile = etc/certs/MyRootCA.pem
```

4\. Restart emqttd service

`systemctl restart emqttd`

​

### Test SSL Verification

1\. Test SSL  using mosquittto

execute below command to publish a message using mosquitto

```shell
[root@localhost certs]  mosquitto_pub -p 8883 -t test -d --cafile MqttRootCA.pem  --insecure -m "test"
Client mosqpub|27861-localhost sending CONNECT
Client mosqpub|27861-localhost received CONNACK
Client mosqpub|27861-localhost sending PUBLISH (d0, q0, r0, m1, 'test', ... (4 bytes))
Client mosqpub|27861-localhost sending DISCONNECT
```

​

2\. Using Python3.6 to test SSL connection

You can use the root certificate to connect to the MQTT server. below is the Python test code with Paho MQTT client

```python
    __IoEE_MQTT_SERVER_IP = "10.69.94.176"
    __IoEE_MQTT_SERVER_PORT = 8883
    __IoEE_MQTT_SERVER_CERTIFICATE_PATH = "cert/cacert.pem"
    __IoEE_MQTT_CLIENT_KEEPALIVE_TIME = 10
    __STOP_MONITORING_COMMAND = "stop"
    __EQUIPMENT_STATUS_GATEWAY_VERSION_TOPIC = "equipment/status/gateway/version"
```

```python
def create_client(self):
       self.client = mqtt.Client(client_id=self.client_identifier, clean_session=0)
       self.client.tls_set(self.__IoEE_MQTT_SERVER_CERTIFICATE_PATH, None, None, cert_reqs=ssl.CERT_NONE,
                           tls_version=ssl.PROTOCOL_TLSv1, ciphers=None)
       self.client.tls_insecure_set(True)
       mqtt_client_util.initialize_client(self.client)
       online_status = {
           "OnlineStatus": "False"
       }
       self.client.will_set(self.equipment_status_topic, json.dumps(online_status), 1)
       self.client.connect(self.__IoEE_MQTT_SERVER_IP, self.__IoEE_MQTT_SERVER_PORT,
                           self.__IoEE_MQTT_CLIENT_KEEPALIVE_TIME)
```

3\. Using Java program to test SSL connection

You should convert the server certificate cert.pem file into a JKS file before you connecting to MQTT server. execute below command to convert the pem file.

```shell
# make sure to conver the server certificate, not the root certificate
keytool -keystore cert.jks -import -file cert.pem -alias cert
```

Below code section shows how to connect to the MQTT server using a JKS certificate.

```java
private static MqttClient createMqttClient() {
        MemoryPersistence persistence = new MemoryPersistence();
        MqttClient mqttClient = null;
        try {
            mqttClient = new MqttClient(IOEE_MQTT_SERVER_IP, IOEE_MQTT_CLIENT_IDENTIFIER, persistence);
            MqttConnectOptions connOpts = new MqttConnectOptions();
            connOpts.setCleanSession(false);
            connOpts.setAutomaticReconnect(true);
            connOpts.setConnectionTimeout(IOEE_MQTT_CLIENT_KEEPALIVE_TIME);
            connOpts.setKeepAliveInterval(IOEE_MQTT_CLIENT_KEEPALIVE_TIME);
```

```java
try {
    System.setProperty("javax.net.ssl.debug", "true");
    KeyStore keyStore = KeyStore.getInstance("JKS");
    InputStream inputStream = EquipmentMonitorApplication.class.getClassLoader().getResourceAsStream("MqttCert.jks");
    keyStore.load(inputStream, "123465".toCharArray());

    TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance("X509");
    trustManagerFactory.init(keyStore);
    TrustManager[] trustManagers = trustManagerFactory.getTrustManagers();
    SSLContext sslContext = SSLContext.getInstance("TLS");
    sslContext.init(null, trustManagers, null);
    SSLSocketFactory sslSockFactory = sslContext.getSocketFactory();
    connOpts.setSocketFactory(sslSockFactory);

} catch (Exception e) {
    e.printStackTrace();
}
```



   ```java
   mqttClient.connect(connOpts);
   
mqttClient.setCallback(new MqttCallbackExtended() {
    @Override
    public void connectComplete(boolean reconnect, String serverURI) {
        System.out.println("Connection:connect completed!");
    }
   ```



```java
@Override
public void connectionLost(Throwable throwable) {
    System.out.println("Connection:connection lost");
}

@Override
public void messageArrived(String s, MqttMessage mqttMessage) throws Exception {
    System.out.println(s + ": " + new String(mqttMessage.getPayload()));
}

@Override
public void deliveryComplete(IMqttDeliveryToken iMqttDeliveryToken) {
    System.out.println("Delivery:deliver complete");
}
});
} catch (Exception e) {
    System.out.println(e.toString());
}

return mqttClient;
```


**Note**

The PKCS12 certificate doesn't work very well in Java, refer to this article http://blog.palominolabs.com/2011/10/18/java-2-way-tlsssl-client-certificates-and-pkcs12-vs-jks-keystores/index.html for more details.



We only cover the server side authentication in this article, if you need enable client authentication, please refer below article.   

https://medium.com/@emqtt/securing-emq-connections-with-ssl-432672ab9f06 