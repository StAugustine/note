以下是一个完整的最终功能实现方案，包括Kafka工具类、Celery定时任务、MySQL存储逻辑等模块化实现。该方案将Kafka消息消费逻辑封装成工具类，方便在Celery定时任务中调用，同时确保任务在完成一批数据的处理后正常退出，以便被定时调度再次调用。

### 最终方案实现

#### 1. 配置文件 `config.py`

用于管理应用的配置参数，包括Kafka、MySQL和Celery相关配置。

```python
# config.py
class Config:
    # MySQL 配置
    MYSQL_USER = 'your_user'
    MYSQL_PASSWORD = 'your_password'
    MYSQL_DB = 'your_database'
    MYSQL_HOST = 'localhost'

    # Kafka 配置
    KAFKA_SERVER = 'localhost:9092'
    KAFKA_TOPIC = 'your_topic_name'

    # Celery 配置
    CELERY_BROKER_URL = 'redis://localhost:6379/0'
    CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'
```

#### 2. Flask 和 Celery 应用初始化 (`app.py`)

初始化Flask应用及Celery任务队列。

```python
# app.py
from flask import Flask
from celery import Celery
from config import Config
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

app = Flask(__name__)
app.config.from_object(Config)

# 初始化Celery
def make_celery(app):
    celery = Celery(
        app.import_name,
        broker=app.config['CELERY_BROKER_URL'],
        backend=app.config['CELERY_RESULT_BACKEND']
    )
    celery.conf.update(app.config)
    return celery

celery = make_celery(app)

# 初始化数据库
DATABASE_URI = f"mysql+mysqlconnector://{Config.MYSQL_USER}:{Config.MYSQL_PASSWORD}@{Config.MYSQL_HOST}/{Config.MYSQL_DB}"
engine = create_engine(DATABASE_URI)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

#### 3. 数据库模型定义 (`models.py`)

定义Kafka消息数据的存储模型，例如`UserData`模型。

```python
# models.py
from sqlalchemy import Column, Integer, String
from app import Base

class UserData(Base):
    __tablename__ = 'user_data'
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(50))
    email = Column(String(100))
    age = Column(Integer)

Base.metadata.create_all(bind=engine)
```

#### 4. Kafka 消费者工具类 (`kafka_utils.py`)

定义一个通用的Kafka消费者工具类，以便在Celery任务中复用。

```python
# kafka_utils.py
from kafka import KafkaConsumer
import json
from config import Config

class KafkaConsumerTool:
    def __init__(self, topic, group_id, bootstrap_servers=None, auto_offset_reset='earliest', consumer_timeout_ms=1000):
        """
        初始化Kafka消费者工具类
        :param topic: 消费的Kafka主题
        :param group_id: 消费者组ID
        :param bootstrap_servers: Kafka服务器地址列表
        :param auto_offset_reset: 偏移重置策略，默认是 'earliest'
        :param consumer_timeout_ms: 消费者超时时间（毫秒），超时后自动退出循环
        """
        self.topic = topic
        self.group_id = group_id
        self.bootstrap_servers = bootstrap_servers or [Config.KAFKA_SERVER]
        self.auto_offset_reset = auto_offset_reset
        self.consumer_timeout_ms = consumer_timeout_ms
        self.consumer = self._create_consumer()

    def _create_consumer(self):
        """
        内部方法：创建Kafka消费者
        """
        return KafkaConsumer(
            self.topic,
            bootstrap_servers=self.bootstrap_servers,
            auto_offset_reset=self.auto_offset_reset,
            enable_auto_commit=True,
            group_id=self.group_id,
            consumer_timeout_ms=self.consumer_timeout_ms  # 设置消费超时
        )

    def consume_messages(self, max_messages=None):
        """
        消费Kafka中的消息并解析为Python字典
        :param max_messages: 最大消费消息数量，默认None表示不限制数量
        :return: 生成器，每次返回一条消息数据的字典
        """
        count = 0
        for message in self.consumer:
            try:
                yield json.loads(message.value.decode('utf-8'))
                count += 1
                if max_messages and count >= max_messages:
                    break  # 达到最大消息数量限制后退出
            except json.JSONDecodeError:
                print(f"Failed to decode message: {message.value}")
```

#### 5. Celery任务逻辑 (`tasks.py`)

编写Celery任务，使用`KafkaConsumerTool`消费消息并将其存储到MySQL数据库。

```python
# tasks.py
from celery import Celery
from sqlalchemy.orm import Session
from app import celery, SessionLocal
from models import UserData
from kafka_utils import KafkaConsumerTool
from config import Config

@celery.task
def consume_kafka_messages():
    """
    Celery任务：从Kafka消费消息并存储到MySQL
    """
    # 初始化Kafka消费者工具类，设置最大消费数量为100
    kafka_tool = KafkaConsumerTool(
        topic=Config.KAFKA_TOPIC,
        group_id='my-group',
        bootstrap_servers=[Config.KAFKA_SERVER],
        consumer_timeout_ms=1000  # 超时1秒后自动退出
    )
    
    # 创建数据库会话
    db: Session = SessionLocal()
    
    try:
        # 处理Kafka消息，设置最大消费消息数量为100
        messages = list(kafka_tool.consume_messages(max_messages=100))
        
        # 批量插入数据库
        for message_data in messages:
            try:
                user_data = UserData(
                    name=message_data.get('name'),
                    email=message_data.get('email'),
                    age=message_data.get('age')
                )
                db.add(user_data)
            except Exception as e:
                print(f"Failed to process message: {message_data}, error: {e}")
        
        db.commit()
    finally:
        db.close()
```

#### 6. Celery定时任务配置 (`app.py`或配置文件中直接配置)

通过Celery的`beat`来设置定时任务，每隔一段时间调用`consume_kafka_messages`任务。

```python
# app.py (添加到Celery的配置中)
from celery.schedules import crontab

celery.conf.beat_schedule = {
    'consume-kafka-every-minute': {
        'task': 'tasks.consume_kafka_messages',
        'schedule': crontab(minute='*/1'),  # 每分钟执行一次
    },
}
celery.conf.timezone = 'UTC'
```

### 启动步骤

1. **启动Kafka服务器和Redis服务器**（Celery消息代理）。
2. **启动Celery Worker** 及 **Celery Beat**：
   ```bash
   celery -A app.celery worker --loglevel=info
   celery -A app.celery beat --loglevel=info
   ```

### 代码逻辑说明

- **`KafkaConsumerTool`**：封装了Kafka消费逻辑，支持主题、消费组、超时和最大消息数量等配置，具有较好的通用性。
- **Celery任务 `consume_kafka_messages`**：使用Kafka工具类消费消息，并将消息批量存储到MySQL，任务每次消费最多100条消息且支持超时退出，以避免阻塞。
- **定时任务**：通过Celery Beat 定时调用 `consume_kafka_messages`，例如每分钟一次。

### 总结

这个方案实现了Kafka消息消费、定时任务调度、MySQL持久化等功能，且具备良好的代码复用性和模块化结构。