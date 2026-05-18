# Прикладные протоколы: HTTP, DNS, FTP, SMTP (TICODE - https://ticode.online/)

## HTTP — HyperText Transfer Protocol

### Что такое HTTP

HTTP — протокол передачи гипертекста, основа веба. Работает по модели **запрос-ответ** (request-response).

- **Stateless** — каждый запрос независим
- **Текстовый** (HTTP/1.1) / **Бинарный** (HTTP/2, HTTP/3)
- Поверх **TCP** (HTTP/1.1, HTTP/2) или **UDP/QUIC** (HTTP/3)
- Стандартный порт: **80** (HTTP), **443** (HTTPS)

---

### Структура HTTP запроса

```
GET /api/users/42 HTTP/1.1\r\n        ← Стартовая строка: метод URL версия
Host: api.example.com\r\n             ← Обязательный заголовок
Accept: application/json\r\n          ← Опциональные заголовки
Authorization: Bearer token123\r\n
\r\n                                  ← Пустая строка = конец заголовков
                                      ← Тело (для GET — пустое)
```

Для POST с телом:
```
POST /api/users HTTP/1.1\r\n
Host: api.example.com\r\n
Content-Type: application/json\r\n
Content-Length: 27\r\n
\r\n
{"name": "Alice", "age": 30}
```

---

### Структура HTTP ответа

```
HTTP/1.1 200 OK\r\n                   ← Статусная строка
Content-Type: application/json\r\n    ← Заголовки
Content-Length: 45\r\n
\r\n
{"id": 42, "name": "Alice", "age": 30}  ← Тело
```

---

### HTTP методы

| Метод | Действие | Идемпотентный | Безопасный |
|-------|---------|--------------|-----------|
| GET | Получить данные | Да | Да |
| POST | Создать | Нет | Нет |
| PUT | Заменить целиком | Да | Нет |
| PATCH | Частично обновить | Нет | Нет |
| DELETE | Удалить | Да | Нет |
| HEAD | Как GET, но без тела | Да | Да |
| OPTIONS | Узнать возможности | Да | Да |

**Идемпотентный** — повторный вызов даёт тот же результат (DELETE /users/1 второй раз = та же ситуация).

---

### HTTP коды ответов

```
1xx — Информационные
  100 Continue
  101 Switching Protocols (WebSocket upgrade)

2xx — Успех
  200 OK
  201 Created
  204 No Content

3xx — Редиректы
  301 Moved Permanently
  302 Found (временный)
  304 Not Modified (кэш актуален)

4xx — Ошибка клиента
  400 Bad Request (неверный формат)
  401 Unauthorized (нет авторизации)
  403 Forbidden (нет доступа)
  404 Not Found
  422 Unprocessable Entity (ошибка валидации)
  429 Too Many Requests (rate limit)

5xx — Ошибка сервера
  500 Internal Server Error
  502 Bad Gateway
  503 Service Unavailable
  504 Gateway Timeout
```

---

### HTTP в Python

```python
import httpx  # или requests

# GET запрос
response = httpx.get(
    "https://httpbin.org/get",
    params={"key": "value"},
    headers={"Accept": "application/json"},
    timeout=10.0,
)

print(response.status_code)   # 200
print(response.headers)       # заголовки
print(response.json())        # тело как dict

# POST запрос
response = httpx.post(
    "https://httpbin.org/post",
    json={"name": "Alice", "age": 30},  # автоматически Content-Type: application/json
)

# PUT, PATCH, DELETE
httpx.put("https://api.example.com/users/1", json={"name": "Bob"})
httpx.patch("https://api.example.com/users/1", json={"age": 31})
httpx.delete("https://api.example.com/users/1")
```

---

### Простой HTTP сервер

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/api/hello":
            self._respond(200, {"message": "Hello, World!"})
        else:
            self._respond(404, {"error": "Not found"})
    
    def do_POST(self):
        content_length = int(self.headers.get("Content-Length", 0))
        body = self.rfile.read(content_length)
        data = json.loads(body)
        
        self._respond(201, {"created": True, "data": data})
    
    def _respond(self, status: int, data: dict):
        body = json.dumps(data).encode()
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", len(body))
        self.end_headers()
        self.wfile.write(body)
    
    def log_message(self, format, *args):
        pass  # отключить логи

