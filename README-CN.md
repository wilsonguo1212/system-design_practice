# system-design_practice
To share the experiences on system design during daily work
# 设计物流中心的智能仓储和物流配送系统

## 项目背景

本项目旨在通过 Microsoft Azure 云服务，为物流公司设计一套高可用架构。此架构包含两个核心功能：

- **智能仓储管理**：利用 IoT 技术实现货物出入库的自动化数据处理，对仓储环境进行温湿度监控。
- **最优配送路径选择**：通过 AI 和实时地图优化物流配送路径，提供实时订单状态跟踪。

## 步骤1，概述用例和约束

使用案例

我们将问题限定在仅处理以下用例

. 商品通过射频标签等物联网技术实现自动化登记入库

. 用户可以查看过去1个月的派送的快递产品信息

. 仓储中心实现智能监控

. 服务具有高可用性


超出范围：

. 一般的智能工厂管理系统

  . 本案例只涉及商品跟踪，和物流派送两项重点任务。


约束与假设

. 流量分布不均匀

. 项目属于多个类别

. 商品无法更改类别

. 每月有5亿笔货运单交易

. 每月有500亿次读取请求

. 读写比为100:1

计算使用量：

. 每个货运单的信息大小：

​	. Order_ID: 4 bytes

​	. RFID: 32 bytes

​	. Location: 50 bytes

​	. Status: 4 bytes

  ​. Timestamp: 8 bytes
  
. 总计：~100 bytes

. 每月50GB的新增货运信息

  ​. 每笔货运单100字节*每月5亿笔交易 = 50GB
  
. 平均每秒200笔交易

. 平均每秒20000次读取请求




## 步骤2，创建高级设计
![IoT smart warehouse](https://github.com/user-attachments/assets/ba5a6a37-f278-4630-8b79-7dffba57e583)




## 步骤3，设计核心组件

我们需要把RFID阅读器配置成一个IoT设备，并通过与Azure IoT Hub集成，实现数据的自动上传。具体工作流是RFID阅读器读取多标签的信息，然后使用WiFi，通过HTTP传输协议，将RFID读取的标签数据作为JSON数据发送到IoT Hub。

以下是用Python语言，编写的RFID阅读器

```Python
import time
import json
from azure.iot.device import IoTHubDeviceClient, Message

# IoT Hub 连接字符串（从设备身份注册中获取）
connection_string = "HostName=<Your IoT Hub Name>.azure-devices.net;DeviceId=<Your Device ID>;SharedAccessKey=<Your Device Key>"
 
# 创建 IoT Hub 客户端
client = IoTHubDeviceClient.create_from_connection_string(connection_string)

 
# 模拟从 RFID 阅读器获取数据
def read_rfid_data():
  # 这里模拟从 RFID 阅读器读取到的数据
  # 例如：RFID 标签的唯一标识符、时间戳、位置和状态
  rfid_data = {
    "order_id": 1001         # 模拟订单ID
	"rfid_id": "1234567890", # 模拟 RFID 标签 ID
    "timestamp": time.time(), # 当前时间戳
    "location": "Warehouse A", # 仓库位置
    "status": "scanned" # 状态信息
  }
  return rfid_data

# 定期向 Azure IoT Hub 发送数据
def send_rfid_data():
    while True:
      # 获取 RFID 数据
      data = read_rfid_data()

      # 将数据转换为 JSON 格式
	  message = Message(json.dumps(data))

	  # 发送数据到 IoT Hub
      client.send_message(message)
      print("Data sent to IoT Hub:", data)

      # 每 5 秒发送一次数据
      time.sleep(5)

# 启动数据发送
send_rfid_data()
```

类似的，仓储中心的温度和湿度传感器，与IoT Hub集成，实现数据的自动上传。代码编写示例如下：

```python
pip install azure-iot-device Adafruit-DHT     #安装依赖库
```

```python
import time
import json
import Adafruit_DHT # 用于从传感器读取温湿度
from azure.iot.device import IoTHubDeviceClient, Message

# IoT Hub 设备连接字符串（从 Azure IoT Hub 获取）
connection_string = "HostName=<Your IoT Hub Name>.azure-devices.net;DeviceId=<Your Device ID>;SharedAccessKey=<Your Device Key>"

# 创建 IoT Hub 客户端
client = IoTHubDeviceClient.create_from_connection_string(connection_string)

# 配置传感器类型，假设使用 DHT22
sensor = Adafruit_DHT.DHT22
pin = 4 # 假设 DHT22 连接到 Raspberry Pi 的 GPIO 4 引脚

# 读取传感器数据    
def read_temperature_humidity():
	humidity, temperature = Adafruit_DHT.read_retry(sensor, pin)
    if humidity is not None and temperature is not None:
	    return {"temperature": temperature, "humidity": humidity}
	else:
	    return {"temperature": None, "humidity": None}

# 定期向 Azure IoT Hub 发送数据
def send_sensor_data():
      while True:
		# 获取温湿度数据
	    data = read_temperature_humidity()

		# 如果读取成功，发送数据到 IoT Hub
	    if data["temperature"] is not None and data["humidity"] is not None:
	      data["timestamp"] = time.time() # 添加时间戳

 		  # 将数据转换为 JSON 格式
		  message = Message(json.dumps(data))

       	  # 发送数据到 IoT Hub
	      client.send_message(message)
		  print("Data sent to IoT Hub:", data)
        else:
	      print("Failed to read from sensor, trying again...")

        # 每 10 秒上传一次数据
        time.sleep(10)

# 启动数据发送
send_sensor_data()
```

