# Как реализовать свой протокол (TICODE - https://ticode.online/)

## Когда нужен свой протокол

Свой протокол нужен когда:
- Стандартные (HTTP, WebSocket) слишком тяжёлые для вашей задачи
- Нужна специфическая семантика сообщений
- Оптимизация под конкретное железо/сеть
- Обучение (лучший способ понять — реализовать)

---

## Дизайн протокола: что нужно решить

### 1. Формат сообщения

**Вариант А: Fixed-length header + variable body**

```
┌────────────────────────────────┬────────────────────────────┐
│ HEADER (фиксированный, N байт) │ BODY (длина из заголовка)  │
└────────────────────────────────┴────────────────────────────┘
```

**Вариант Б: Length-prefixed messages**

```
┌──────────────────────┬─────────────────────────────────────┐
│ LENGTH (4 байта)     │ MESSAGE (length байт)               │
└──────────────────────┴─────────────────────────────────────┘
```

**Вариант В: Delimiter-based**

```
MESSAGE_DATA\n
```

Не рекомендуется для бинарных данных (разделитель может быть в данных).

---

### 2. Порядок байт (Byte Order / Endianness)

```python
import struct

number = 0x01020304  # Число 16909060

# Big-endian (сетевой порядок): от старшего к младшему
big = struct.pack(">I", number)
print(list(big))   # [1, 2, 3, 4]

# Little-endian (x86 CPU): от младшего к старшему
little = struct.pack("<I", number)
print(list(little))  # [4, 3, 2, 1]

# В сетевых протоколах используют big-endian (network byte order)
# struct format: ! = network (big-endian)
```

**Правило**: всегда явно выбирайте порядок байт и документируйте его. Сетевые протоколы обычно используют **big-endian**.

---

### 3. Версионирование

Закладывайте версию с самого начала:

```python
import struct

# Заголовок с версией
# Version(1 байт) | Type(1 байт) | Length(4 байта) | Checksum(4 байта)
HEADER_FORMAT = "!BBII"  # Big-endian, 2 bytes + 2 ints
HEADER_SIZE = struct.calcsize(HEADER_FORMAT)  # 10 байт
```

---

## Реализация: простой бинарный протокол

Спроектируем протокол для чат-приложения.

### Дизайн

```
Заголовок (10 байт):
┌──────────┬──────────┬──────────────────────┬─────────────────────┐
│ version  │  type    │       length         │      checksum       │
│  1 байт  │  1 байт  │       4 байта        │      4 байта        │
└──────────┴──────────┴──────────────────────┴─────────────────────┘

Типы сообщений:
  0x01 — HELLO (установка соединения)
  0x02 — MESSAGE (текстовое сообщение)
  0x03 — ACK (подтверждение)
  0x04 — PING
  0x05 — PONG
  0xFF — ERROR

HELLO payload:
  username (length-prefixed string)

MESSAGE payload:
  message_id (4 байта)
  timestamp (8 байт, Unix epoch ms)
  text (length-prefixed string)
```

---

### Реализация протокола

