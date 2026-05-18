# Протоколы обмена сообщениями: WebSocket, MQTT, AMQP (TICODE - https://ticode.online/)

## WebSocket

### Что такое WebSocket

WebSocket — протокол **полнодуплексной** (двусторонней) связи поверх одного TCP соединения. Разработан для реалтайм-коммуникации в браузере.

**Проблема HTTP**: только pull — клиент запрашивает, сервер отвечает. Для чатов/уведомлений это плохо.

**WebSocket решение**: после установки соединения и сервер, и клиент могут отправлять данные **в любой момент**.

```
HTTP polling (плохо):
Клиент ─► "Есть новые сообщения?" ─► Сервер
Клиент ◄─ "Нет" ◄─────────────────── Сервер
(через 1 сек)
Клиент ─► "Есть новые сообщения?" ─► Сервер
Клиент ◄─ "Нет" ◄─────────────────── Сервер

WebSocket (хорошо):
Клиент ◄══════ соединение ══════════ Сервер
(через 5 сек)
Клиент ◄─ "Новое сообщение: Привет!" ─ Сервер
(через 10 сек)
Клиент ─► "Ответ: Привет!" ──────────► Сервер
```

---

### WebSocket Handshake

WebSocket начинается как HTTP запрос и "апгрейдится":

```
Клиент → Сервер (HTTP Upgrade):
GET /ws HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

Сервер → Клиент:
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

После этого TCP соединение используется для WebSocket фреймов.

---

### WebSocket фрейм

```
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┴─┴─┴─┼─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┤
│F│R│R│R│ opcode │M│     Payload len       │  Extended len ...   │
│I│S│S│S│       │A│                       │                     │
│N│V│V│V│       │S│                       │                     │
│ │1│2│3│       │K│                       │                     │
└─┴─┴─┴─┴───────┴─┴───────────────────────┴─────────────────────┘
```

Opcodes:
- `0x0` — Continuation frame
- `0x1` — Text frame (UTF-8)
- `0x2` — Binary frame
- `0x8` — Close
- `0x9` — Ping
- `0xA` — Pong

---

### WebSocket сервер на Python

```python
import asyncio
import websockets
import json

connected_clients: set = set()

async def handler(websocket):
    connected_clients.add(websocket)
    print(f"Подключился: {websocket.remote_address}")
    
    try:
        async for message in websocket:
            data = json.loads(message)
            print(f"Получено: {data}")
            
            # Broadcast всем клиентам
            response = json.dumps({
                "type": "message",
                "from": str(websocket.remote_address),
                "text": data.get("text", ""),
            })
            
            await asyncio.gather(
                *[client.send(response) for client in connected_clients]
            )
    except websockets.exceptions.ConnectionClosed:
        pass
    finally:
        connected_clients.discard(websocket)
        print(f"Отключился: {websocket.remote_address}")

async def main():
    async with websockets.serve(handler, "127.0.0.1", 8765):
        print("WebSocket сервер на ws://127.0.0.1:8765")
        await asyncio.Future()  # работаем вечно

# asyncio.run(main())
```

---

### WebSocket клиент на Python

```python
import asyncio
import websockets
import json

async def client():
    uri = "ws://127.0.0.1:8765"
    
    async with websockets.connect(uri) as ws:
        # Отправить сообщение
        await ws.send(json.dumps({"text": "Привет всем!"}))
        
        # Получить ответ
        response = await ws.recv()
        print(f"Получено: {response}")
        
        # Пинг для поддержания соединения
        await ws.ping()

