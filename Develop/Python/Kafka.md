为满足需求，我将设计一个使用 `kafka-python` 的工具类。此类将实现一个消费者方法，定期从 Kafka 中拉取消息，以便 Celery 定时任务可以调用该方法并处理接收到的数据。

### 代码设计思路

1. **初始化 Kafka 消费者：**
   
   - 配置 Kafka 消费者（例如 `bootstrap_servers`、`group_id`、`auto_offset_reset` 等）。
   - 支持从特定主题消费数据。

2. **消费数据的主方法：**
   
   - 编写一个 `consume_data` 方法，从 Kafka 拉取数据。
   - 使用 `poll` 或 `consume` 方法进行消费，获取后可立即处理或返回。

3. **错误处理和日志：**
   
   - 使用 try-except 结构处理 Kafka 连接和消费异常，并记录异常信息。

4. **集成 Celery 定时任务：**
   
   - 确保该工具类能够在 Celery 定时任务中被调用，符合 Celery 的异步任务结构。

以下是实现代码：

```python
from kafka import KafkaConsumer
from typing import List, Optional
import json
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class KafkaConsumerTool:
    def __init__(self, topic: str, bootstrap_servers: List[str], group_id: str = 'default-group', auto_offset_reset: str = 'earliest'):
        """
        初始化 Kafka Consumer 工具类。

        :param topic: Kafka 主题名称。
        :param bootstrap_servers: Kafka 服务器地址列表。
        :param group_id: 消费组 ID，默认 'default-group'。
        :param auto_offset_reset: 从何处开始消费消息，'earliest' 表示从最早的消息开始。
        """
        self.topic = topic
        self.consumer = KafkaConsumer(
            topic,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            auto_offset_reset=auto_offset_reset,
            value_deserializer=lambda x: json.loads(x.decode('utf-8'))  # 解码为 JSON
        )

    def consume_data(self, max_records: Optional[int] = None):
        """
        消费 Kafka 数据，可用于 Celery 定时任务中。

        :param max_records: 最大消费记录数，为 None 表示消费所有可用数据。
        :return: 消费到的数据列表。
        """
        records = []
        try:
            # 消费记录
            for message in self.consumer:
                data = message.value  # 获取消息内容
                records.append(data)
                logger.info(f"消费到消息: {data}")

                # 若达到最大记录数，停止消费
                if max_records and len(records) >= max_records:
                    break

        except Exception as e:
            logger.error("消费 Kafka 数据时出错", exc_info=True)
        finally:
            self.consumer.close()

        return records

# Celery 定时任务中使用示例
from celery import Celery

# 初始化 Celery 应用
app = Celery('tasks', broker='redis://localhost:6379/0')  # 这里使用 Redis 作为消息代理

@app.task
def consume_kafka_data():
    # 使用 KafkaConsumerTool 来消费数据
    kafka_tool = KafkaConsumerTool(
        topic='your_topic_name',
        bootstrap_servers=['localhost:9092'],
        group_id='celery-consumer-group',
        auto_offset_reset='earliest'
    )

    data = kafka_tool.consume_data(max_records=10)  # 可选的最大记录数
    # 在这里处理 data 或进一步传递给其他 Celery 任务
    logger.info(f"获取的数据: {data}")
    return data
```

### 代码说明

- `KafkaConsumerTool` 工具类初始化时，接受主题名、服务器地址、消费组 ID 等配置，并自动反序列化 JSON 格式的消息内容。
- `consume_data` 方法用来消费 Kafka 中的消息，支持最大记录数限制；若未设定，默认消费所有拉取到的消息。
- `consume_kafka_data` 是 Celery 定时任务的一个示例，创建了一个 `KafkaConsumerTool` 实例并消费指定主题的消息。
