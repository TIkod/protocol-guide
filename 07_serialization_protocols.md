# Протоколы сериализации: JSON, XML, Protobuf, MessagePack (TICODE - https://ticode.online/)

## Что такое сериализация

**Сериализация** — преобразование структуры данных (объекта, словаря) в последовательность байт для передачи или хранения.

**Десериализация** — обратный процесс.

```
Python dict → [сериализация] → байты → [передача по сети] → байты → [десериализация] → dict
```

Критерии выбора формата:
- **Читаемость** — можно ли прочитать без инструментов
- **Размер** — насколько компактно
- **Скорость** — сериализации и десериализации
- **Типизация** — поддержка типов данных
- **Схема** — нужно ли описание структуры

---

## JSON — JavaScript Object Notation

### Что такое JSON

JSON — самый популярный текстовый формат обмена данными. Читаемый человеком, поддерживается везде.

```json
{
  "id": 42,
  "name": "Alice",
  "active": true,
  "scores": [95, 87, 91],
  "address": {
    "city": "Moscow",
    "zip": "101000"
  },
  "metadata": null
}
```

**Типы данных JSON:**
- `string` — `"text"`
- `number` — `42`, `3.14`
- `boolean` — `true`, `false`
- `null` — `null`
- `array` — `[1, 2, 3]`
- `object` — `{"key": "value"}`

**Нет поддержки:** дат, бинарных данных, `undefined`, `NaN`, `Infinity`.

---

### JSON в Python

```python
import json
from datetime import datetime

data = {
    "id": 42,
    "name": "Alice",
    "scores": [95, 87, 91],
    "active": True,
    "metadata": None,
}

# Сериализация
json_str = json.dumps(data)
print(json_str)
# {"id": 42, "name": "Alice", "scores": [95, 87, 91], "active": true, "metadata": null}

# Красивый вывод
pretty = json.dumps(data, indent=2, ensure_ascii=False)
print(pretty)

# Десериализация
parsed = json.loads(json_str)
print(parsed["name"])  # Alice

# Файл
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

with open("data.json") as f:
    data = json.load(f)
```

---

### Custom encoder/decoder

```python
import json
from datetime import datetime
from dataclasses import dataclass, asdict

@dataclass
class User:
    id: int
    name: str
    created_at: datetime

# Кастомный encoder для несериализуемых типов
class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if hasattr(obj, "__dict__"):
            return obj.__dict__
        return super().default(obj)

user = User(id=1, name="Alice", created_at=datetime.now())

# С кастомным encoder
json_str = json.dumps(user, cls=CustomEncoder)
print(json_str)
# {"id": 1, "name": "Alice", "created_at": "2024-01-15T10:30:00"}

# dataclass через asdict
json_str = json.dumps(asdict(user), default=str)
```

---

### JSON Schema — валидация структуры

```python
import jsonschema  # pip install jsonschema

schema = {
    "type": "object",
    "properties": {
        "id": {"type": "integer", "minimum": 1},
        "name": {"type": "string", "minLength": 1},
        "email": {"type": "string", "format": "email"},
        "age": {"type": "integer", "minimum": 0, "maximum": 150},
    },
    "required": ["id", "name", "email"],
    "additionalProperties": False,
}

# Валидный
jsonschema.validate({"id": 1, "name": "Alice", "email": "a@b.com"}, schema)

# Невалидный
try:
    jsonschema.validate({"id": -1, "name": "", "email": "not-email"}, schema)
except jsonschema.ValidationError as e:
    print(e.message)
```

---

## XML — eXtensible Markup Language

### Что такое XML

XML — текстовый формат с тегами. Verbose (многословный), но строго типизированный через схемы (XSD).

```xml
<?xml version="1.0" encoding="UTF-8"?>
<users>
    <user id="42">
        <name>Alice</name>
        <email>alice@example.com</email>
        <scores>
            <score>95</score>
            <score>87</score>
        </scores>
        <active>true</active>
    </user>
</users>
```

XML используется в: SOAP веб-сервисах, конфигурационных файлах, офисных документах (DOCX, XLSX).

---

### XML в Python

```python
import xml.etree.ElementTree as ET

# Парсинг
xml_str = """
<users>
    <user id="42">
        <name>Alice</name>
        <email>alice@example.com</email>
    </user>
    <user id="43">
        <name>Bob</name>
        <email>bob@example.com</email>
    </user>
</users>
"""

root = ET.fromstring(xml_str)

for user in root.findall("user"):
    user_id = user.get("id")       # Атрибут
    name = user.find("name").text   # Дочерний элемент
    email = user.find("email").text
    print(f"ID: {user_id}, Name: {name}, Email: {email}")

# Создание XML
root = ET.Element("users")
user = ET.SubElement(root, "user", id="1")
ET.SubElement(user, "name").text = "Alice"
ET.SubElement(user, "email").text = "alice@example.com"

tree = ET.ElementTree(root)
ET.indent(tree, space="  ")
tree.write("users.xml", encoding="unicode", xml_declaration=True)
```

---

### JSON vs XML

| | JSON | XML |
|--|------|-----|
| Размер | Меньше | Больше (теги) |
| Читаемость | Хорошая | Хуже (verbose) |
| Атрибуты | Нет | Есть |
| Комментарии | Нет | Есть |
| Схема | JSON Schema | XSD (мощнее) |
| Применение | REST API, NoSQL | SOAP, конфиги, офис |

---

## Protocol Buffers (Protobuf)

### Что такое Protobuf

