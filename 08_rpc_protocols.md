# RPC протоколы: REST, gRPC, GraphQL, JSON-RPC (TICODE - https://ticode.online/)

## Что такое RPC

**RPC (Remote Procedure Call)** — парадигма, при которой вы вызываете функцию на удалённом сервере так, как будто она локальная.

```python
# Без RPC: нужно вручную делать HTTP запрос
response = httpx.get("http://user-service/api/users/42")
user = response.json()

# С RPC: выглядит как обычный вызов функции
user = user_service.get_user(id=42)
```

---

## REST — Representational State Transfer

### Что такое REST

REST — архитектурный стиль для построения API поверх HTTP. Не протокол, а набор ограничений (constraints).

**Принципы REST:**
1. **Ресурсы** — всё моделируется как ресурс с URL
2. **Методы HTTP** — GET/POST/PUT/PATCH/DELETE для операций
3. **Stateless** — сервер не хранит состояние клиента
4. **Представления** — ресурс может быть в JSON, XML, HTML

---

### REST дизайн URL

```
Ресурс: Users (пользователи)

GET    /users           → список всех пользователей
POST   /users           → создать пользователя
GET    /users/42        → получить пользователя с id=42
PUT    /users/42        → полностью обновить пользователя 42
PATCH  /users/42        → частично обновить пользователя 42
DELETE /users/42        → удалить пользователя 42

Вложенные ресурсы:
GET    /users/42/posts  → посты пользователя 42
POST   /users/42/posts  → создать пост у пользователя 42
GET    /users/42/posts/7 → конкретный пост

Фильтрация/пагинация через query params:
GET    /users?page=2&limit=20&sort=name&active=true
```

---

### REST сервер на Python (FastAPI)

```python
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# In-memory база данных
db: dict[int, dict] = {}
next_id = 1

class UserCreate(BaseModel):
    name: str
    email: str
    age: Optional[int] = None

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None
    age: Optional[int] = None

@app.get("/users")
def list_users(
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100),
    active: Optional[bool] = None,
):
    users = list(db.values())
    if active is not None:
        users = [u for u in users if u.get("active") == active]
    start = (page - 1) * limit
    return {"users": users[start:start + limit], "total": len(users)}

@app.get("/users/{user_id}")
def get_user(user_id: int):
    if user_id not in db:
        raise HTTPException(status_code=404, detail="User not found")
    return db[user_id]

@app.post("/users", status_code=201)
def create_user(user: UserCreate):
    global next_id
    new_user = {"id": next_id, **user.model_dump(), "active": True}
    db[next_id] = new_user
    next_id += 1
    return new_user

@app.patch("/users/{user_id}")
def update_user(user_id: int, update: UserUpdate):
    if user_id not in db:
        raise HTTPException(status_code=404, detail="User not found")
    
    patch_data = {k: v for k, v in update.model_dump().items() if v is not None}
    db[user_id].update(patch_data)
    return db[user_id]

@app.delete("/users/{user_id}", status_code=204)
def delete_user(user_id: int):
    if user_id not in db:
        raise HTTPException(status_code=404, detail="User not found")
    del db[user_id]

# uvicorn main:app --reload
```

---

### REST клиент

```python
import httpx

BASE_URL = "http://127.0.0.1:8000"

# CRUD операции
def create_user(name: str, email: str) -> dict:
    r = httpx.post(f"{BASE_URL}/users", json={"name": name, "email": email})
    r.raise_for_status()
    return r.json()

def get_user(user_id: int) -> dict:
    r = httpx.get(f"{BASE_URL}/users/{user_id}")
    r.raise_for_status()
    return r.json()

def list_users(page: int = 1, limit: int = 20) -> dict:
    r = httpx.get(f"{BASE_URL}/users", params={"page": page, "limit": limit})
    r.raise_for_status()
    return r.json()

def update_user(user_id: int, **fields) -> dict:
    r = httpx.patch(f"{BASE_URL}/users/{user_id}", json=fields)
    r.raise_for_status()
    return r.json()

def delete_user(user_id: int):
    r = httpx.delete(f"{BASE_URL}/users/{user_id}")
    r.raise_for_status()
```

