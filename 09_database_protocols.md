# Протоколы баз данных: PostgreSQL wire protocol, Redis RESP (TICODE - https://ticode.online/)

## Зачем знать протоколы баз данных

Большинство разработчиков используют ОРМ или клиентские библиотеки и не думают о протоколах. Но знание протоколов помогает:
- Понять почему запрос медленный (сколько round-trips)
- Писать кастомные клиенты/прокси
- Оптимизировать connection pooling
- Разобраться в ошибках на уровне соединения

---

## PostgreSQL Wire Protocol (версия 3)

### Архитектура

PostgreSQL использует **message-based бинарный протокол** поверх TCP (порт 5432).

Два режима:
- **Simple Query** — одно сообщение = один SQL запрос
- **Extended Query** — Parse → Bind → Execute (подготовленные выражения)

---

### Установка соединения

```
Клиент                          PostgreSQL
  │                                  │
  │──── StartupMessage ─────────────►│
  │   {protocol_version: 3.0,        │
  │    user: "alice",                │
  │    database: "mydb"}             │
  │                                  │
  │◄─── AuthenticationCleartextPassword │ (или MD5, SCRAM-SHA-256)
  │                                  │
  │──── PasswordMessage ────────────►│
  │                                  │
  │◄─── AuthenticationOK ────────────│
  │◄─── ParameterStatus ─────────────│ (server_encoding, timezone...)
  │◄─── BackendKeyData ──────────────│ (pid для cancel)
  │◄─── ReadyForQuery ───────────────│ (статус: 'I'=idle, 'T'=transaction)
  │                                  │
  │   [готов к запросам]             │
```

---

### Simple Query протокол

```
Клиент                          PostgreSQL
  │                                  │
  │──── Query('SELECT * FROM users') ►│
  │                                  │
  │◄─── RowDescription ──────────────│  описание колонок
  │◄─── DataRow (id=1, name=Alice) ──│  строка данных
  │◄─── DataRow (id=2, name=Bob) ────│  строка данных
  │◄─── CommandComplete ('SELECT 2') │  завершение
  │◄─── ReadyForQuery ───────────────│  готов
```

---

### Extended Query протокол (prepared statements)

```
Клиент                          PostgreSQL
  │                                  │
  │──── Parse(name="", query, types)►│  подготовить
  │──── Bind(params=[42]) ──────────►│  привязать параметры
  │──── Execute ────────────────────►│  выполнить
  │──── Sync ───────────────────────►│  синхронизировать
  │                                  │
  │◄─── ParseComplete ───────────────│
  │◄─── BindComplete ────────────────│
  │◄─── DataRow ...                  │
  │◄─── CommandComplete              │
  │◄─── ReadyForQuery ───────────────│
```

Преимущество: один Parse для повторяющихся запросов, разные параметры — без SQL инъекций.

---

### Формат сообщений PostgreSQL

Каждое сообщение:
```
Byte  — тип сообщения (1 байт)
Int32 — длина сообщения включая эти 4 байта
...   — тело сообщения
```

```python
import struct
import socket

def read_message(sock: socket.socket) -> tuple[str, bytes]:
    # Читаем тип (1 байт)
    msg_type = sock.recv(1).decode("ascii")
    
    # Читаем длину (4 байта, big-endian)
    length_bytes = sock.recv(4)
    length = struct.unpack(">I", length_bytes)[0]
    
    # Читаем тело (length - 4 байта уже прочитаны)
    body = sock.recv(length - 4)
    
    return msg_type, body

def send_startup(sock: socket.socket, user: str, database: str):
    # Startup message не имеет типа — особый случай
    params = f"user\x00{user}\x00database\x00{database}\x00\x00"
    protocol_version = 196608  # 3.0
    
    body = struct.pack(">I", protocol_version) + params.encode()
    length = len(body) + 4
    
    sock.send(struct.pack(">I", length) + body)
```

---