# asyncio.run(client())
```

---

### WebSocket vs HTTP

| | HTTP | WebSocket |
|--|------|-----------|
| Инициатор | Только клиент | Обе стороны |
| Соединение | Новое на каждый запрос (HTTP/1.1) | Одно, живёт долго |
| Overhead | Заголовки (~200-800 байт) каждый раз | Только при установке |
| Применение | REST API, страницы | Чат, игры, биржи, уведомления |

---

## MQTT — Message Queuing Telemetry Transport

### Что такое MQTT

MQTT — лёгкий протокол **publish/subscribe** для IoT устройств. Разработан для устройств с ограниченными ресурсами (датчики, микроконтроллеры) и ненадёжными сетями.

Ключевые принципы:
- **Publish/Subscribe** — отправитель не знает получателей, посредник-брокер доставляет
- **Topics** — иерархические пути как каналы (`sensors/room1/temperature`)
- **QoS** — уровни гарантии доставки
- **Retain** — брокер хранит последнее сообщение по топику

---

### Архитектура MQTT

```
Устройство 1 (Publisher)        MQTT Broker         Устройство 2 (Subscriber)
  │                               │                        │
  │─ CONNECT ───────────────────► │                        │
  │◄─ CONNACK ──────────────────  │                        │
  │                               │◄── CONNECT ────────────│
  │                               │──► CONNACK ────────────│
  │                               │◄── SUBSCRIBE ──────────│
  │                               │    sensors/+/temp      │
  │                               │──► SUBACK ─────────────│
  │                               │                        │
  │─ PUBLISH ───────────────────► │                        │
  │  topic: sensors/room1/temp    │──► PUBLISH ────────────│
  │  payload: 23.5                │    (доставка)          │
```

---

### MQTT QoS уровни

| QoS | Название | Гарантия | Механизм |
|-----|---------|---------|---------|
| 0 | At most once | Возможна потеря | Fire and forget |
| 1 | At least once | Дублирование возможно | PUBACK подтверждение |
| 2 | Exactly once | Ровно один раз | 4-way handshake |

```python
import paho.mqtt.client as mqtt  # pip install paho-mqtt

def on_connect(client, userdata, flags, rc, properties=None):
    print(f"Подключён, код: {rc}")
    # Подписываемся сразу после подключения
    client.subscribe("sensors/#", qos=1)

def on_message(client, userdata, msg):
    print(f"Топик: {msg.topic}")
    print(f"Данные: {msg.payload.decode()}")
    print(f"QoS: {msg.qos}")

# Подписчик
subscriber = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
subscriber.on_connect = on_connect
subscriber.on_message = on_message

subscriber.connect("mqtt.eclipseprojects.io", 1883, keepalive=60)
subscriber.loop_start()

# Публикатор
publisher = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)
publisher.connect("mqtt.eclipseprojects.io", 1883)

publisher.publish(
    topic="sensors/room1/temperature",
    payload="23.5",
    qos=1,
    retain=True,  # Брокер хранит последнее значение
)

publisher.disconnect()
```

---

### MQTT топики и wildcards

```
sensors/room1/temperature    ← конкретный топик
sensors/+/temperature        ← + = один уровень (room1, room2, ...)
sensors/#                    ← # = все уровни ниже
```

```python
# Wildcard подписки
client.subscribe("sensors/+/temperature")  # Все комнаты, только температура
client.subscribe("sensors/#")              # Все данные со всех сенсоров
client.subscribe([
    ("sensors/room1/+", 1),   # Все данные из room1
    ("alerts/#", 2),           # Все алерты с QoS 2
])
```

---

### Last Will and Testament (LWT)

Сообщение, которое брокер отправит если устройство неожиданно отключится:

```python
client = mqtt.Client(mqtt.CallbackAPIVersion.VERSION2)

# Настраиваем LWT до подключения
client.will_set(
    topic="devices/sensor1/status",
    payload="offline",
    qos=1,
    retain=True,
)

client.connect("broker.example.com", 1883)