---

## gRPC — Google Remote Procedure Call

### Что такое gRPC

gRPC — высокопроизводительный RPC фреймворк от Google. Использует:
- **Protobuf** — сериализация данных
- **HTTP/2** — транспорт (мультиплексирование, server push)

Преимущества над REST:
- Строгая схема (`.proto` файл)
- Бинарный, компактный формат
- Встроенный стриминг (server/client/bidirectional)
- Кодогенерация для разных языков

---

### .proto файл для gRPC

```protobuf
syntax = "proto3";

package users;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (UserList);
    rpc CreateUser(CreateUserRequest) returns (User);
    
    // Server-side streaming: сервер шлёт поток ответов
    rpc WatchUsers(WatchUsersRequest) returns (stream User);
    
    // Client-side streaming: клиент шлёт поток запросов
    rpc BulkCreateUsers(stream CreateUserRequest) returns (BulkResult);
    
    // Bidirectional streaming
    rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
    bool active = 4;
}

message GetUserRequest {
    int32 id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 limit = 2;
}

message UserList {
    repeated User users = 1;
    int32 total = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}
```

---

### gRPC сервер на Python

```python
# pip install grpcio grpcio-tools
# python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. users.proto
# Это генерирует users_pb2.py и users_pb2_grpc.py

import grpc
from concurrent import futures
# import users_pb2
# import users_pb2_grpc

# Демонстрация структуры (без реального .proto файла):

class UserServiceImpl:  # наследуем от users_pb2_grpc.UserServiceServicer
    def __init__(self):
        self._db = {}
        self._next_id = 1
    
    def GetUser(self, request, context):
        user_id = request.id
        if user_id not in self._db:
            context.abort(grpc.StatusCode.NOT_FOUND, f"User {user_id} not found")
        return self._db[user_id]
    
    def CreateUser(self, request, context):
        user = {
            "id": self._next_id,
            "name": request.name,
            "email": request.email,
            "active": True,
        }
        self._db[self._next_id] = user
        self._next_id += 1
        return user
    
    def WatchUsers(self, request, context):
        # Server-side streaming
        import time
        for user in self._db.values():
            yield user
            time.sleep(0.1)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    # users_pb2_grpc.add_UserServiceServicer_to_server(UserServiceImpl(), server)
    server.add_insecure_port("[::]:50051")
    server.start()
    print("gRPC сервер запущен на порту 50051")
    server.wait_for_termination()
```

---

### gRPC vs REST

| | REST | gRPC |
|--|------|------|
| Протокол | HTTP/1.1 или HTTP/2 | HTTP/2 |
| Формат | JSON (текст) | Protobuf (бинарный) |
| Схема | Опционально (OpenAPI) | Обязательно (.proto) |
| Стриминг | Нет (SSE — ограниченно) | Да (все виды) |
| Читаемость | Высокая | Нет без инструментов |
| Скорость | Средняя | Высокая |
| Браузер | Да | Нет (нужен grpc-web) |

---

## GraphQL

### Что такое GraphQL

GraphQL — язык запросов для API, разработанный Facebook. Клиент сам описывает что именно хочет получить — никаких over-fetching (лишних данных) и under-fetching (нескольких запросов).

**Проблема REST:**
```
GET /users/42  → {"id": 42, "name": "Alice", "email": "...", "address": {...}, "settings": {...}}
# Нужно только name, но получили всё
```

**GraphQL решение:**
```graphql
query {
  user(id: 42) {
    name         # Только то, что нужно
    email
  }
}
```

---

### GraphQL запросы

```graphql
# Query — чтение данных
query GetUser {
  user(id: 42) {
    id
    name
    email
    posts {
      title
      createdAt
    }
  }
}

# Mutation — изменение данных
mutation CreateUser {
  createUser(input: {name: "Bob", email: "bob@example.com"}) {
    id
    name
  }
}

# Subscription — подписка на изменения (через WebSocket)
subscription OnNewPost {
  newPost {
    id
    title
    author {
      name
    }
  }
}

# Фрагменты — переиспользование полей
fragment UserFields on User {
  id
  name
  email
}

query {
  user(id: 42) { ...UserFields }
  me { ...UserFields }
}
```

