# Протоколы безопасности: TLS/SSL, SSH, OAuth, JWT (TICODE - https://ticode.online/)

## TLS/SSL — Transport Layer Security

### Что такое TLS

TLS (ранее SSL) — протокол шифрования транспортного уровня. Обеспечивает:
- **Конфиденциальность** — данные зашифрованы, посредник не прочтёт
- **Целостность** — данные не изменены в пути (MAC/HMAC)
- **Аутентификацию** — сервер подтверждает свою личность сертификатом

HTTPS = HTTP + TLS. SSH, SMTPS, FTPS тоже используют TLS.

---

### TLS Handshake (TLS 1.3)

```
Клиент                                  Сервер
  │                                       │
  │──── ClientHello ─────────────────────►│
  │  [поддерживаемые cipher suites,       │
  │   версии TLS, случайные данные]       │
  │                                       │
  │◄─── ServerHello ──────────────────────│
  │  [выбранный cipher suite,             │
  │   случайные данные,                   │
  │   сертификат сервера,                 │
  │   публичный ключ для обмена]          │
  │                                       │
  │   [Клиент проверяет сертификат:       │
  │    - Подписан доверенным CA?          │
  │    - Срок действия?                   │
  │    - Совпадает домен?]                │
  │                                       │
  │──── Finished (зашифровано) ──────────►│
  │  [подтверждение handshake]            │
  │                                       │
  │◄─── Finished (зашифровано) ───────────│
  │                                       │
  │   [Симметричный ключ согласован]      │
  │   [Все данные теперь шифруются]       │
  │                                       │
  │══ Зашифрованные данные (HTTP) ════════│
```

TLS 1.3 — только **1 round-trip** (1-RTT) для установки соединения, в отличие от 2-RTT в TLS 1.2.

---

### Сертификаты X.509

Сертификат — это файл, подписанный удостоверяющим центром (CA), подтверждающий, что публичный ключ принадлежит домену.

```
Сертификат example.com:
┌─────────────────────────────────────────┐
│ Subject: CN=example.com                 │
│ Issuer: Let's Encrypt Authority X3      │
│ Valid: 2024-01-01 — 2024-04-01          │
│ Public Key: (RSA 2048 bit)              │
│ SAN: example.com, www.example.com       │
│ Signature: (подпись Let's Encrypt)      │
└─────────────────────────────────────────┘
```

Цепочка доверия:
```
Root CA (встроен в ОС)
  └── Intermediate CA
        └── example.com certificate
```

```python
import ssl
import socket

def get_certificate(host: str, port: int = 443) -> dict:
    context = ssl.create_default_context()
    
    with socket.create_connection((host, port)) as sock:
        with context.wrap_socket(sock, server_hostname=host) as ssock:
            cert = ssock.getpeercert()
            return {
                "subject": dict(x[0] for x in cert["subject"]),
                "issuer": dict(x[0] for x in cert["issuer"]),
                "version": cert["version"],
                "serial": cert["serialNumber"],
                "not_before": cert["notBefore"],
                "not_after": cert["notAfter"],
                "san": cert.get("subjectAltName", []),
            }

info = get_certificate("google.com")
print(info)
```

---

### TLS в Python

```python
import ssl
import socket

# Создание контекста (настройки TLS)
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.load_verify_locations("/etc/ssl/certs/ca-certificates.crt")

# Подключение с TLS
with socket.create_connection(("example.com", 443)) as sock:
    with context.wrap_socket(sock, server_hostname="example.com") as tls_sock:
        # Теперь всё зашифровано
        tls_sock.send(b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n")
        response = tls_sock.recv(4096)

# TLS сервер с самоподписанным сертификатом
import subprocess

# Генерация сертификата (нужен openssl)
# subprocess.run([
#     "openssl", "req", "-x509", "-newkey", "rsa:4096",
#     "-keyout", "key.pem", "-out", "cert.pem",
#     "-days", "365", "-nodes", "-subj", "/CN=localhost"
# ])

server_context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
server_context.load_cert_chain("cert.pem", "key.pem")

with socket.socket() as server_sock:
    server_sock.bind(("127.0.0.1", 4443))
    server_sock.listen(1)
    with server_context.wrap_socket(server_sock, server_side=True) as tls_server:
        conn, addr = tls_server.accept()
        data = conn.recv(1024)
```

---

### Cipher Suites

Cipher Suite — набор алгоритмов для TLS соединения.