```python
import struct
import zlib
import time
from dataclasses import dataclass
from enum import IntEnum
from typing import Optional

PROTOCOL_VERSION = 1
HEADER_FORMAT = "!BBII"   # version, type, length, checksum
HEADER_SIZE = struct.calcsize(HEADER_FORMAT)

class MessageType(IntEnum):
    HELLO   = 0x01
    MESSAGE = 0x02
    ACK     = 0x03
    PING    = 0x04
    PONG    = 0x05
    ERROR   = 0xFF

@dataclass
class Frame:
    type: MessageType
    payload: bytes
    version: int = PROTOCOL_VERSION

def encode_string(s: str) -> bytes:
    encoded = s.encode("utf-8")
    return struct.pack("!H", len(encoded)) + encoded  # 2-byte length prefix

def decode_string(data: bytes, offset: int = 0) -> tuple[str, int]:
    length = struct.unpack_from("!H", data, offset)[0]
    offset += 2
    text = data[offset:offset + length].decode("utf-8")
    return text, offset + length

def encode_frame(msg_type: MessageType, payload: bytes) -> bytes:
    checksum = zlib.crc32(payload) & 0xFFFFFFFF
    header = struct.pack(HEADER_FORMAT, PROTOCOL_VERSION, int(msg_type), len(payload), checksum)
    return header + payload

def decode_frame(data: bytes) -> tuple[Frame, bytes]:
    if len(data) < HEADER_SIZE:
        raise ValueError("Недостаточно данных для заголовка")
    
    version, msg_type, length, expected_checksum = struct.unpack_from(HEADER_FORMAT, data)
    
    if version != PROTOCOL_VERSION:
        raise ValueError(f"Неподдерживаемая версия протокола: {version}")
    
    total_size = HEADER_SIZE + length
    if len(data) < total_size:
        raise ValueError(f"Неполное сообщение: нужно {total_size}, есть {len(data)}")
    
    payload = data[HEADER_SIZE:total_size]
    actual_checksum = zlib.crc32(payload) & 0xFFFFFFFF
    
    if actual_checksum != expected_checksum:
        raise ValueError(f"Ошибка контрольной суммы: ожидали {expected_checksum}, получили {actual_checksum}")
    
    frame = Frame(type=MessageType(msg_type), payload=payload)
    remaining = data[total_size:]
    return frame, remaining

# Хелперы для создания конкретных сообщений

def make_hello(username: str) -> bytes:
    payload = encode_string(username)
    return encode_frame(MessageType.HELLO, payload)

def make_message(text: str, message_id: int = 0) -> bytes:
    timestamp = int(time.time() * 1000)
    payload = struct.pack("!IQ", message_id, timestamp) + encode_string(text)
    return encode_frame(MessageType.MESSAGE, payload)

def make_ping() -> bytes:
    return encode_frame(MessageType.PING, b"")

def make_pong() -> bytes:
    return encode_frame(MessageType.PONG, b"")

def make_ack(message_id: int) -> bytes:
    payload = struct.pack("!I", message_id)
    return encode_frame(MessageType.ACK, payload)

def parse_message_payload(payload: bytes) -> dict:
    msg_id, timestamp = struct.unpack_from("!IQ", payload)
    text, _ = decode_string(payload, 12)  # 4 + 8 байт
    return {"id": msg_id, "timestamp": timestamp, "text": text}

def parse_hello_payload(payload: bytes) -> str:
    username, _ = decode_string(payload)
    return username
```

---

### Сервер на asyncio

```python
import asyncio
import logging
from collections import defaultdict

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ChatServer:
    def __init__(self):
        self.clients: dict[asyncio.StreamWriter, str] = {}
        self._next_msg_id = 1
    
    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        addr = writer.get_extra_info("peername")
        logger.info(f"Новое соединение: {addr}")
        
        buffer = b""
        username = None
        
        try:
            while True:
                # Читаем данные
                data = await reader.read(4096)
                if not data:
                    break
                
                buffer += data
                
                # Обрабатываем все доступные фреймы
                while len(buffer) >= HEADER_SIZE:
                    try:
                        frame, buffer = decode_frame(buffer)
                    except ValueError as e:
                        if "Неполное сообщение" in str(e):
                            break  # Ждём больше данных
                        logger.error(f"Ошибка парсинга: {e}")
                        break
                    
                    # Обрабатываем фрейм
                    response = await self._handle_frame(frame, username, writer)
                    
                    if frame.type == MessageType.HELLO:
                        username = parse_hello_payload(frame.payload)
                        self.clients[writer] = username
                        logger.info(f"Вошёл: {username}")
        
        except Exception as e:
            logger.error(f"Ошибка: {e}")
        finally:
            if writer in self.clients:
                username = self.clients.pop(writer)
                logger.info(f"Вышел: {username}")
            writer.close()
    
    async def _handle_frame(
        self, frame: Frame, username: Optional[str], writer: asyncio.StreamWriter
    ) -> None:
        if frame.type == MessageType.PING:
            writer.write(make_pong())
            await writer.drain()
        
        elif frame.type == MessageType.MESSAGE:
            if not username:
                return
            
            msg_data = parse_message_payload(frame.payload)
            msg_id = self._next_msg_id
            self._next_msg_id += 1
            
            logger.info(f"[{username}]: {msg_data['text']}")
            
            # Подтверждаем получение
            writer.write(make_ack(msg_id))
            await writer.drain()
            
            # Рассылаем всем остальным
            broadcast = make_message(f"{username}: {msg_data['text']}", msg_id)
            for client_writer, client_name in list(self.clients.items()):
                if client_writer != writer:
                    try:
                        client_writer.write(broadcast)
                        await client_writer.drain()
                    except Exception:
                        pass
    
    async def start(self, host: str = "127.0.0.1", port: int = 9000):
        server = await asyncio.start_server(self.handle_client, host, port)
        logger.info(f"Чат сервер запущен на {host}:{port}")
        async with server:
            await server.serve_forever()

# asyncio.run(ChatServer().start())
```

