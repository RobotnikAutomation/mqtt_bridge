# mqtt_bridge
The `mqtt_bridge` package provides a functionality to bridge between ROS and MQTT bidirectionally.

> [!NOTE]
> `mqtt_bridge` is not actively maintained now. Feel free to check out [`mqtt_client`](https://github.com/ika-rwth-aachen/mqtt_client), a high-performance C++ ROS nodelet with recent development!


## 1 Principle
`mqtt_bridge` uses ROS message as its protocol. Messages from ROS are serialized by JSON (or messagepack) for MQTT, and messages from MQTT are deserialized for ROS topic. So MQTT messages should be ROS message compatible (`rosbridge_library.internal.message_conversion` is used for message conversion).

This limitation can be overcome by defining custom bridge class, though.

## 2 2Demo
### 2.1 Prerequisites
```sh
sudo apt install python3-pip
sudo apt install ros-noetic-rosbridge-library
sudo apt install mosquitto mosquitto-clients
```

### 2.2 Install Python Modules
```sh
pip3 install -r requirements.txt
```

### 2.3 Launch Node
``` bash
roslaunch mqtt_bridge demo.launch
```

Publish to `/ping`:
```sh
rostopic pub /ping std_msgs/Bool "data: true"
```

And see the response in `/pong`:
```sh
rostopic echo /pong
data: True
---
```

Publish "hello" to `/echo`:
```sh
rostopic pub /echo std_msgs/String "data: 'hello'"
```

And see the response in `/back`:
```sh
rostopic echo /back
data: hello
---
```

You can also see the MQTT messages using `mosquitto_sub`:
```sh
mosquitto_sub -t '#'
```

## 2.4 Usage
The configuration file is `config.yaml`:
``` yaml
mqtt:
  client:
    protocol: 4 # MQTTv311
  connection:
    host: localhost
    port: 1883
    keepalive: 60
  account:
    username: robocomp
    password: robocomp
  private_path: device/001
serializer: json:dumps
deserializer: json:loads

bridge:
  # ping pong
  - factory: mqtt_bridge.bridge:RosToMqttBridge
    msg_type: std_msgs.msg:Bool
    topic_from: /ping
    topic_to: ping
  - factory: mqtt_bridge.bridge:MqttToRosBridge
    msg_type: std_msgs.msg:Bool
    topic_from: ping
    topic_to: /pong
```
> [!TIP]
> You can use any message type, like `sensor_msgs.msg:Imu`, for example.

The launch file is basically:
``` xml
<launch>
  <arg name="use_tls" default="false" />
  <node name="mqtt_bridge" pkg="mqtt_bridge" type="mqtt_bridge_node.py" output="screen">
    <rosparam command="load" file="$(find mqtt_bridge)/config/demo_params.yaml" />
    <rosparam if="$(arg use_tls)" command="load" ns="mqtt" file="$(find mqtt_bridge)/config/tls_params.yaml" />
  </node>
</launch>
```

## 3 Configuration
### 3.1 MQTT
Parameters under `mqtt` section are used for creating paho's `mqtt.Client` and its configuration.

#### 3.2 Subsections
* `client`: used for `mqtt.Client` constructor
* `tls`: used for tls configuration
* `account`: used for username and password configuration
* `message`: used for MQTT message configuration
* `userdata`: used for MQTT userdata configuration
* `will`: used for MQTT's will configuration

See `mqtt_bridge.mqtt_client` for detail.

### 3.3 MQTT Private Path
If `mqtt/private_path` parameter is set, leading `~/` in MQTT topic path will be replaced by this value. For example, if `mqtt/pivate_path` is set as `"device/001"`, MQTT path `"~/value"` will be converted to `"device/001/value"`.

### 3.4 Serializer and Deserializer
`mqtt_bridge` uses `msgpack` as a serializer by default. But you can also configure other serializers. For example, if you want to use JSON for serialization, add following configuration.

``` yaml
serializer: json:dumps
deserializer: json:loads
```

### 3.5 Bridges
You can list ROS <--> MQTT transfer specifications in the following format:
``` yaml
bridge:
  # ping pong
  - factory: mqtt_bridge.bridge:RosToMqttBridge
    msg_type: std_msgs.msg:Bool
    topic_from: /ping
    topic_to: ping
  - factory: mqtt_bridge.bridge:MqttToRosBridge
    msg_type: std_msgs.msg:Bool
    topic_from: ping
    topic_to: /pong
```

* `factory`: bridge class for transfering message from ROS to MQTT, and viceversa
* `msg_type`: ROS message type transfering through the bridge
* `topic_from`: topic incoming from (ROS or MQTT)
* `topic_to`: topic outgoing to (ROS or MQTT)

Also, you can create custom bridge class by inheriting `mqtt_brige.bridge.Bridge`.

## 4 License
This software is released under the MIT License, see LICENSE.txt.