# Когда подключились — публикуем "online"
client.publish("devices/sensor1/status", "online", retain=True)
```

---

## AMQP — Advanced Message Queuing Protocol

### Что такое AMQP

AMQP — протокол для **enterprise messaging** (RabbitMQ, ActiveMQ). Более мощный чем MQTT, с понятиями:
- **Exchange** — получает сообщения от публикатора, направляет в очереди по правилам
- **Queue** — буфер хранения сообщений
- **Binding** — правило связи exchange → queue
- **Routing key** — метка сообщения для маршрутизации

---

### Типы Exchange

```
Direct Exchange:
Publisher → [exchange: direct] → routing_key=="order" → [Queue: orders]
                              → routing_key=="email" → [Queue: emails]

Fanout Exchange (broadcast):
Publisher → [exchange: fanout] → [Queue: service1]
                              → [Queue: service2]
                              → [Queue: service3]

Topic Exchange (wildcard):
Publisher → [exchange: topic]  → "order.created" → [Queue: all_orders]
                              → "order.paid"    → [Queue: payments]
                              → "user.*"        → [Queue: user_events]

Headers Exchange:
Publisher → [exchange: headers] → {"type": "report"} → [Queue: reports]
```

---

### RabbitMQ с Python (pika)

```python
import pika  # pip install pika
import json

# Подключение
connection = pika.BlockingConnection(
    pika.ConnectionParameters("localhost")
)
channel = connection.channel()

# Объявить exchange и очередь
channel.exchange_declare(exchange="orders", exchange_type="direct", durable=True)
channel.queue_declare(queue="order_processing", durable=True)
channel.queue_bind(exchange="orders", queue="order_processing", routing_key="new_order")

# Публикация
def publish_order(order: dict):
    channel.basic_publish(
        exchange="orders",
        routing_key="new_order",
        body=json.dumps(order),
        properties=pika.BasicProperties(
            delivery_mode=pika.DeliveryMode.Persistent,  # Сохранять на диск
            content_type="application/json",
        ),
    )
    print(f"Опубликован заказ: {order['id']}")

# Потребитель
def on_message(ch, method, properties, body):
    order = json.loads(body)
    print(f"Обрабатываю заказ {order['id']}")
    
    # Обработка...
    
    ch.basic_ack(delivery_tag=method.delivery_tag)  # Подтверждаем получение

channel.basic_qos(prefetch_count=1)  # Брать по одному сообщению
channel.basic_consume(queue="order_processing", on_message_callback=on_message)

# publish_order({"id": "123", "product": "Book", "qty": 2})
# channel.start_consuming()

connection.close()
```

---

### Dead Letter Queue (DLQ)

Очередь для сообщений, которые не удалось обработать:

```python
# Создаём DLQ
channel.queue_declare(queue="order_processing_dead", durable=True)

# Основная очередь с настройкой DLQ
channel.queue_declare(
    queue="order_processing",
    durable=True,
    arguments={
        "x-dead-letter-exchange": "",
        "x-dead-letter-routing-key": "order_processing_dead",
        "x-message-ttl": 30000,  # 30 сек — потом в DLQ
    }
)

def on_message(ch, method, properties, body):
    order = json.loads(body)
    try:
        process_order(order)
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        print(f"Ошибка: {e}")
        # reject + requeue=False → сообщение уйдёт в DLQ
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)
```

---

## Сравнение messaging протоколов

| Протокол | Модель | Целевая аудитория | Сложность | Гарантии |
|---------|--------|-----------------|---------|---------|
| WebSocket | Push/bidirectional | Веб-клиенты | Средняя | Нет (TCP) |
| MQTT | Pub/Sub | IoT, embedded | Низкая | QoS 0/1/2 |
| AMQP | Pub/Sub + Queue | Enterprise | Высокая | Да (ACK) |
| Kafka | Log/Streaming | Big data | Высокая | At-least-once |
| Redis Pub/Sub | Pub/Sub | Кэш + мессенджер | Низкая | Нет |

---

## Когда что выбирать

```
Реалтайм в браузере?         → WebSocket
IoT устройства?              → MQTT
Надёжные задачи/микросервисы? → AMQP (RabbitMQ)
Потоковая обработка данных?  → Kafka
Простое pub/sub внутри сервиса? → Redis Pub/Sub
```
