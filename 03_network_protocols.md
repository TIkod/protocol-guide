# Сетевые протоколы: IP, TCP, UDP, ICMP (TICODE - https://ticode.online/)

## IP — Internet Protocol

### Что такое IP

IP (Internet Protocol) — основа интернета. Отвечает за **адресацию** и **маршрутизацию** пакетов от отправителя к получателю через множество промежуточных узлов.

Ключевые свойства IP:
- **Connectionless** — нет соединения, каждый пакет независим
- **Best effort** — доставка не гарантирована, нет подтверждений
- **Unreliable** — пакеты могут теряться, дублироваться, приходить не по порядку

Надёжность — задача протоколов выше (TCP).

---

### IPv4

IPv4 адрес — 32 бита, записывается как 4 числа 0-255 через точку.

```
192.168.1.100
  │   │ │  └── Хост (100)
  │   │ └───── Подсеть (1)
  │   └─────── Сеть класса C (168)
  └─────────── Сеть (192)
```

Диапазоны:
- `10.x.x.x`, `172.16-31.x.x`, `192.168.x.x` — приватные (локальные сети)
- `127.0.0.1` — loopback (сам себе)
- Всё остальное — публичные адреса интернета

```python
import socket
import struct

# Преобразование IP в число и обратно
ip_str = "192.168.1.100"
ip_num = struct.unpack("!I", socket.inet_aton(ip_str))[0]
print(f"{ip_str} = {ip_num} = {ip_num:032b}")
# 192.168.1.100 = 3232235876 = 11000000101010000000000101100100

# Обратно
ip_back = socket.inet_ntoa(struct.pack("!I", ip_num))
print(ip_back)  # 192.168.1.100
```

**Проблема IPv4**: 32 бита = ~4.3 млрд адресов. В 2011 году они закончились.

---

### IPv6

IPv6 — 128 бит, ~340 ундециллионов адресов. Записывается как 8 групп по 16 бит в hex.

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
2001:db8:85a3::8a2e:370:7334   # сокращённая форма (:: = группы нулей)
::1                             # loopback
```

```python
import socket

# Резолв IPv6
addr = socket.getaddrinfo("google.com", 80, socket.AF_INET6)
print(addr[0][4][0])  # например: 2607:f8b0:4004:c07::71
```

---

### Структура IP пакета (IPv4 header)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┴─┴─┴─┼─┴─┴─┴─┴─┴─┴─┴─┼─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┤
│Version│  IHL  │   DSCP  │ECN│         Total Length         │
├───────┴───────┴─────────┴───┴─────────────────────────────┤
│          Identification           │Flags│  Fragment Offset  │
├─────────────────────────┬─────────┴─────┴───────────────────┤
│    Time to Live (TTL)   │  Protocol  │    Header Checksum    │
├─────────────────────────┴────────────┴───────────────────────┤
│                      Source IP Address                        │
├───────────────────────────────────────────────────────────────┤
│                   Destination IP Address                      │
└───────────────────────────────────────────────────────────────┘
```

Важные поля:
- **TTL (Time to Live)** — счётчик хопов, уменьшается на 1 на каждом маршрутизаторе. При TTL=0 пакет уничтожается (защита от бесконечных петель)
- **Protocol** — какой протокол внутри: 6=TCP, 17=UDP, 1=ICMP
- **Source/Destination** — откуда и куда

```python
import struct
import socket

def parse_ip_header(data: bytes) -> dict:
    header = struct.unpack("!BBHHHBBH4s4s", data[:20])
    return {
        "version": header[0] >> 4,
        "ihl": (header[0] & 0xF) * 4,
        "ttl": header[5],
        "protocol": header[6],
        "src_ip": socket.inet_ntoa(header[8]),
        "dst_ip": socket.inet_ntoa(header[9]),
    }
```

---

## TCP — Transmission Control Protocol

### Что такое TCP

TCP — надёжный, упорядоченный, потоковый протокол. Поверх ненадёжного IP создаёт надёжный **канал**.

Гарантии TCP:
- ✅ Доставка (все байты дойдут)
- ✅ Порядок (в том же порядке, что отправлены)
- ✅ Целостность (без искажений)
- ✅ Контроль перегрузки (не перегрузит сеть)

Цена: задержка, overhead от подтверждений.

---

### TCP Three-Way Handshake