Protobuf (Protocol Buffers) — бинарный формат сериализации от Google. Требует схему (`.proto` файл). Намного компактнее и быстрее JSON.

---

### .proto схема

```protobuf
syntax = "proto3";

package users;

message Address {
    string city = 1;
    string zip = 2;
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    bool active = 4;
    repeated int32 scores = 5;  // массив
    Address address = 6;
    
    // Числа 1-15 кодируются 1 байтом (часто используемые поля)
}

message UserList {
    repeated User users = 1;
}
```

Числа (1, 2, 3...) — это **field numbers**, используются в бинарном представлении вместо имён.

---

### Protobuf в Python

```python
# Установка: pip install protobuf grpcio-tools
# Генерация кода: python -m grpc_tools.protoc -I. --python_out=. users.proto

# После генерации появится users_pb2.py
# from users_pb2 import User, UserList

# Демонстрация без генерации (вручную):
from google.protobuf import descriptor_pool, descriptor_pb2
from google.protobuf.message import Message

# Если код сгенерирован из .proto:
# user = User(id=42, name="Alice", email="alice@example.com", active=True)
# user.scores.extend([95, 87, 91])

# Сериализация
# serialized = user.SerializeToString()
# print(f"Размер Protobuf: {len(serialized)} байт")

# Десериализация
# user2 = User()
# user2.ParseFromString(serialized)
# print(user2.name)  # Alice
```

Реальный пример размеров:

```python
import json

data = {
    "id": 42,
    "name": "Alice Johnson",
    "email": "alice@example.com",
    "active": True,
    "scores": [95, 87, 91, 100, 78],
}

json_size = len(json.dumps(data).encode())
print(f"JSON: {json_size} байт")  # ~108 байт

# Protobuf для того же: ~45 байт (примерно в 2-3 раза меньше)
```

---

### Как работает бинарная кодировка Protobuf

```
Field 1 (id=42, type=varint):
  Tag:     (field_num=1, wire_type=0) = 0x08
  Value:   42 → 0x2A
  Итого:   08 2A

Field 2 (name="Alice", type=string):
  Tag:     (field_num=2, wire_type=2) = 0x12
  Length:  5 → 0x05
  Data:    "Alice" → 41 6C 69 63 65
  Итого:   12 05 41 6C 69 63 65
```

**Wire types:**
- `0` — Varint (int32, int64, bool, enum)
- `1` — 64-bit (double, fixed64)
- `2` — Length-delimited (string, bytes, embedded messages)
- `5` — 32-bit (float, fixed32)

---

## MessagePack

### Что такое MessagePack

MessagePack — "бинарный JSON". Та же структура данных что JSON, но в бинарном виде. Не требует схемы.

```python
import msgpack  # pip install msgpack

data = {
    "id": 42,
    "name": "Alice",
    "active": True,
    "scores": [95, 87, 91],
}

# Сериализация
packed = msgpack.packb(data, use_bin_type=True)
print(f"MessagePack: {len(packed)} байт")
print(f"JSON: {len(json.dumps(data).encode())} байт")

# Десериализация
unpacked = msgpack.unpackb(packed, raw=False)
print(unpacked)

# Потоковая распаковка (большие данные)
buf = packed + packed  # два сообщения
unpacker = msgpack.Unpacker(raw=False)
unpacker.feed(buf)
for item in unpacker:
    print(item)
```

---

## CBOR — Concise Binary Object Representation

Ещё один бинарный JSON-подобный формат. Используется в IoT и COSE (криптографические структуры данных).

```python
import cbor2  # pip install cbor2
from datetime import datetime

data = {
    "timestamp": datetime.now(),  # CBOR поддерживает даты нативно!
    "value": 23.5,
    "tags": ["sensor", "temperature"],
}

encoded = cbor2.dumps(data)
decoded = cbor2.loads(encoded)
print(decoded)
```

---

## Сравнение форматов

| Формат | Размер | Скорость | Читаем | Схема | Поддержка типов |
|--------|--------|---------|--------|-------|----------------|
| JSON | Средний | Средняя | Да | Опционально | Базовые |
| XML | Большой | Медленная | Да | XSD | Базовые |
| Protobuf | Малый | Быстрая | Нет | Обязательно | Богатые |
| MessagePack | Малый | Быстрая | Нет | Нет | Как JSON |
| CBOR | Малый | Быстрая | Нет | Нет | Расширенные |

### Бенчмарк (примерные цифры)

```python
import json
import msgpack
import timeit

data = {"users": [{"id": i, "name": f"User{i}", "active": True} for i in range(100)]}

# JSON
json_time = timeit.timeit(lambda: json.dumps(data), number=10000)
json_size = len(json.dumps(data).encode())

# MessagePack
msgpack_time = timeit.timeit(lambda: msgpack.packb(data), number=10000)
msgpack_size = len(msgpack.packb(data))

print(f"JSON:        {json_size:5} байт, {json_time:.3f}s")
print(f"MessagePack: {msgpack_size:5} байт, {msgpack_time:.3f}s")
# JSON:         2891 байт, 0.412s
# MessagePack:  1847 байт, 0.089s  (быстрее в ~5 раз, меньше в ~1.5 раза)
```

---

## Когда что выбирать

```
Читаемость важна (API, логи):    → JSON
Строгая схема нужна (микросервисы, gRPC): → Protobuf
Замена JSON без схемы:           → MessagePack
Работа с XML-системами (SOAP):   → XML
IoT с нестандартными типами:     → CBOR
```