---

### Клиент

```python
import asyncio

class ChatClient:
    def __init__(self, host: str = "127.0.0.1", port: int = 9000):
        self.host = host
        self.port = port
        self.reader: asyncio.StreamReader = None
        self.writer: asyncio.StreamWriter = None
        self.buffer = b""
    
    async def connect(self, username: str):
        self.reader, self.writer = await asyncio.open_connection(self.host, self.port)
        logger.info(f"Подключён к {self.host}:{self.port}")
        
        # Отправляем HELLO
        self.writer.write(make_hello(username))
        await self.writer.drain()
    
    async def send_message(self, text: str):
        self.writer.write(make_message(text))
        await self.writer.drain()
    
    async def receive_loop(self):
        while True:
            data = await self.reader.read(4096)
            if not data:
                break
            
            self.buffer += data
            
            while len(self.buffer) >= HEADER_SIZE:
                try:
                    frame, self.buffer = decode_frame(self.buffer)
                    await self._handle_frame(frame)
                except ValueError as e:
                    if "Неполное" in str(e):
                        break
                    break
    
    async def _handle_frame(self, frame: Frame):
        if frame.type == MessageType.MESSAGE:
            msg = parse_message_payload(frame.payload)
            print(f"\n>>> {msg['text']}")
        elif frame.type == MessageType.PONG:
            print("PONG получен")
        elif frame.type == MessageType.ACK:
            msg_id = struct.unpack_from("!I", frame.payload)[0]
            print(f"ACK для сообщения {msg_id}")
    
    async def disconnect(self):
        self.writer.close()
        await self.writer.wait_closed()

async def chat_demo():
    client = ChatClient()
    await client.connect("Alice")
    
    # Запускаем приём в фоне
    recv_task = asyncio.create_task(client.receive_loop())
    
    # Отправляем сообщения
    await client.send_message("Привет!")
    await asyncio.sleep(1)
    await client.send_message("Как дела?")
    await asyncio.sleep(2)
    
    recv_task.cancel()
    await client.disconnect()

# asyncio.run(chat_demo())
```

---

## Типичные проблемы при реализации протоколов

### 1. Частичное чтение (partial reads)

TCP — **потоковый** протокол. Одна `recv()` может вернуть часть сообщения.

```python
# Неправильно:
data = sock.recv(1024)
frame = decode_frame(data)  # Может упасть если данные неполные

# Правильно — буфер:
buffer = b""

def process_incoming(new_data: bytes):
    global buffer
    buffer += new_data
    
    while len(buffer) >= HEADER_SIZE:
        # Сначала смотрим длину
        _, _, length, _ = struct.unpack_from(HEADER_FORMAT, buffer)
        total = HEADER_SIZE + length
        
        if len(buffer) < total:
            break  # Ждём остальные данные
        
        frame_data = buffer[:total]
        buffer = buffer[total:]
        
        frame = decode_frame_exact(frame_data)
        handle_frame(frame)
```

---

### 2. Наивный recv

```python
# Неправильно: recv может вернуть меньше N байт
header = sock.recv(HEADER_SIZE)  # Нет гарантии что придёт ровно 10 байт

# Правильно:
def recv_exactly(sock: socket.socket, n: int) -> bytes:
    data = b""
    while len(data) < n:
        chunk = sock.recv(n - len(data))
        if not chunk:
            raise EOFError("Соединение закрыто")
        data += chunk
    return data

header = recv_exactly(sock, HEADER_SIZE)
```