server = HTTPServer(("127.0.0.1", 8000), Handler)
print("Сервер на http://127.0.0.1:8000")
# server.serve_forever()
```

---

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Версия | Транспорт | Мультиплексирование | Сжатие заголовков | Head-of-line blocking |
|--------|-----------|--------------------|-----------------|-----------------------|
| HTTP/1.1 | TCP | Нет (1 запрос/соединение) | Нет | Да |
| HTTP/2 | TCP | Да (много потоков) | HPACK | Частично |
| HTTP/3 | QUIC (UDP) | Да | QPACK | Нет |

**HTTP/2 multiplexing** — несколько запросов одновременно в одном TCP соединении:

```
HTTP/1.1:                HTTP/2:
Req1 → ─────────────    Req1 → ──────────
← Resp1                  Req2 → ──────
Req2 → ─────────────    Req3 → ──────────────
← Resp2                 ← Resp2
                        ← Resp1
                        ← Resp3
```

---

### Заголовки HTTP — важные

```python
# Запрос
headers_request = {
    "Host": "example.com",              # Обязателен в HTTP/1.1
    "Authorization": "Bearer <token>",  # Аутентификация
    "Content-Type": "application/json", # Тип тела запроса
    "Accept": "application/json",       # Что хочу получить
    "Accept-Encoding": "gzip, br",      # Принимаю сжатые ответы
    "User-Agent": "MyApp/1.0",          # Кто я
    "Cookie": "session=abc123",         # Куки
}

# Ответ
headers_response = {
    "Content-Type": "application/json; charset=utf-8",
    "Content-Length": "156",
    "Cache-Control": "max-age=3600",    # Кэшировать 1 час
    "Set-Cookie": "session=abc123; HttpOnly; Secure",
    "Access-Control-Allow-Origin": "*", # CORS
    "X-Request-Id": "uuid-here",        # Трейсинг
}
```

---

## DNS — Domain Name System

### Что такое DNS

DNS — распределённая система, переводящая **имена** (google.com) в **IP адреса** (142.250.185.46).

Без DNS пришлось бы запоминать IP всех сайтов.

---

### Иерархия DNS

```
.                            ← Root (корень)
├── com.
│   ├── google.com.
│   │   └── www.google.com.  ← Полное доменное имя (FQDN)
│   └── example.com.
├── org.
│   └── wikipedia.org.
└── ru.
    └── yandex.ru.
```

---

### Как работает DNS запрос

```
Браузер                DNS Resolver         Root NS      .com NS    google NS
  │                    (ISP/8.8.8.8)           │            │           │
  │─ "www.google.com?" ►│                       │            │           │
  │                     │─ "www.google.com?" ──►│            │           │
  │                     │◄─ "спроси .com NS" ───│            │           │
  │                     │─ "www.google.com?" ───────────────►│           │
  │                     │◄─ "спроси google NS" ─────────────│           │
  │                     │─ "www.google.com?" ────────────────────────────►│
  │                     │◄─ "142.250.185.46" ─────────────────────────────│
  │◄─ "142.250.185.46" ─│                       │            │           │
  │                     │ [кэш на TTL]           │            │           │
```

---

### DNS записи

| Тип | Назначение | Пример |
|-----|-----------|--------|
| A | Имя → IPv4 | `google.com → 142.250.185.46` |
| AAAA | Имя → IPv6 | `google.com → 2607:f8b0::` |
| CNAME | Псевдоним | `www.example.com → example.com` |
| MX | Почтовый сервер | `example.com → mail.example.com` |
| TXT | Произвольный текст | SPF, DKIM записи |
| NS | Сервер имён | Делегирование зоны |
| PTR | IP → Имя (обратный DNS) | `46.185.250.142 → google.com` |
| SOA | Start of Authority | Метаданные зоны |

```python
import dns.resolver  # pip install dnspython