```
Клиент                              Сервер
  │                                    │
  │──── SYN (seq=x) ──────────────────►│  "Привет, хочу соединиться"
  │                                    │
  │◄─── SYN-ACK (seq=y, ack=x+1) ─────│  "Принял, вот мой seq"
  │                                    │
  │──── ACK (ack=y+1) ────────────────►│  "Принял твой seq"
  │                                    │
  │   [соединение установлено]         │
  │                                    │
  │──── данные ───────────────────────►│
  │◄─── ACK ───────────────────────────│
```

```python
import socket

# Клиент — TCP соединение
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.connect(("example.com", 80))  # SYN → SYN-ACK → ACK за кулисами

client.send(b"GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
response = b""
while chunk := client.recv(4096):
    response += chunk

client.close()
print(response[:200])
```

---

### TCP сервер

```python
import socket
import threading

def handle_client(conn: socket.socket, addr: tuple):
    print(f"Подключился: {addr}")
    with conn:
        while True:
            data = conn.recv(1024)
            if not data:
                break
            print(f"Получено: {data.decode()}")
            conn.sendall(data)  # echo back

def tcp_server(host: str = "127.0.0.1", port: int = 9000):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
        server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server.bind((host, port))
        server.listen(5)
        print(f"Слушаю {host}:{port}")
        
        while True:
            conn, addr = server.accept()
            thread = threading.Thread(target=handle_client, args=(conn, addr))
            thread.daemon = True
            thread.start()

# tcp_server()
```

---

### Состояния TCP соединения

```
CLOSED → LISTEN (сервер ждёт)
CLOSED → SYN_SENT (клиент отправил SYN)
SYN_SENT → ESTABLISHED (после SYN-ACK и ACK)

Закрытие:
ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

```bash
# Посмотреть активные TCP соединения
# ss -tn  или  netstat -tn
```

---

### Sliding Window и управление потоком

TCP не ждёт подтверждения каждого пакета — отправляет несколько (window). Получатель сообщает, сколько он готов принять (`window size`).

```
Отправитель:   [1][2][3][4][5][6][7][8]
               └─window(4)─┘
               Отправлены 1,2,3,4 — ждём ACK

Получатель:   ← ACK 3 (принял 1,2,3)
               
Отправитель:   [1][2][3][4][5][6][7][8]
                        └─window─┘
               Отправляем 5 (сдвинули окно)
```

---

## UDP — User Datagram Protocol

### Что такое UDP

UDP — простой, быстрый, ненадёжный протокол. **Нет соединения, нет гарантий, нет порядка**.

```
UDP дейтаграмма:
┌──────────┬──────────┬────────┬──────────┬─────────┐
│ src port │ dst port │ length │ checksum │  data   │
│  2 байта │  2 байта │ 2 байта│  2 байта │   ...   │
└──────────┴──────────┴────────┴──────────┴─────────┘
```

Всего 8 байт заголовка против 20+ у TCP.

Когда использовать UDP:
- Видеостриминг, VoIP — лучше потерять кадр, чем ждать
- DNS запросы — один маленький запрос-ответ
- Игры — важна актуальность, не история
- DHCP — нет ещё IP адреса для TCP

---

### UDP сервер и клиент

```python
import socket

# UDP сервер
def udp_server(host: str = "127.0.0.1", port: int = 9001):
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as server:
        server.bind((host, port))
        print(f"UDP сервер на {host}:{port}")
        
        while True:
            data, addr = server.recvfrom(1024)
            print(f"От {addr}: {data.decode()}")
            server.sendto(b"OK", addr)  # Ответить обратно

# UDP клиент
def udp_client():
    with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
        # Нет connect() — просто отправляем
        client.sendto(b"Hello UDP!", ("127.0.0.1", 9001))
        
        # Получаем ответ
        client.settimeout(2.0)  # Нет гарантий — нужен таймаут
        try:
            data, server_addr = client.recvfrom(1024)
            print(f"Ответ: {data.decode()}")
        except socket.timeout:
            print("Нет ответа — пакет потерялся")