---

### 3. Состояние гонки в многопоточном сервере

```python
import threading

class Server:
    def __init__(self):
        self.clients: dict = {}
        self._lock = threading.Lock()  # Защита от race condition
    
    def add_client(self, sock, username):
        with self._lock:
            self.clients[sock] = username
    
    def broadcast(self, message: bytes, exclude=None):
        with self._lock:
            clients_copy = dict(self.clients)
        
        # Рассылаем вне лока — чтобы не блокировать на медленных клиентах
        for sock, username in clients_copy.items():
            if sock != exclude:
                try:
                    sock.sendall(message)
                except Exception:
                    pass
```

---

## Чеклист при разработке протокола

```
□ Формат сообщения задокументирован
□ Порядок байт явно указан (big-endian/little-endian)
□ Версия протокола в заголовке
□ Контрольная сумма для целостности
□ Обработка частичных чтений (буфер)
□ Таймауты на все IO операции
□ Обработка разрыва соединения
□ Keepalive/Heartbeat механизм (PING/PONG)
□ Ограничение размера сообщения (защита от OOM)
□ Код ошибок и их обработка
□ Graceful shutdown
□ Логирование для отладки
```

---

## Пример: минимальный рабочий протокол за 50 строк

```python
import asyncio
import struct

# Протокол: [4 байта length][данные]
# Данные: UTF-8 строка

async def send_msg(writer: asyncio.StreamWriter, text: str):
    data = text.encode()
    writer.write(struct.pack("!I", len(data)) + data)
    await writer.drain()

async def recv_msg(reader: asyncio.StreamReader) -> str:
    length_bytes = await reader.readexactly(4)
    length = struct.unpack("!I", length_bytes)[0]
    if length > 1_000_000:
        raise ValueError("Слишком большое сообщение")
    data = await reader.readexactly(length)
    return data.decode()

async def server_handler(reader, writer):
    addr = writer.get_extra_info("peername")
    print(f"+ {addr}")
    try:
        while True:
            msg = await asyncio.wait_for(recv_msg(reader), timeout=30)
            print(f"[{addr}] {msg}")
            await send_msg(writer, f"Echo: {msg}")
    except (asyncio.IncompleteReadError, asyncio.TimeoutError):
        pass
    finally:
        print(f"- {addr}")
        writer.close()

async def run_server():
    server = await asyncio.start_server(server_handler, "127.0.0.1", 9999)
    async with server:
        await server.serve_forever()

async def run_client():
    reader, writer = await asyncio.open_connection("127.0.0.1", 9999)
    await send_msg(writer, "Привет, протокол!")
    response = await recv_msg(reader)
    print(f"Ответ: {response}")
    writer.close()

# asyncio.run(run_server())
# asyncio.run(run_client())
```

---

## Инструменты для отладки протоколов

```bash
# Wireshark — визуальный анализ пакетов
# tcpdump — захват пакетов в терминале
tcpdump -i lo -w capture.pcap port 9000

# netcat — тестирование TCP/UDP
nc 127.0.0.1 9000
echo "TEST" | nc 127.0.0.1 9000

# socat — более мощный netcat
socat - TCP:127.0.0.1:9000

# Redis CLI (для Redis протокола)
redis-cli -h localhost -p 6379 PING

# openssl — тестирование TLS
openssl s_client -connect google.com:443
```

```python
# Python: простой hex dump для отладки
def hexdump(data: bytes, width: int = 16):
    for i in range(0, len(data), width):
        chunk = data[i:i + width]
        hex_part = " ".join(f"{b:02x}" for b in chunk)
        ascii_part = "".join(chr(b) if 32 <= b < 127 else "." for b in chunk)
        print(f"{i:04x}  {hex_part:<{width*3}}  {ascii_part}")

hexdump(make_message("Hello, Protocol!"))
# 0000  01 02 00 00 00 1e a3 b4  c5 d6 01 00 00 00 00 00  ................
# 0010  ...
```