def dns_lookup(domain: str):
    resolver = dns.resolver.Resolver()
    resolver.nameservers = ["8.8.8.8"]
    
    # A запись
    a_records = resolver.resolve(domain, "A")
    print(f"A: {[r.address for r in a_records]}")
    
    # MX запись
    mx_records = resolver.resolve(domain, "MX")
    print(f"MX: {[(r.preference, str(r.exchange)) for r in mx_records]}")
    
    # TXT запись
    txt_records = resolver.resolve(domain, "TXT")
    print(f"TXT: {[r.to_text() for r in txt_records]}")

dns_lookup("google.com")
```

---

### DNS протокол на уровне байт

DNS использует **UDP порт 53** (для маленьких ответов) или TCP (для больших).

```python
import socket
import struct

def dns_query(domain: str, dns_server: str = "8.8.8.8") -> bytes:
    # Формируем DNS запрос вручную
    transaction_id = 0x1234
    flags = 0x0100        # Стандартный запрос, рекурсия желательна
    questions = 1
    answer_rrs = 0
    authority_rrs = 0
    additional_rrs = 0
    
    header = struct.pack("!HHHHHH",
        transaction_id, flags, questions,
        answer_rrs, authority_rrs, additional_rrs
    )
    
    # Кодируем доменное имя: google.com → \x06google\x03com\x00
    qname = b""
    for part in domain.split("."):
        qname += bytes([len(part)]) + part.encode()
    qname += b"\x00"
    
    qtype = 1   # A запись
    qclass = 1  # IN (интернет)
    question = qname + struct.pack("!HH", qtype, qclass)
    
    packet = header + question
    
    # Отправляем через UDP
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(3)
    sock.sendto(packet, (dns_server, 53))
    response, _ = sock.recvfrom(512)
    sock.close()
    
    return response

# response = dns_query("google.com")
```

---

## FTP — File Transfer Protocol

### Что такое FTP

FTP — протокол передачи файлов. Один из старейших (1971 год). Работает на TCP.

Особенность: использует **два TCP соединения**:
- **Командный канал** (порт 21) — управление, команды
- **Данных канал** (порт 20 или случайный) — передача файлов

Два режима:
- **Active mode** — сервер сам подключается к клиенту (проблемы с firewall)
- **Passive mode** — клиент подключается к серверу (стандарт сегодня)

---

### FTP команды

```
USER alice          → Имя пользователя
PASS secret123      → Пароль
LIST                → Список файлов
RETR file.txt       → Скачать файл
STOR file.txt       → Загрузить файл
CWD /path           → Сменить директорию
MKD dirname         → Создать директорию
QUIT                → Выйти
PASV                → Переключиться в пассивный режим
```

```python
import ftplib

# FTP клиент
ftp = ftplib.FTP("ftp.example.com")
ftp.login(user="alice", passwd="secret")

# Список файлов
files = ftp.nlst()
print(files)

# Скачать файл
with open("local_file.txt", "wb") as f:
    ftp.retrbinary("RETR remote_file.txt", f.write)

# Загрузить файл
with open("local_file.txt", "rb") as f:
    ftp.storbinary("STOR remote_file.txt", f)