---

### GraphQL сервер на Python (Strawberry)

```python
import strawberry
from typing import Optional

# pip install strawberry-graphql

@strawberry.type
class Post:
    id: int
    title: str
    content: str

@strawberry.type
class User:
    id: int
    name: str
    email: str
    
    @strawberry.field
    def posts(self) -> list[Post]:
        # Загружаем посты пользователя
        return [Post(id=1, title="Hello", content="World")]

@strawberry.type
class Query:
    @strawberry.field
    def user(self, id: int) -> Optional[User]:
        # Здесь обращение к базе данных
        return User(id=id, name="Alice", email="alice@example.com")
    
    @strawberry.field
    def users(self) -> list[User]:
        return [User(id=1, name="Alice", email="alice@example.com")]

@strawberry.input
class CreateUserInput:
    name: str
    email: str

@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_user(self, input: CreateUserInput) -> User:
        # Сохранить в базу
        return User(id=100, name=input.name, email=input.email)

schema = strawberry.Schema(query=Query, mutation=Mutation)

# FastAPI интеграция:
# from strawberry.fastapi import GraphQLRouter
# graphql_app = GraphQLRouter(schema)
# app.include_router(graphql_app, prefix="/graphql")
```

---

### GraphQL клиент

```python
import httpx

def graphql_query(url: str, query: str, variables: dict = None) -> dict:
    payload = {"query": query}
    if variables:
        payload["variables"] = variables
    
    response = httpx.post(url, json=payload)
    data = response.json()
    
    if "errors" in data:
        raise ValueError(f"GraphQL ошибки: {data['errors']}")
    
    return data["data"]

# Пример использования
result = graphql_query(
    "http://localhost:8000/graphql",
    """
    query GetUser($id: Int!) {
        user(id: $id) {
            id
            name
            email
        }
    }
    """,
    variables={"id": 42},
)
print(result["user"]["name"])
```

---

## JSON-RPC

### Что такое JSON-RPC

JSON-RPC — простой протокол RPC поверх JSON. Вместо REST-ресурсов — вызовы методов.

```json
// Запрос
{
  "jsonrpc": "2.0",
  "method": "users.get",
  "params": {"id": 42},
  "id": 1
}

// Успешный ответ
{
  "jsonrpc": "2.0",
  "result": {"id": 42, "name": "Alice"},
  "id": 1
}

// Ошибка
{
  "jsonrpc": "2.0",
  "error": {"code": -32601, "message": "Method not found"},
  "id": 1
}

// Уведомление (без id, нет ответа)
{
  "jsonrpc": "2.0",
  "method": "log.event",
  "params": {"type": "login", "user_id": 42}
}
```

```python
import json
import httpx

class JsonRpcClient:
    def __init__(self, url: str):
        self.url = url
        self._id = 0
    
    def call(self, method: str, **params) -> any:
        self._id += 1
        payload = {
            "jsonrpc": "2.0",
            "method": method,
            "params": params,
            "id": self._id,
        }
        response = httpx.post(self.url, json=payload)
        data = response.json()
        
        if "error" in data:
            raise RuntimeError(f"RPC Error: {data['error']}")
        return data["result"]

# client = JsonRpcClient("http://localhost:8000/rpc")
# user = client.call("users.get", id=42)
```

---

## Сравнение подходов

| | REST | gRPC | GraphQL | JSON-RPC |
|--|------|------|---------|---------|
| Транспорт | HTTP | HTTP/2 | HTTP | HTTP/WebSocket |
| Формат | JSON | Protobuf | JSON | JSON |
| Схема | OpenAPI | .proto | SDL | Нет |
| Стриминг | Нет | Да | Subscriptions | Нет |
| Сложность | Низкая | Высокая | Средняя | Низкая |
| Подходит для | Публичные API | Микросервисы | Сложные запросы | Простое RPC |

```
Публичный API (мобильные/веб клиенты)?    → REST
Межсервисное взаимодействие?              → gRPC
Гибкие клиентские запросы (BFF)?          → GraphQL
Простое внутреннее RPC?                   → JSON-RPC
```