```
TLS_AES_128_GCM_SHA256
│   │   │   │   └── Хэш для HMAC (SHA-256)
│   │   │   └────── Режим шифрования (GCM)
│   │   └────────── Алгоритм шифрования (AES-128)
│   └────────────── (нет — TLS 1.3 использует ECDHE для key exchange)
└────────────────── Протокол
```

---

## Асимметричная и симметричная криптография

### Асимметричная (публичный/приватный ключ)

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes, serialization

# Генерация ключевой пары
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# Шифрование публичным ключом (только приватный может расшифровать)
message = b"Secret message"
encrypted = public_key.encrypt(
    message,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)

# Расшифровка приватным ключом
decrypted = private_key.decrypt(
    encrypted,
    padding.OAEP(
        mgf=padding.MGF1(algorithm=hashes.SHA256()),
        algorithm=hashes.SHA256(),
        label=None
    )
)
print(decrypted)  # b"Secret message"
```

### Симметричная (один ключ)

```python
from cryptography.fernet import Fernet

# Генерация ключа
key = Fernet.generate_key()
f = Fernet(key)

# Шифрование
message = b"Secret message"
token = f.encrypt(message)
print(token)  # зашифрованные байты

# Расшифровка
decrypted = f.decrypt(token)
print(decrypted)  # b"Secret message"
```

**TLS использует оба**: асимметричная криптография для **согласования ключа** (handshake), симметричная — для **передачи данных** (быстрее).

---

## OAuth 2.0

### Что такое OAuth 2.0

OAuth 2.0 — протокол **авторизации** (не аутентификации!). Позволяет сервису получить доступ к ресурсам пользователя на другом сервисе **без передачи пароля**.

Пример: "Войти через Google" — ваш пароль остаётся у Google, приложение получает только токен.

---

### Роли в OAuth

- **Resource Owner** — пользователь (владелец данных)
- **Client** — приложение, которое хочет доступ
- **Authorization Server** — выдаёт токены (Google Auth)
- **Resource Server** — хранит данные (Google Drive API)

---

### Authorization Code Flow (основной для веб)

```
Пользователь      Приложение           Google Auth         Google API
     │                │                    │                    │
     │─ "Войти" ──────►│                   │                    │
     │                 │                   │                    │
     │                 │─ redirect ──────► │                    │
     │                 │  (client_id,      │                    │
     │                 │   scope,          │                    │
     │                 │   redirect_uri)   │                    │
     │                 │                   │                    │
     │◄────────────────────────────────────│  Форма логина      │
     │─ Вводит логин/пароль ──────────────►│                    │
     │◄────────────────────────────────────│  Разрешить?        │
     │─ Разрешаю ─────────────────────────►│                    │
     │                 │                   │                    │
     │◄────── redirect ─── code=ABC ───────│                    │
     │        (на redirect_uri)            │                    │
     │                 │                   │                    │
     │─── code=ABC ───►│                   │                    │
     │                 │─ code + secret ──►│                    │
     │                 │◄── access_token ──│                    │
     │                 │                   │                    │
     │                 │─ Bearer token ─────────────────────────►│
     │                 │◄── данные пользователя ─────────────────│
```

---

### OAuth в Python (пример с httpx)

```python
import httpx
import secrets
import urllib.parse

CLIENT_ID = "your_client_id"
CLIENT_SECRET = "your_client_secret"
REDIRECT_URI = "http://localhost:8080/callback"
AUTH_URL = "https://accounts.google.com/o/oauth2/v2/auth"
TOKEN_URL = "https://oauth2.googleapis.com/token"

def get_authorization_url() -> tuple[str, str]:
    state = secrets.token_urlsafe(32)  # CSRF защита
    
    params = {
        "client_id": CLIENT_ID,
        "redirect_uri": REDIRECT_URI,
        "response_type": "code",
        "scope": "openid email profile",
        "state": state,
    }
    
    url = AUTH_URL + "?" + urllib.parse.urlencode(params)
    return url, state

def exchange_code_for_token(code: str) -> dict:
    response = httpx.post(TOKEN_URL, data={
        "code": code,
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "redirect_uri": REDIRECT_URI,
        "grant_type": "authorization_code",
    })
    return response.json()
    # {"access_token": "...", "token_type": "Bearer",
    #  "expires_in": 3600, "refresh_token": "..."}

