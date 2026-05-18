# Полное руководство по протоколам от TICODE (https://ticode.online/)

Это руководство охватывает всё, что нужно знать о протоколах — от базовых понятий до практической реализации.

## Структура руководства

| Файл | Тема |
|------|------|
| [01_what_is_protocol.md](01_what_is_protocol.md) | Что такое протокол, основные понятия и термины |
| [02_osi_model.md](02_osi_model.md) | Модель OSI и стек TCP/IP — как устроена сеть |
| [03_network_protocols.md](03_network_protocols.md) | Сетевые протоколы: IP, TCP, UDP, ICMP |
| [04_application_protocols.md](04_application_protocols.md) | Прикладные протоколы: HTTP, HTTPS, FTP, DNS, SMTP |
| [05_security_protocols.md](05_security_protocols.md) | Протоколы безопасности: TLS/SSL, SSH, OAuth, JWT |
| [06_messaging_protocols.md](06_messaging_protocols.md) | Протоколы обмена сообщениями: WebSocket, MQTT, AMQP |
| [07_serialization_protocols.md](07_serialization_protocols.md) | Протоколы сериализации: JSON, XML, Protobuf, MessagePack |
| [08_rpc_protocols.md](08_rpc_protocols.md) | RPC протоколы: REST, gRPC, GraphQL, JSON-RPC |
| [09_database_protocols.md](09_database_protocols.md) | Протоколы БД: PostgreSQL wire protocol, Redis RESP |
| [10_implementation_guide.md](10_implementation_guide.md) | Как реализовать свой протокол с нуля на Python |

## Рекомендуемый порядок изучения

```
1. Что такое протокол (01)
        ↓
2. Модель OSI — понять слои (02)
        ↓
3. Сетевые протоколы — фундамент (03)
        ↓
4. Прикладные протоколы — повседневные (04)
        ↓
5. Безопасность — как защищаются данные (05)
        ↓
6. Специализированные: messaging (06), RPC (08)
        ↓
7. Форматы данных — сериализация (07)
        ↓
8. Протоколы БД (09)
        ↓
9. Реализация своего протокола (10)
```

## Ключевые концепции, которые встретятся везде

- **Handshake** — установка соединения между двумя сторонами
- **Frame / Packet / Segment** — единица данных на разных уровнях
- **Stateful vs Stateless** — сохраняет ли протокол состояние между запросами
- **Синхронный vs асинхронный** — блокирует ли вызов выполнение
- **Кодирование** — как байты превращаются в смысл (big-endian, UTF-8 и т.д.)