```

---

### TCP vs UDP сравнение

| Характеристика | TCP | UDP |
|---------------|-----|-----|
| Установка соединения | Handshake (3 пакета) | Нет |
| Гарантия доставки | Да | Нет |
| Порядок данных | Сохраняется | Не гарантируется |
| Скорость | Медленнее | Быстрее |
| Заголовок | 20+ байт | 8 байт |
| Применение | HTTP, SSH, FTP | DNS, Video, Games |
| Контроль перегрузки | Да | Нет |

---

## ICMP — Internet Control Message Protocol

### Что такое ICMP

ICMP — служебный протокол для передачи **сообщений об ошибках и диагностики** в IP сетях. Не передаёт данные пользователя.

Типы сообщений:
- `Type 0` — Echo Reply (ответ на ping)
- `Type 3` — Destination Unreachable (цель недостижима)
- `Type 8` — Echo Request (ping)
- `Type 11` — Time Exceeded (TTL истёк — используется в traceroute)

---

### Как работает ping

```
Клиент          Роутер         Сервер
  │                │              │
  │─ICMP Echo ────────────────── ►│  "Ты там?"
  │                │              │
  │◄──────────────────── ICMP Reply│  "Да, я тут"
  │                │              │
  │  RTT = время туда + обратно   │
```

```python
import subprocess
import re

def ping(host: str, count: int = 3) -> dict:
    result = subprocess.run(
        ["ping", "-c", str(count), host],
        capture_output=True, text=True
    )
    
    output = result.stdout
    
    # Парсим статистику
    loss_match = re.search(r"(\d+)% packet loss", output)
    rtt_match = re.search(r"min/avg/max.*?= ([\d.]+)/([\d.]+)/([\d.]+)", output)
    
    return {
        "host": host,
        "packet_loss": int(loss_match.group(1)) if loss_match else 100,
        "rtt_min": float(rtt_match.group(1)) if rtt_match else None,
        "rtt_avg": float(rtt_match.group(2)) if rtt_match else None,
        "rtt_max": float(rtt_match.group(3)) if rtt_match else None,
    }

result = ping("8.8.8.8")
print(result)
# {'host': '8.8.8.8', 'packet_loss': 0, 'rtt_min': 1.2, 'rtt_avg': 1.5, ...}
```

---

### Как работает traceroute

Traceroute отправляет пакеты с TTL=1, 2, 3... Каждый маршрутизатор, где TTL=0, возвращает ICMP "Time Exceeded" — так мы узнаём маршрут.

```
TTL=1: пакет умирает на роутере 1, получаем ICMP от роутера 1
TTL=2: пакет умирает на роутере 2, получаем ICMP от роутера 2
TTL=3: пакет умирает на роутере 3, получаем ICMP от роутера 3
...
TTL=N: пакет достигает цели
```

```python
import subprocess

def traceroute(host: str):
    result = subprocess.run(
        ["traceroute", "-n", host],  # -n = не резолвить имена
        capture_output=True, text=True
    )
    print(result.stdout)

traceroute("google.com")
```

---

## Практика: сырые сокеты и анализ пакетов

```python
import socket
import struct

def create_raw_icmp_echo(identifier: int, sequence: int, payload: bytes) -> bytes:
    # Тип=8 (Echo Request), Код=0
    icmp_type = 8
    icmp_code = 0
    checksum = 0
    
    # Пакет без контрольной суммы
    header = struct.pack("!BBHHH", icmp_type, icmp_code, checksum, identifier, sequence)
    packet = header + payload
    
    # Считаем контрольную сумму
    checksum = calculate_checksum(packet)
    header = struct.pack("!BBHHH", icmp_type, icmp_code, checksum, identifier, sequence)
    
    return header + payload

def calculate_checksum(data: bytes) -> int:
    if len(data) % 2 != 0:
        data += b'\x00'
    
    total = 0
    for i in range(0, len(data), 2):
        word = (data[i] << 8) + data[i + 1]
        total += word
    
    total = (total >> 16) + (total & 0xFFFF)
    total += total >> 16
    return ~total & 0xFFFF

# Использование (требует root/admin прав):
# sock = socket.socket(socket.AF_INET, socket.SOCK_RAW, socket.IPPROTO_ICMP)
# packet = create_raw_icmp_echo(1, 1, b"Hello!")
# sock.sendto(packet, ("8.8.8.8", 0))
```

---

## Ключевые выводы

| Протокол | Уровень OSI | Задача | Надёжность |
|---------|-------------|--------|-----------|
| IP | 3 (Сетевой) | Адресация и маршрутизация | Best effort |
| TCP | 4 (Транспортный) | Надёжная доставка потока | Гарантирована |
| UDP | 4 (Транспортный) | Быстрая доставка датаграмм | Не гарантирована |
| ICMP | 3 (Сетевой) | Диагностика и ошибки | — |
