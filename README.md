# Prepare

Install Podman

```bash
$ sudo apk update && sudo apk add podman 
```

# Starting Mosquitto container

## Create work directory
```
$ mkdir iot && cd iot 
```

## Setting mosquitto config file and password_file

Create Config file
```
$ nano mosquitto.conf

listener 1883
allow_anonymous false
password_file /mosquitto/data/password_file
```

Create password_file

```bash
$ touch password_file
```


## Create mosquitto container

```bash
$ sudo podman run -itd \
 --restart=always \
 --name mosquitto \
 -p 1883:1883 \
 -v ./mosquitto.conf:/mosquitto/config/mosquitto.conf \
 -v ./password_file:/mosquitto/data/password_file \
 eclipse-mosquitto
```



# Create mosquttio user

Use `mosquitto_passwd` command to create user, please replace {USERNAME} and {PASSWORD} to you wanted.

```bash
# Add user
$ sudo podman exec -it mosquitto mosquitto_passwd -b /mosquitto/data/password_file {USERNAME} {PASSWORD}

# Reload password_file
$ sudo podman exec -it mosquitto kill -SIGHUP 1
```

# Testing

## Install MQTT Client tools

```bash
$ sudo apk add mosquitto_clients
```

## Subscribe test
```bash
$ mosquitto_sub -t '#' \
-h 'localhost' \
-p 1883 \
-u {USERNAME} \
-P {PASSWORD}
```

## Publish test
```bash
$ mosquitto_pub -t 'sometopic' \
-h 'localhost' \
-p 1883 \
-u {USERNAME} \
-P {PASSWORD} \
-m 'Hello World'
```


# MQTT Client using Shell script

## Prepare

Install MQTT client tools

```bash
$ sudo apk add mosquitto-clients
```

## Publisher

```bash
$ nano pub.sh
```

Modify variable `{USERNAME}` and `{PASSWORD}`.

```bash
#!/bin/bash

# Get system memory free
memUsage=$(cat /proc/meminfo | grep MemFree)

# publish data to broker
mosquitto_pub -t 'sometopic' \
-h 'localhost' \
-p 1883 \
-u "{USERNAME}" \
-P '{PASSWORD}' \
-m "$memUsage"
```

execute

```bash
$ bash pub.sh
```

## Subscriber

```bash
$ nano sub.sh
```

```bash
#!/bin/bash

while read message
do
  echo "$(date):$message"
done < <(mosquitto_sub -v \
-t 'sometopic' \
-h 'localhost' \
-p 1883 \
-u "{USERNAME}" \
-P '{PASSWORD}')

```

execute
```bash
$ bash sub.sh
```


# MQTT Client using python

## Prepare

Install python3 and upgrade it.

```bash
$ sudo apk update && sudo apk add --upgrade python3 py3-pip
```

Install Paho.mqtt library

```bash
$ pip3 install paho-mqtt
```



## Publisher

```
$ nano pub.py
```

Modify variable `serverAddress`, `serverPort`, `username` and `password`.
```python
import paho.mqtt.publish as publish

serverAddress='localhst'
serverPort=1883
password='bigred'
username='bigred'

publish.single(
  topic = "somtopic",
  payload = "payload",
  port = serverPort,
  hostname = serverAddress,
  auth = {'username' : username,'password' : password}
  )
```

```bash
$ python3 pub.py
```


## Subsriber

```
$ nano sub.py
```


Modify variable `serverAddress`, `serverPort`, `username` and `password`.

```python
import paho.mqtt.client as mqtt

serverAddress = ''
serverPort = 1883
keepAlive = 60
username = ''
password = ''

# 當成功連線到 MQTT Broker 時的動作 (Event)
def on_connect(client, userdata, flags, rc):
    print("成功連線到 Broker...")

    # 向 Broker 訂閱所有 MQTT Topic
    print("開始訂閱 Topic")
    client.subscribe("#")

# 當收到 MQTT Broker 訂閱的訊息時的動作 (Event)
def on_message(client, userdata, msg):
    print(msg.topic + " " + msg.payload.decode('utf-8'))


# 當與 MQTT Broker 連線中斷時的動作
# def on_disconnect(client, userdata, rc):
#     print("與 Broker 連線中斷")

client = mqtt.Client()

# 設定 MQTT Broker 連線資訊 (IP, Port, 連線時間)
client.connect(serverAddress, serverPort, keepAlive)

# 設定 MQTT Broker 帳號密碼
client.username_pw_set(username, password)

# 當成功連線到 MQTT Broker 時的動作 (Event)
client.on_connect = on_connect

# 當收到 MQTT Broker 訂閱的訊息時的動作 (Event)
client.on_message = on_message

# 當與 MQTT Broker 連線中斷時的動作
# client.on_disconnect = on_disconnect

# 使用 Lib 預設的 Loop 函數進行無窮迴圈，或是可以使用自己的 Loop 函數, EX. for , while true...
client.loop_forever()

```

```bash
$ python3 sub.sh
```