### Простой PostgreSQL клиент

```python
import asyncpg  # pip install asyncpg — быстрый async драйвер
import asyncio

async def demo():
    # asyncpg напрямую работает с протоколом PostgreSQL
    conn = await asyncpg.connect(
        host="localhost",
        port=5432,
        user="alice",
        password="secret",
        database="mydb",
    )
    
    # Simple query
    rows = await conn.fetch("SELECT id, name FROM users WHERE active = $1", True)
    for row in rows:
        print(dict(row))
    
    # Prepared statement (автоматически кэшируется)
    stmt = await conn.prepare("SELECT * FROM users WHERE id = $1")
    user = await stmt.fetchrow(42)
    
    # Transaction
    async with conn.transaction():
        await conn.execute("INSERT INTO users(name, email) VALUES($1, $2)", "Bob", "bob@ex.com")
        await conn.execute("UPDATE counters SET value = value + 1 WHERE name = 'users'")
    
    # COPY protocol — быстрая загрузка данных
    result = await conn.copy_to_table(
        "users",
        records=[
            (1, "Alice", "alice@ex.com"),
            (2, "Bob", "bob@ex.com"),
        ],
        columns=["id", "name", "email"],
    )
    
    await conn.close()

# asyncio.run(demo())
```

---

### Connection Pooling

PostgreSQL создание соединения дорогое (~10-50мс). Пул соединений решает это:

```python
import asyncpg
import asyncio

async def setup_pool():
    pool = await asyncpg.create_pool(
        host="localhost",
        user="alice",
        password="secret",
        database="mydb",
        min_size=5,   # Минимум 5 соединений всегда открыты
        max_size=20,  # Максимум 20 одновременных соединений
    )
    return pool

async def handle_request(pool: asyncpg.Pool, user_id: int):
    # Берёт соединение из пула, возвращает после блока
    async with pool.acquire() as conn:
        return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
```

---

## Redis RESP — REdis Serialization Protocol

### Что такое RESP

RESP (Redis Serialization Protocol) — простой текстовый протокол Redis. Намеренно простой — легко реализовать, легко отлаживать через telnet/netcat.

Порт Redis: **6379**

---

### RESP формат

Каждая строка в RESP заканчивается `\r\n` (CRLF).

```
+ — Simple String   +OK\r\n
- — Error           -ERR unknown command\r\n
: — Integer         :42\r\n
$ — Bulk String     $5\r\nhello\r\n     (5 байт, потом данные)
* — Array           *2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n
_ — Null (RESP3)    _\r\n
```

---

### Как выглядит Redis команда

Каждая команда — массив строк:

```
SET mykey "Hello"
     ↓ в RESP:
*3\r\n           ← массив из 3 элементов
$3\r\n           ← строка длиной 3
SET\r\n
$5\r\n           ← строка длиной 5
mykey\r\n
$5\r\n           ← строка длиной 5
Hello\r\n

Ответ: +OK\r\n
```

```
GET mykey
     ↓ в RESP:
*2\r\n
$3\r\n
GET\r\n
$5\r\n
mykey\r\n

Ответ: $5\r\nHello\r\n
```

---

### Redis вручную через сокет

```python
import socket

def send_command(sock: socket.socket, *args: str) -> str:
    # Формируем RESP команду
    cmd = f"*{len(args)}\r\n"
    for arg in args:
        arg_bytes = str(arg).encode()
        cmd += f"${len(arg_bytes)}\r\n{arg}\r\n"
    
    sock.send(cmd.encode())
    return read_response(sock)

def read_response(sock: socket.socket) -> any:
    line = b""
    while not line.endswith(b"\r\n"):
        line += sock.recv(1)
    line = line.decode().strip()
    
    prefix = line[0]
    data = line[1:]
    
    if prefix == "+":
        return data  # Simple string
    elif prefix == "-":
        raise RuntimeError(data)  # Error
    elif prefix == ":":
        return int(data)  # Integer
    elif prefix == "$":
        size = int(data)
        if size == -1:
            return None  # Null bulk string
        value = b""
        while len(value) < size:
            value += sock.recv(size - len(value))
        sock.recv(2)  # CRLF
        return value.decode()
    elif prefix == "*":
        count = int(data)
        if count == -1:
            return None
        return [read_response(sock) for _ in range(count)]

# Пример использования
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("127.0.0.1", 6379))

send_command(sock, "SET", "name", "Alice")           # → "OK"
send_command(sock, "GET", "name")                     # → "Alice"
send_command(sock, "INCR", "counter")                 # → 1
send_command(sock, "LPUSH", "list", "a", "b", "c")   # → 3

sock.close()
```