def refresh_access_token(refresh_token: str) -> dict:
    response = httpx.post(TOKEN_URL, data={
        "refresh_token": refresh_token,
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET,
        "grant_type": "refresh_token",
    })
    return response.json()
```

---

### Client Credentials Flow (для сервисов)

```python
# Когда нет пользователя — сервис-сервис
def get_service_token(client_id: str, client_secret: str, token_url: str) -> str:
    response = httpx.post(token_url, data={
        "grant_type": "client_credentials",
        "client_id": client_id,
        "client_secret": client_secret,
        "scope": "api.read",
    })
    data = response.json()
    return data["access_token"]
```

---

## JWT — JSON Web Token

### Что такое JWT

JWT — компактный, самодостаточный токен для передачи информации между сторонами. Содержит **claims** (утверждения) и подписан.

Структура: `header.payload.signature` (base64url)

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  ← header
.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIiwiaWF0IjoxNTE2MjM5MDIyfQ  ← payload
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← signature
```

---

### Структура JWT

```python
import json
import base64

# Декодировать JWT (без проверки подписи)
def decode_jwt_payload(token: str) -> dict:
    parts = token.split(".")
    # base64url → base64 (добавляем padding)
    payload = parts[1] + "=" * (4 - len(parts[1]) % 4)
    return json.loads(base64.urlsafe_b64decode(payload))

token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIiwiaWF0IjoxNTE2MjM5MDIyfQ.sig"
print(decode_jwt_payload(token))
# {'sub': '1234567890', 'name': 'Alice', 'iat': 1516239022}
```

### Создание и проверка JWT

```python
import jwt  # pip install PyJWT
from datetime import datetime, timedelta, UTC

SECRET = "super_secret_key"

def create_jwt(user_id: int, username: str) -> str:
    payload = {
        "sub": str(user_id),           # Subject (кому выдан)
        "name": username,
        "iat": datetime.now(UTC),      # Issued at
        "exp": datetime.now(UTC) + timedelta(hours=1),  # Expiration
        "iss": "myapp.example.com",    # Issuer
    }
    return jwt.encode(payload, SECRET, algorithm="HS256")

def verify_jwt(token: str) -> dict:
    try:
        payload = jwt.decode(
            token, SECRET,
            algorithms=["HS256"],
            options={"verify_exp": True},
        )
        return payload
    except jwt.ExpiredSignatureError:
        raise ValueError("Токен истёк")
    except jwt.InvalidTokenError as e:
        raise ValueError(f"Неверный токен: {e}")

token = create_jwt(42, "Alice")
print(token)

payload = verify_jwt(token)
print(payload)  # {'sub': '42', 'name': 'Alice', ...}
```

---

### JWT vs Session Tokens

| | JWT | Session |
|--|-----|---------|
| Хранение состояния | Без состояния (данные в токене) | На сервере (в БД/Redis) |
| Масштабирование | Легко (нет обращений к БД) | Нужна синхронизация |
| Отзыв токена | Сложно (нужен blacklist) | Просто (удалить из БД) |
| Размер | Больше (~200-500 байт) | Маленький (случайная строка) |

---

## HMAC — проверка целостности

```python
import hmac
import hashlib
import secrets

def sign_message(message: bytes, key: bytes) -> bytes:
    return hmac.new(key, message, hashlib.sha256).digest()

def verify_message(message: bytes, signature: bytes, key: bytes) -> bool:
    expected = sign_message(message, key)
    # Константное время сравнения (защита от timing attacks)
    return hmac.compare_digest(signature, expected)

key = secrets.token_bytes(32)
message = b"Transfer $100 to Alice"

sig = sign_message(message, key)
print(verify_message(message, sig, key))  # True

# Если сообщение изменено
tampered = b"Transfer $1000 to Alice"
print(verify_message(tampered, sig, key))  # False
```

---

## Краткое резюме

| Протокол/стандарт | Задача | Где применяется |
|-------------------|--------|----------------|
| TLS 1.3 | Шифрование канала | HTTPS, SMTPS, любой TCP |
| X.509 | Сертификаты идентификации | TLS, подпись кода |
| OAuth 2.0 | Делегирование авторизации | "Войти через Google/GitHub" |
| JWT | Передача claims | API токены, SSO |
| HMAC | Проверка целостности | Подпись webhook, API ключи |
| RSA/ECDSA | Асимметричная криптография | TLS handshake, подписи |
| AES-GCM | Симметричное шифрование | TLS record layer |