ftp.quit()
```

**Важно**: FTP передаёт логин и пароль в открытом виде. Используйте **SFTP** (SSH) или **FTPS** (FTP + TLS) для безопасности.

---

## SMTP — Simple Mail Transfer Protocol

### Что такое SMTP

SMTP — протокол отправки email. Работает на TCP порт **25** (сервер-сервер) или **587** (клиент-сервер с авторизацией).

**SMTP** — только отправка. Для получения: **IMAP** (порт 993) или **POP3** (порт 995).

---

### Диалог SMTP

```
Клиент                        Сервер (SMTP)
  │                               │
  │◄──────── 220 smtp.gmail.com ──│  Приветствие
  │                               │
  │──── EHLO myapp.example.com ──►│  Представляемся
  │◄──────── 250-OK               │
  │◄──────── 250 AUTH PLAIN LOGIN │
  │                               │
  │──── AUTH LOGIN ──────────────►│  Авторизация
  │◄──────── 334 (base64 prompt) ─│
  │──── dXNlcm5hbWU= ────────────►│  Username (base64)
  │◄──────── 334                  │
  │──── cGFzc3dvcmQ= ────────────►│  Password (base64)
  │◄──────── 235 Auth OK          │
  │                               │
  │──── MAIL FROM:<alice@ex.com> ►│  Отправитель
  │◄──────── 250 OK               │
  │                               │
  │──── RCPT TO:<bob@ex.com> ────►│  Получатель
  │◄──────── 250 OK               │
  │                               │
  │──── DATA ────────────────────►│  Начало тела письма
  │◄──────── 354 Send data        │
  │──── Subject: Hello            │
  │──── From: alice@ex.com        │
  │──── To: bob@ex.com            │
  │──── (пустая строка)           │
  │──── Hello Bob!                │
  │──── . ───────────────────────►│  Точка на отдельной строке = конец
  │◄──────── 250 Message queued   │
  │                               │
  │──── QUIT ────────────────────►│
  │◄──────── 221 Bye              │
```

```python
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

def send_email(
    smtp_host: str,
    smtp_port: int,
    username: str,
    password: str,
    to: str,
    subject: str,
    body: str,
):
    msg = MIMEMultipart()
    msg["From"] = username
    msg["To"] = to
    msg["Subject"] = subject
    msg.attach(MIMEText(body, "plain", "utf-8"))
    
    # Используем STARTTLS (порт 587)
    with smtplib.SMTP(smtp_host, smtp_port) as server:
        server.ehlo()
        server.starttls()           # Включаем шифрование
        server.login(username, password)
        server.send_message(msg)
    
    print(f"Письмо отправлено на {to}")

# Пример для Gmail (нужен App Password)
# send_email(
#     "smtp.gmail.com", 587,
#     "alice@gmail.com", "app_password",
#     "bob@example.com",
#     "Привет!",
#     "Тестовое письмо от Python"
# )
```

---

## SSH — Secure Shell

### Что такое SSH

SSH — протокол безопасного удалённого доступа. Шифрует весь трафик.

Используется для:
- Удалённого управления серверами
- Туннелирования (проброс портов)
- Передачи файлов (SFTP, SCP)
- Git через SSH

```python
import paramiko  # pip install paramiko

def ssh_execute(host: str, user: str, key_path: str, command: str) -> str:
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    # Подключение по ключу (рекомендуется)
    client.connect(host, username=user, key_filename=key_path)
    
    stdin, stdout, stderr = client.exec_command(command)
    result = stdout.read().decode()
    error = stderr.read().decode()
    
    client.close()
    
    if error:
        raise RuntimeError(f"Ошибка: {error}")
    return result

# output = ssh_execute("192.168.1.10", "ubuntu", "~/.ssh/id_rsa", "ls -la")

# SFTP — передача файлов
def sftp_upload(host: str, user: str, key_path: str, local: str, remote: str):
    client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect(host, username=user, key_filename=key_path)
    
    sftp = client.open_sftp()
    sftp.put(local, remote)
    sftp.close()
    client.close()
```

---

## Сравнение прикладных протоколов

| Протокол | Порт | Транспорт | Шифрование | Назначение |
|---------|------|-----------|-----------|-----------|
| HTTP | 80 | TCP | Нет | Веб |
| HTTPS | 443 | TCP/QUIC | TLS | Веб + безопасность |
| FTP | 21 | TCP | Нет | Файлы |
| FTPS | 990 | TCP | TLS | Файлы + безопасность |
| SFTP | 22 | TCP | SSH | Файлы через SSH |
| SMTP | 25/587 | TCP | TLS (STARTTLS) | Отправка email |
| IMAP | 993 | TCP | TLS | Получение email |
| SSH | 22 | TCP | Встроенное | Удалённый доступ |
| DNS | 53 | UDP/TCP | DoH/DoT | Разрешение имён |