---

### Redis с библиотекой

```python
import redis  # pip install redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# Strings
r.set("name", "Alice", ex=3600)  # TTL 1 час
name = r.get("name")
r.incr("counter")

# Hash (словарь)
r.hset("user:42", mapping={"name": "Alice", "email": "alice@ex.com", "age": "30"})
user = r.hgetall("user:42")  # {'name': 'Alice', 'email': '...', 'age': '30'}

# List
r.rpush("queue", "task1", "task2", "task3")
task = r.lpop("queue")  # "task1" (FIFO очередь)

# Set
r.sadd("tags", "python", "backend", "api")
r.smembers("tags")  # {'python', 'backend', 'api'}

# Sorted Set (leaderboard)
r.zadd("scores", {"alice": 100, "bob": 85, "charlie": 92})
top3 = r.zrevrange("scores", 0, 2, withscores=True)

# Pub/Sub
pubsub = r.pubsub()
pubsub.subscribe("events")

r.publish("events", "user_logged_in")

for message in pubsub.listen():
    if message["type"] == "message":
        print(message["data"])

# Pipeline (batch команд за один round-trip)
pipe = r.pipeline()
pipe.set("a", 1)
pipe.set("b", 2)
pipe.incr("a")
results = pipe.execute()  # [True, True, 2] — три команды, один TCP запрос
```

---

### Redis паттерны

```python
import redis
import json
import functools
import hashlib

r = redis.Redis(decode_responses=True)

# Паттерн: кэш с TTL
def cache(ttl: int = 300):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = f"cache:{func.__name__}:{hashlib.md5(str(args).encode()).hexdigest()}"
            
            cached = r.get(key)
            if cached:
                return json.loads(cached)
            
            result = func(*args, **kwargs)
            r.set(key, json.dumps(result), ex=ttl)
            return result
        return wrapper
    return decorator

@cache(ttl=60)
def get_user_from_db(user_id: int) -> dict:
    # Тяжёлый запрос к БД
    return {"id": user_id, "name": "Alice"}

# Паттерн: distributed lock
import time

def acquire_lock(lock_name: str, timeout: int = 10) -> bool:
    lock_key = f"lock:{lock_name}"
    return r.set(lock_key, "1", nx=True, ex=timeout)  # nx=True: только если не существует

def release_lock(lock_name: str):
    r.delete(f"lock:{lock_name}")

# Паттерн: Rate limiting
def is_rate_limited(user_id: int, max_requests: int = 100, window: int = 60) -> bool:
    key = f"ratelimit:{user_id}:{int(time.time()) // window}"
    count = r.incr(key)
    if count == 1:
        r.expire(key, window)
    return count > max_requests
```

---

## Краткое сравнение

| | PostgreSQL | Redis |
|--|-----------|-------|
| Протокол | Бинарный, message-based | Текстовый (RESP) |
| Транспорт | TCP 5432 | TCP 6379 |
| Тип | Реляционная БД | In-memory хранилище |
| Данные | Таблицы, SQL | Строки, хеши, списки, множества |
| Персистентность | Да (всегда) | Опционально (RDB/AOF) |
| Транзакции | ACID | MULTI/EXEC (ограниченно) |
