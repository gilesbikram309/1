# RabbitMQ Consumer — Полное руководство

> **Уровень:** Middle+ / Senior  
> **PHP:** 8.2  
> **Symfony:** 6.4  
> **RabbitMQ:** 3.x / 4.x  
> **Тема:** Понимание работы Consumer, архитектура, паттерны, Symfony Messenger

---

## Содержание

1. [Основы RabbitMQ: протокол AMQP](#1-основы-rabbitmq-протокол-amqp)
2. [Архитектура: Exchange, Queue, Binding](#2-архитектура-exchange-queue-binding)
3. [Что такое Consumer и как он работает](#3-что-такое-consumer-и-как-он-работает)
4. [Жизненный цикл сообщения](#4-жизненный-цикл-сообщения)
5. [Acknowledgement (ack/nack/reject)](#5-acknowledgement-acknackreject)
6. [Prefetch Count (QoS)](#6-prefetch-count-qos)
7. [Типы Exchange](#7-типы-exchange)
8. [Dead Letter Exchange (DLX)](#8-dead-letter-exchange-dlx)
9. [TTL, Expiration, Priority](#9-ttl-expiration-priority)
10. [Retry-стратегии](#10-retry-стратегии)
11. [Symfony Messenger + RabbitMQ](#11-symfony-messenger--rabbitmq)
12. [Конфигурация транспорта RabbitMQ в Symfony](#12-конфигурация-транспорта-rabbitmq-в-symfony)
13. [Создание Message и Handler](#13-создание-message-и-handler)
14. [Запуск и управление Consumer (Worker)](#14-запуск-и-управление-consumer-worker)
15. [Supervisor — управление воркерами в продакшене](#15-supervisor--управление-воркерами-в-продакшене)
16. [Failure Transport (обработка ошибок)](#16-failure-transport-обработка-ошибок)
17. [Retry механизм в Symfony Messenger](#17-retry-механизм-в-symfony-messenger)
18. [Multiple Consumers и приоритетные очереди](#18-multiple-consumers-и-приоритетные-очереди)
19. [Serialization сообщений](#19-serialization-сообщений)
20. [Graceful Shutdown и сигналы](#20-graceful-shutdown-и-сигналы)
21. [Memory Leaks и долгоживущие процессы](#21-memory-leaks-и-долгоживущие-процессы)
22. [Consumer в Docker и Kubernetes](#22-consumer-в-docker-и-kubernetes)
23. [Мониторинг RabbitMQ и Consumer'ов](#23-мониторинг-rabbitmq-и-consumerов)
24. [Идемпотентность обработки сообщений](#24-идемпотентность-обработки-сообщений)
25. [Гарантии доставки: At-Least-Once, At-Most-Once, Exactly-Once](#25-гарантии-доставки)
26. [Паттерны: CQRS, Event Sourcing, Saga](#26-паттерны-cqrs-event-sourcing-saga)
27. [Отложенные сообщения (Delayed Messages)](#27-отложенные-сообщения-delayed-messages)
28. [Тестирование Consumer'ов](#28-тестирование-consumerов)
29. [Типичные проблемы и решения](#29-типичные-проблемы-и-решения)
30. [Чек-лист для продакшена](#30-чек-лист-для-продакшена)
31. [Источники](#31-источники)
32. [Вопросы для самопроверки](#32-вопросы-для-самопроверки)

---

## 1. Основы RabbitMQ: протокол AMQP

### Что такое RabbitMQ

**RabbitMQ** — брокер сообщений (message broker), реализующий протокол **AMQP 0-9-1** (Advanced Message Queuing Protocol). Он принимает сообщения от producers и доставляет их consumers через очереди.

> 🔰 **Новичку:** «брокер сообщений» — это посредник, который принимает сообщения (events/команды/задачи) от одного сервиса (producer) и надёжно передаёт их другому (consumer). Аналог — почта: отправитель кладёт письмо в ящик, почтальон доставляет получателю. Если получатель «спит» или упал — письмо полежит в очереди и дождётся. Это делает систему **асинхронной** (producer не ждёт, пока consumer обработает) и **отказоустойчивой** (сервисы могут падать по очереди, не теряя данных).
>
> **AMQP 0-9-1** — это стандартизированный бинарный сетевой протокол (как HTTP для веба, только для сообщений). Его реализуют разные брокеры (RabbitMQ, ActiveMQ, Qpid) — то есть клиентский код можно переключать между ними. Важно не путать с **AMQP 1.0** — это фактически другой протокол. RabbitMQ изначально заточен под 0-9-1; поддержка 1.0 есть через плагин, но это редкий кейс.

### Зачем нужен Message Broker

| Проблема | Решение через RabbitMQ |
|----------|----------------------|
| Синхронная обработка тяжёлых задач замедляет отклик API | Consumer обрабатывает задачи в фоне |
| Сервис-получатель недоступен | Сообщения ждут в очереди |
| Пиковые нагрузки перегружают систему | Очередь буферизует, consumer обрабатывает в своём темпе |
| Тесная связанность сервисов | Pub/Sub через exchange — слабая связанность |
| Одно событие → много обработчиков | Fanout exchange: одно сообщение → N очередей |

### Зачем Consumer, а не просто Cron + Database?

Часто возникает вопрос: «Зачем мне RabbitMQ, если можно записать задачу в таблицу `jobs` и обрабатывать cron'ом каждую минуту?»

| Критерий | Cron + Database | RabbitMQ Consumer |
|----------|----------------|-------------------|
| **Задержка** | Минимум 1 минута (интервал cron) | Миллисекунды (push-доставка) |
| **Ресурсы** | Polling: запрос к БД каждую минуту, даже если задач нет | Event-driven: consumer ждёт, не тратит CPU |
| **Масштабирование** | Сложно: блокировки в БД, дублирование обработки | Просто: добавил 10 consumer'ов → 10x throughput |
| **Приоритеты** | Ручная реализация через ORDER BY | Встроенные priority queues |
| **Retry / DLX** | Ручная реализация (поле `attempts`, `next_try_at`) | Встроенный retry + Dead Letter Exchange |
| **Гарантия доставки** | Нужна ручная реализация (транзакции, статусы) | Встроенный ack/nack на уровне протокола |
| **Мониторинг** | Кастомные метрики | RabbitMQ Management UI, Prometheus exporter |
| **Отказоустойчивость** | Если БД упала — задачи не обработаются | Cluster + quorum queues / streams; classic mirrored queues удалены в RabbitMQ 4.0 |

> **Правило:** Если задержка до 1 минуты приемлема, задач мало (< 100/час), и инфраструктура проста — cron + DB допустим. Во всех остальных случаях — Message Broker.

### Участники AMQP

```
Producer → [Exchange] → Binding → [Queue] → Consumer
              │                      │
              │  Routing Key         │  Consumer Tag
              │  Binding Key         │  Prefetch Count
              │                      │  Ack/Nack
              ▼                      ▼
         Маршрутизация          Доставка и подтверждение
```

- **Connection** — TCP-соединение с RabbitMQ
- **Channel** — логический канал внутри connection (мультиплексирование)
- **Producer** — отправляет сообщения в exchange
- **Exchange** — маршрутизирует сообщения в очереди по правилам
- **Binding** — связь между exchange и queue (с routing key)
- **Queue** — буфер сообщений (FIFO)
- **Consumer** — получает и обрабатывает сообщения из queue

> 🔰 **Почему Channel, а не просто Connection?** TCP-соединение — дорогой ресурс (handshake, TLS, keep-alive). Один процесс может открыть одно connection и в нём мультиплексировать много channel'ов — каждый channel это «виртуальное соединение» со своим состоянием (QoS, consumer'ы, транзакции). Рекомендация из официальной документации: **не шарить канал между потоками/корутинами** и **не открывать connection на каждое сообщение** — это антипаттерн, убивающий производительность брокера (см. [RabbitMQ Connections](https://www.rabbitmq.com/docs/connections) и [Channels](https://www.rabbitmq.com/docs/channels)).
>
> ⚠️ **Важно:** producer НИКОГДА не публикует напрямую в queue — он всегда публикует в exchange, а тот по binding'у маршрутизирует в очереди. «Отправка напрямую в очередь» (`exchange=''`) — это синтаксический сахар: так используется **default exchange**, к которому каждая очередь автоматически привязана по её имени.

---

## 2. Архитектура: Exchange, Queue, Binding

### Полная схема

```
                     ┌──────────────────────────────────────────┐
                     │              RabbitMQ Broker              │
                     │                                          │
Producer ──msg──►    │  ┌──────────┐    ┌──────────────────┐   │
                     │  │ Exchange │───►│ Queue: orders     │───┼──► Consumer 1
Producer ──msg──►    │  │ (direct) │    └──────────────────┘   │
                     │  │          │    ┌──────────────────┐   │
                     │  │          │───►│ Queue: emails     │───┼──► Consumer 2
                     │  └──────────┘    └──────────────────┘   │
                     │                                          │
                     │  ┌──────────┐    ┌──────────────────┐   │
                     │  │ Exchange │───►│ Queue: audit_log  │───┼──► Consumer 3
                     │  │ (fanout) │    └──────────────────┘   │
                     │  │          │    ┌──────────────────┐   │
                     │  │          │───►│ Queue: analytics  │───┼──► Consumer 4
                     │  └──────────┘    └──────────────────┘   │
                     │                                          │
                     └──────────────────────────────────────────┘
```

### Свойства Queue

```
Queue properties:
├── name          — уникальное имя очереди
├── durable       — переживает перезапуск RabbitMQ (true для prod!)
├── exclusive     — монопольная (только одно соединение, удаляется при disconnect)
├── auto_delete   — удаляется, когда последний consumer отключается
├── arguments     — дополнительные параметры:
│   ├── x-message-ttl          — TTL сообщений в очереди (мс)
│   ├── x-expires              — TTL самой очереди (мс)
│   ├── x-max-length           — макс. кол-во сообщений
│   ├── x-max-length-bytes     — макс. размер очереди (байт)
│   ├── x-overflow             — политика при переполнении (drop-head, reject-publish)
│   ├── x-dead-letter-exchange — DLX для отклонённых сообщений
│   ├── x-dead-letter-routing-key — routing key для DLX
│   ├── x-max-priority         — макс. приоритет (1–255)
│   └── x-queue-type           — тип очереди (classic, quorum, stream)
└── bindings      — список binding'ов к exchange'ам
```

---

## 3. Что такое Consumer и как он работает

### Определение

**Consumer** — процесс, который подписывается на очередь RabbitMQ и получает сообщения для обработки. Consumer — это долгоживущий процесс (long-running), который работает в бесконечном цикле, ожидая новых сообщений.

### Два режима получения сообщений

#### 1. Push (Basic.Consume) — рекомендуемый

```
RabbitMQ ──push──► Consumer
```

Consumer подписывается на очередь, и RabbitMQ **сам** отправляет сообщения по мере поступления. Это основной режим работы.

```php
// Низкоуровневый пример с php-amqplib
$channel->basic_consume(
    queue: 'orders',           // Имя очереди
    consumer_tag: 'worker-1',  // Уникальный тег consumer'а
    no_local: false,           // Не получать свои сообщения
    no_ack: false,             // Требовать подтверждения (ВАЖНО!)
    exclusive: false,          // Не эксклюзивная подписка
    nowait: false,
    callback: function (AMQPMessage $msg) {
        // Обработка сообщения
        echo "Received: " . $msg->body . "\n";
        
        // Подтверждение
        $msg->ack();
    }
);

// Бесконечный цикл ожидания сообщений
while ($channel->is_consuming()) {
    $channel->wait(); // Блокирующее ожидание
}
```

#### 2. Pull (Basic.Get) — для специальных случаев

```
Consumer ──get──► RabbitMQ (polling)
```

Consumer сам запрашивает сообщение. Менее эффективно — используется для batch-обработки или если push-модель не подходит.

```php
// Pull-режим: получаем одно сообщение
$message = $channel->basic_get('orders', no_ack: false);

if ($message !== null) {
    processMessage($message->body);
    $message->ack();
} else {
    echo "Queue is empty\n";
}
```

### Как Consumer работает внутри (Push)

```
1. Consumer → RabbitMQ: Basic.Consume (подписка на очередь)
2. RabbitMQ → Consumer: Basic.ConsumeOk (подтверждение подписки)

--- Цикл ---
3. RabbitMQ → Consumer: Basic.Deliver (сообщение + delivery_tag)
4. Consumer: обрабатывает сообщение
5. Consumer → RabbitMQ: Basic.Ack (или Basic.Nack/Reject)
6. RabbitMQ: удаляет сообщение из очереди (или requeue)
--- Повтор с п.3 ---
```

### Конкурентные Consumer'ы (Competing Consumers)

```
Queue: orders
├── message 1 → Consumer A (обрабатывает)
├── message 2 → Consumer B (обрабатывает)
├── message 3 → Consumer A (обрабатывает следующий)
├── message 4 → Consumer B (обрабатывает следующий)
└── ...
```

RabbitMQ распределяет сообщения между consumer'ами **round-robin** (по умолчанию). Это позволяет масштабировать обработку горизонтально.

---

## 4. Жизненный цикл сообщения

```
┌──────────┐     ┌──────────┐     ┌─────────┐     ┌──────────┐
│ Producer │────►│ Exchange │────►│  Queue  │────►│ Consumer │
│ publish  │     │ routing  │     │  FIFO   │     │ process  │
└──────────┘     └──────────┘     └─────────┘     └──────────┘
                                       │                │
                                       │          ┌─────▼─────┐
                                       │          │  Ack?     │
                                       │          ├───────────┤
                                       │          │ Yes → Del │
                                       │          │ Nack+Req →│──► Обратно в Queue
                                       │          │ Reject  → │──► DLX или Del
                                       │          └───────────┘
                                       │
                                  TTL expired?
                                       │
                                  Yes → DLX
```

### Состояния сообщения в очереди

```
Message States:
├── Ready       — ожидает доставки consumer'у
├── Unacked     — доставлено consumer'у, ожидает ack/nack
├── [Acked]     — подтверждено → удалено из очереди
├── [Nacked]    — отклонено → requeue или DLX
└── [Rejected]  — отвергнуто → drop или DLX
```

### Persistent Messages (долговечные сообщения)

Для гарантии сохранности при перезапуске RabbitMQ:

```php
// Producer: delivery_mode=2 → persistent
$message = new AMQPMessage(
    json_encode(['order_id' => 42]),
    [
        'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT, // 2
        'content_type'  => 'application/json',
    ]
);

// Queue должна быть durable=true
$channel->queue_declare('orders', durable: true);

$channel->basic_publish($message, exchange: '', routing_key: 'orders');
```

**Оба условия необходимы:**
1. Queue: `durable: true`
2. Message: `delivery_mode: 2` (persistent)

---

## 5. Acknowledgement (ack/nack/reject)

### Зачем нужен Acknowledgement

Без подтверждения RabbitMQ не знает, обработано ли сообщение. Если consumer упал — сообщение будет потеряно.

### Три типа подтверждения

#### 1. basic_ack — успешная обработка

```php
// Сообщение обработано — удалить из очереди
$channel->basic_ack($message->getDeliveryTag());

// Multiple ack: подтвердить все сообщения до этого delivery_tag включительно
$channel->basic_ack($message->getDeliveryTag(), multiple: true);
```

#### 2. basic_nack — неудачная обработка

```php
// Неудача — вернуть в очередь для повторной обработки
$channel->basic_nack($message->getDeliveryTag(), multiple: false, requeue: true);

// Неудача — НЕ возвращать (отправить в DLX или удалить)
$channel->basic_nack($message->getDeliveryTag(), multiple: false, requeue: false);
```

#### 3. basic_reject — отклонение одного сообщения

```php
// Отклонить и вернуть в очередь
$channel->basic_reject($message->getDeliveryTag(), requeue: true);

// Отклонить без возврата (DLX или удаление)
$channel->basic_reject($message->getDeliveryTag(), requeue: false);
```

### Разница между nack и reject

| | `basic_reject` | `basic_nack` |
|---|---|---|
| Количество | Только 1 сообщение | Одно или несколько (multiple) |
| requeue | Да | Да |
| Использование | Стандартный AMQP | Расширение RabbitMQ |

### Что происходит при disconnect без ack

Если consumer отключается (crash, network failure) и НЕ отправил ack:
- RabbitMQ **автоматически** возвращает все unacked сообщения в очередь
- Другой consumer получит эти сообщения
- Это обеспечивает **at-least-once** семантику

### Auto-ack (no_ack=true) — опасно!

```php
// ❌ ПЛОХО для продакшена: сообщение считается доставленным сразу
$channel->basic_consume('orders', no_ack: true, callback: $handler);
// Если consumer упал ВО ВРЕМЯ обработки — сообщение ПОТЕРЯНО!

// ✅ ХОРОШО: ручное подтверждение
$channel->basic_consume('orders', no_ack: false, callback: $handler);
```

> 🔰 **Новичку, почему это важно:** `no_ack: true` (в AMQP-спецификации называется «auto-ack mode») означает, что RabbitMQ удаляет сообщение из очереди **сразу после отправки** consumer'у — не дожидаясь подтверждения. Это даёт семантику **at-most-once**: если consumer упадёт во время обработки, сообщение не вернётся. Противоположный режим (`no_ack: false`) даёт **at-least-once**: RabbitMQ будет держать сообщение в статусе `unacked` и вернёт в очередь при обрыве соединения (см. [RabbitMQ Confirms](https://www.rabbitmq.com/docs/confirms)).

> **Правило:** В продакшене ВСЕГДА используйте `no_ack: false` (ручное подтверждение). В Symfony Messenger этот режим включён по умолчанию — вручную включать ничего не нужно.

---

## 6. Prefetch Count (QoS)

### Проблема

По умолчанию RabbitMQ отправляет consumer'у все сообщения из очереди разом. Если у consumer'а 10 000 unacked сообщений, а он обрабатывает медленно — это:
1. Занимает много памяти consumer'а
2. Блокирует доставку другим consumer'ам
3. При crash — 10 000 сообщений вернутся в очередь

### Решение: prefetch_count

```php
// Получать не более 10 сообщений одновременно
// Сигнатура php-amqplib: basic_qos($prefetch_size, $prefetch_count, $a_global)
// ВАЖНО: параметр называется $a_global (а не $global — это зарезервированное слово PHP),
// поэтому при именованных аргументах нужно писать a_global:
$channel->basic_qos(
    prefetch_size: 0,     // Не ограничивать по размеру в байтах (0 = unlimited)
    prefetch_count: 10,   // Максимум 10 unacked сообщений одновременно
    a_global: false       // false = лимит per-consumer, true = per-channel (AMQP 0-9-1 spec)
);
```

> 🔰 **Новичку:** `prefetch_count` — это не «размер батча», а «сколько сообщений RabbitMQ разрешено отправить consumer'у, пока тот не подтвердил (ack) хотя бы одно». Это окно между брокером и consumer'ом. Подробнее в [официальной документации](https://www.rabbitmq.com/docs/consumer-prefetch).
>
> **Про `a_global`:** в AMQP 0-9-1 этот флаг означает область действия QoS. В RabbitMQ интерпретация переопределена: `false` = применить к каждому новому consumer'у в канале, `true` = применить суммарно ко всем consumer'ам канала. Почти всегда нужно `false` ([RabbitMQ docs — Consumer Prefetch: global=true semantics](https://www.rabbitmq.com/docs/consumer-prefetch)).

### Как работает prefetch

```
prefetch_count = 3

RabbitMQ Queue: [msg1] [msg2] [msg3] [msg4] [msg5] ...

              ┌──────────────────────┐
              │ Consumer (3 слота)   │
              │ ┌─────┐┌─────┐┌─────┐│
RabbitMQ ───► │ │msg1 ││msg2 ││msg3 ││ ← Заполнен
              │ └─────┘└─────┘└─────┘│
              └──────────────────────┘

Consumer обработал msg1, отправил ack:

              ┌──────────────────────┐
              │ Consumer (3 слота)   │
              │ ┌─────┐┌─────┐┌─────┐│
RabbitMQ ───► │ │msg4 ││msg2 ││msg3 ││ ← msg4 доставлен на освободившийся слот
              │ └─────┘└─────┘└─────┘│
              └──────────────────────┘
```

### Выбор prefetch_count

| Значение | Когда | Пояснение |
|----------|-------|-----------|
| **1** | Тяжёлая обработка (>1 сек) | Равномерная нагрузка, но низкий throughput |
| **10–50** | Средняя обработка (100ms–1s) | Баланс throughput/fairness |
| **100–250** | Лёгкая обработка (<100ms) | Высокий throughput |
| **0 (unlimited)** | ❌ Не рекомендуется | Захватит всю очередь |

> **Рекомендация для Symfony Messenger:** начните с `prefetch_count=10`, измеряйте и корректируйте.

### Компромисс: Throughput vs Fairness

- **Большой prefetch (50+):** Consumer «забирает» сообщения пачкой → высокая пропускная способность, но если один consumer медленный, а другой быстрый — медленный будет «сидеть» на своих 50 сообщениях, а быстрый может простаивать.
- **Маленький prefetch (1–5):** Сообщения распределяются равномерно → fairness выше, но каждый ack требует round-trip к RabbitMQ → ниже throughput.
- **Как выбрать:** Измерьте среднее время обработки одного сообщения (`T`). Оптимальный prefetch ≈ `(1 / T) × RTT`, где `RTT` — время round-trip до RabbitMQ. Для типичных случаев: если `T < 50ms` → prefetch 50–100, если `T > 1s` → prefetch 1–5.

### Влияние на распределение между consumer'ами

```
Queue: 100 сообщений
Consumer A (prefetch=50): получает 50
Consumer B (prefetch=50): получает 50

vs

Consumer A (prefetch=1): получает 1 → ack → 1 → ack → ...
Consumer B (prefetch=1): получает 1 → ack → 1 → ack → ...
// Более равномерно, но медленнее
```

---

## 7. Типы Exchange

### Direct Exchange

Маршрутизация по точному совпадению routing key:

```
Producer → Exchange (direct) → routing_key = "order.created"
                                    │
                    Binding: "order.created" → Queue: process_orders ✅
                    Binding: "user.created"  → Queue: send_welcome   ❌
```

### Fanout Exchange

Broadcast: копирует сообщение во ВСЕ привязанные очереди:

```
Producer → Exchange (fanout) → Queue: audit_log     ✅
                              → Queue: analytics     ✅
                              → Queue: notifications ✅
```

### Topic Exchange

Маршрутизация по паттерну routing key (с `*` и `#`):

```
Producer → routing_key = "order.created.eu"

Binding: "order.created.*"  → Queue A   ✅ (* = одно слово)
Binding: "order.#"          → Queue B   ✅ (# = ноль или более слов)
Binding: "user.*"           → Queue C   ❌ (не матчит)
Binding: "#"                → Queue D   ✅ (# матчит всё)
```

### Headers Exchange

Маршрутизация по заголовкам сообщения (вместо routing key):

```php
$headers = ['x-match' => 'all', 'region' => 'eu', 'type' => 'order'];
// all: все заголовки должны совпасть
// any: хотя бы один
```

### Default Exchange (nameless)

Каждая очередь автоматически привязана к default exchange с `routing_key = queue_name`:

```php
// Отправка напрямую в очередь "orders"
$channel->basic_publish($message, exchange: '', routing_key: 'orders');
```

---

## 8. Dead Letter Exchange (DLX)

### Назначение

DLX — это exchange, куда попадают «мёртвые» сообщения, которые не удалось обработать.

### Когда сообщение попадает в DLX

1. **Reject/Nack без requeue** — `basic_reject(requeue: false)` или `basic_nack(requeue: false)`
2. **TTL истёк** — сообщение пролежало в очереди дольше `x-message-ttl`
3. **Очередь переполнена** — превышен `x-max-length` или `x-max-length-bytes`

> 🔰 **Новичку, зачем DLX:** представьте, что handler упал на сообщении с «битыми» данными (невалидный JSON, несуществующий userId). Если делать `requeue: true`, сообщение бесконечно будет возвращаться и крашить consumer. Если просто удалить — мы потеряем данные и не сможем разобраться. DLX — это «карантин»: сообщение переносится в отдельную очередь, где инженер/DevOps может его посмотреть, исправить и повторно отправить. Это стандартный паттерн — см. [официальный гайд](https://www.rabbitmq.com/docs/dlx).
>
> **Важный нюанс:** причина попадания в DLX записывается в заголовок `x-death` (массив с историей «смертей» сообщения). По нему можно отличить «истёкший TTL» от «отклонён consumer'ом».

### Конфигурация

```php
// Создаём DLX exchange
$channel->exchange_declare('dlx_exchange', 'direct', durable: true);

// Создаём очередь для "мёртвых" сообщений
$channel->queue_declare('dead_letters', durable: true);
$channel->queue_bind('dead_letters', 'dlx_exchange', routing_key: 'orders');

// Основная очередь с DLX
$channel->queue_declare('orders', durable: true, arguments: new AMQPTable([
    'x-dead-letter-exchange'    => 'dlx_exchange',
    'x-dead-letter-routing-key' => 'orders',
]));
```

### Схема работы

```
Producer → [orders exchange] → [orders queue]
                                     │
                              Consumer fails
                              nack(requeue=false)
                                     │
                                     ▼
                               [dlx_exchange] → [dead_letters queue]
                                                       │
                                                 Admin reviews
                                                 or retry logic
```

---

## 9. TTL, Expiration, Priority

### Message TTL

```php
// TTL для конкретного сообщения
$message = new AMQPMessage($body, [
    'expiration' => '60000', // 60 секунд (в миллисекундах, строка!)
]);

// TTL для всех сообщений в очереди
$channel->queue_declare('orders', durable: true, arguments: new AMQPTable([
    'x-message-ttl' => 3600000, // 1 час (в миллисекундах, int!)
]));
```

**Разница:** `expiration` на сообщении проверяется при доставке consumer'у. `x-message-ttl` на очереди — при поступлении (head of queue).

### Queue TTL

```php
// Очередь удалится, если неиспользуется 30 минут
$channel->queue_declare('temp_queue', arguments: new AMQPTable([
    'x-expires' => 1800000, // 30 минут в мс
]));
```

### Priority Queues

```php
// Очередь с поддержкой приоритетов (1–10)
$channel->queue_declare('priority_orders', durable: true, arguments: new AMQPTable([
    'x-max-priority' => 10,
]));

// Сообщение с высоким приоритетом
$highPriority = new AMQPMessage($body, ['priority' => 9]);
$channel->basic_publish($highPriority, '', 'priority_orders');

// Обычное сообщение
$normalPriority = new AMQPMessage($body, ['priority' => 1]);
$channel->basic_publish($normalPriority, '', 'priority_orders');
```

> **⚠️ Приоритетные очереди** потребляют больше ресурсов. Используйте только когда действительно нужен приоритет. Для большинства задач лучше использовать отдельные очереди.

> 🔰 **Почему «лучше отдельные очереди»?** Priority queues в RabbitMQ реализованы через внутренние суб-очереди на каждый уровень приоритета ([официальная документация](https://www.rabbitmq.com/docs/priority)), что увеличивает расход памяти и CPU, и **несовместимо с quorum queues** (они не поддерживают приоритеты). Альтернатива — создать две очереди (`orders.high`, `orders.low`) и запускать Symfony-consumer с перечислением транспортов: `messenger:consume async_high async_low` — воркер всегда сначала опустошит `high`, потом перейдёт к `low`. Это дешевле и прозрачнее.

---

## 10. Retry-стратегии

### Наивный retry (requeue=true) — антипаттерн!

```php
// ❌ ПЛОХО: бесконечный цикл retry
$callback = function (AMQPMessage $msg) {
    try {
        processMessage($msg->body);
        $msg->ack();
    } catch (\Throwable $e) {
        $msg->nack(requeue: true); // Вернётся в начало очереди → бесконечный retry!
    }
};
```

### Retry через DLX с TTL (delay)

```
Main Queue → (fail) → DLX → Retry Queue (TTL=30s) → (TTL expired) → DLX → Main Queue

Попытка 1: Main Queue → Fail → Retry Queue (30s delay)
Попытка 2: Main Queue → Fail → Retry Queue (30s delay)
Попытка 3: Main Queue → Fail → Dead Letter Queue (финальный отказ)
```

> 🔰 **Новичку, как это работает механически:** мы создаём «буферную» очередь `retry_queue` с двумя аргументами: `x-message-ttl=30000` (любое сообщение живёт 30 секунд) и `x-dead-letter-exchange=main_exchange` (после TTL отправить обратно в главный exchange). При ошибке consumer публикует сообщение в `retry_queue` (или nack без requeue перенаправляет туда). Через 30 секунд RabbitMQ сам перекладывает сообщение обратно в main — получается отложенная повторная попытка **без специальных плагинов и cron-задач**. Это классический паттерн, описанный в [официальном гайде по DLX](https://www.rabbitmq.com/docs/dlx).

### Retry с экспоненциальным backoff через несколько очередей

```
                   fail (attempt 1)
Main Queue ──────────────────────► retry_1 (TTL=5s)  ──► Main Queue
                   fail (attempt 2)
Main Queue ──────────────────────► retry_2 (TTL=30s) ──► Main Queue
                   fail (attempt 3)
Main Queue ──────────────────────► retry_3 (TTL=120s)──► Main Queue
                   fail (attempt 4+)
Main Queue ──────────────────────► dead_letters (финал)
```

---

## 11. Symfony Messenger + RabbitMQ

### Установка

```bash
composer require symfony/messenger symfony/amqp-messenger

# Также нужен ext-amqp
# Debian/Ubuntu: apt install php8.2-amqp
# Alpine: apk add php82-amqp
```

> 🔰 **Важно про транспорт:** `symfony/amqp-messenger` использует расширение **ext-amqp** (нативное C-расширение, работающее поверх librabbitmq). Это НЕ то же самое, что популярная библиотека `php-amqplib/php-amqplib` (чистый PHP). Оба работают, но официальный AMQP-транспорт Symfony Messenger построен на `ext-amqp` — это быстрее и стабильнее для долгоживущих воркеров. См. [Symfony AMQP Transport](https://symfony.com/doc/6.4/messenger.html#amqp-transport).

### Архитектура Symfony Messenger

```
                    ┌────────────────────────────────────┐
                    │         Symfony Application         │
                    │                                    │
Controller/Service ─┤  MessageBus::dispatch($message)   │
                    │       │                            │
                    │  ┌────▼─────┐                     │
                    │  │Middleware│ (validation, logging) │
                    │  └────┬─────┘                     │
                    │       │                            │
                    │  ┌────▼──────────────┐            │
                    │  │ SendMessageMiddleware │         │
                    │  │  (routing → transport)│         │
                    │  └────┬──────────────┘            │
                    │       │                            │
                    │  ┌────▼───────┐                   │
                    │  │  Transport │ ─── AMQP ──► RabbitMQ
                    │  └────────────┘                   │
                    └────────────────────────────────────┘

                    ┌────────────────────────────────────┐
                    │    bin/console messenger:consume    │
                    │                                    │
RabbitMQ ──AMQP──►  │  ┌────────────┐                   │
                    │  │  Transport │ (receive message)  │
                    │  └────┬───────┘                   │
                    │  ┌────▼─────┐                     │
                    │  │Middleware│                      │
                    │  └────┬─────┘                     │
                    │  ┌────▼────────────┐              │
                    │  │ HandleMessage   │              │
                    │  │  Middleware     │              │
                    │  └────┬────────────┘              │
                    │  ┌────▼──────┐                    │
                    │  │  Handler  │ (бизнес-логика)    │
                    │  └───────────┘                    │
                    └────────────────────────────────────┘
```

---

## 12. Конфигурация транспорта RabbitMQ в Symfony

### Базовая конфигурация

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000        # мс (первый retry)
                    multiplier: 2      # каждый следующий retry x2
                    max_delay: 60000   # максимум 60 сек
                options:
                    exchange:
                        name: app_messages
                        type: direct
                    queues:
                        messages:
                            binding_keys: [app_messages]
        
        routing:
            App\Message\SendNotification: async
            App\Message\ProcessOrder: async
```

```dotenv
# .env
MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost:5672/%2f/messages
```

### Расширенная конфигурация

```yaml
framework:
    messenger:
        # Сериализация
        serializer:
            default_serializer: messenger.transport.symfony_serializer
            symfony_serializer:
                format: json
                context: {}

        # Failure Transport (куда складывать failed сообщения)
        failure_transport: failed

        transports:
            # Основной async-транспорт
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000
                    multiplier: 3
                    max_delay: 300000  # 5 минут
                    # Кастомный retry (класс):
                    # service: App\Messenger\CustomRetryStrategy
                options:
                    exchange:
                        name: app_exchange
                        type: direct
                        default_publish_routing_key: default
                    queues:
                        default_queue:
                            binding_keys: [default]
                    # Prefetch count
                    prefetch_count: 10
                    # Heartbeat для long-running consumer'ов
                    # heartbeat: 60
                    
            # Высокоприоритетная очередь
            async_priority_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 5
                    delay: 500
                    multiplier: 2
                options:
                    exchange:
                        name: app_exchange
                        type: direct
                        default_publish_routing_key: high_priority
                    queues:
                        high_priority_queue:
                            binding_keys: [high_priority]
                    prefetch_count: 5
            
            # Транспорт для failed сообщений
            failed:
                dsn: 'doctrine://default?queue_name=failed'
                # Или AMQP:
                # dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                # options:
                #     exchange:
                #         name: failed_exchange
                #     queues:
                #         failed_queue: ~

        routing:
            App\Message\SendOrderConfirmation: async_priority_high
            App\Message\GenerateInvoice: async
            App\Message\SyncToExternalCrm: async
            App\Message\SendNewsletter: async
```

### DSN-формат

```
amqp://user:password@host:port/vhost/queue_name?option=value

# Примеры:
amqp://guest:guest@localhost:5672/%2f/messages
amqp://app_user:secret@rabbitmq:5672/my_vhost/messages?heartbeat=60&connection_timeout=5
amqp://user:pass@rabbit1:5672/%2f?lazy=1&persistent=true

# Кластер (failover):
amqp://user:pass@rabbit1:5672,rabbit2:5672/%2f
```

---

## 13. Создание Message и Handler

### Message (DTO)

```php
<?php

declare(strict_types=1);

namespace App\Message;

/**
 * Сообщение для отправки email-подтверждения заказа.
 * Message — это простой DTO (Data Transfer Object).
 * Он ДОЛЖЕН быть сериализуемым (для передачи через AMQP).
 */
final readonly class SendOrderConfirmation
{
    public function __construct(
        public int $orderId,
        public string $customerEmail,
        public string $customerName,
    ) {}
}
```

> 🔰 **Что такое Envelope и Stamp (нужно знать для собеседования):** Messenger не передаёт «голый» message-объект по шине — он заворачивает его в **Envelope** (конверт). На конверте могут висеть **Stamps** (марки) — метаданные, влияющие на маршрутизацию и обработку:
> - `BusNameStamp` — через какую шину идём (command/query/event).
> - `TransportNamesStamp` — в какой транспорт отправить (можно перекрыть роутинг).
> - `DelayStamp` — отложить доставку на N миллисекунд.
> - `AmqpStamp` — кастомный routing_key, exchange, priority для AMQP.
> - `HandledStamp` — результат работы handler'а (заполняется после обработки).
> - `RedeliveryStamp` — счётчик попыток (используется retry-стратегией).
>
> Stamps иммутабельны — `$envelope->with(new DelayStamp(5000))` возвращает новый Envelope. Подробнее: [Symfony Messenger — Envelopes & Stamps](https://symfony.com/doc/6.4/messenger.html#adding-metadata-to-messages-envelopes).

### Handler

```php
<?php

declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\SendOrderConfirmation;
use App\Repository\OrderRepository;
use App\Service\EmailSender;
use Psr\Log\LoggerInterface;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class SendOrderConfirmationHandler
{
    public function __construct(
        private readonly OrderRepository $orderRepository,
        private readonly EmailSender $emailSender,
        private readonly LoggerInterface $logger,
    ) {}

    public function __invoke(SendOrderConfirmation $message): void
    {
        $order = $this->orderRepository->find($message->orderId);

        if ($order === null) {
            $this->logger->warning('Order not found, skipping confirmation email', [
                'order_id' => $message->orderId,
            ]);
            return; // Ack — сообщение обработано (нечего обрабатывать)
        }

        $this->emailSender->sendOrderConfirmation(
            to: $message->customerEmail,
            customerName: $message->customerName,
            order: $order,
        );

        $this->logger->info('Order confirmation email sent', [
            'order_id' => $message->orderId,
            'email'    => $message->customerEmail,
        ]);
    }
}
```

### Dispatch из контроллера/сервиса

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Message\SendOrderConfirmation;
use App\Message\GenerateInvoice;
use App\Message\SyncToExternalCrm;
use Symfony\Component\Messenger\MessageBusInterface;

final class OrderController extends AbstractController
{
    #[Route('/api/orders', methods: ['POST'])]
    public function create(
        Request $request,
        MessageBusInterface $bus,
    ): JsonResponse {
        $order = $this->orderService->create($request);

        // Тяжёлые операции отправляются в очередь
        $bus->dispatch(new SendOrderConfirmation(
            orderId: $order->getId(),
            customerEmail: $order->getCustomer()->getEmail(),
            customerName: $order->getCustomer()->getName(),
        ));

        $bus->dispatch(new GenerateInvoice($order->getId()));
        $bus->dispatch(new SyncToExternalCrm($order->getId()));

        return $this->json(['id' => $order->getId()], 201);
        // Ответ за 50ms вместо 2000ms
    }
}
```

### Несколько handler'ов на одно сообщение

```php
// Одно сообщение может обрабатываться несколькими handler'ами
#[AsMessageHandler]
final class LogOrderCreatedHandler
{
    public function __invoke(OrderCreated $message): void
    {
        // Логирование
    }
}

#[AsMessageHandler]
final class NotifyWarehouseHandler
{
    public function __invoke(OrderCreated $message): void
    {
        // Уведомление склада
    }
}
// Оба handler'а будут вызваны для каждого OrderCreated сообщения
```

---

## 14. Запуск и управление Consumer (Worker)

### Основная команда

```bash
# Запуск consumer'а для транспорта "async"
bin/console messenger:consume async

# Несколько транспортов (приоритет слева направо)
bin/console messenger:consume async_priority_high async

# Ограничения
bin/console messenger:consume async \
    --limit=100 \             # Остановиться после 100 сообщений
    --time-limit=3600 \       # Остановиться через 1 час
    --memory-limit=256 \      # Остановиться при 256MB памяти
    --sleep=1 \               # Спать 1 сек при пустой очереди
    --queues=high_priority    # Конкретная очередь (AMQP)

# Режим verbose (для отладки)
bin/console messenger:consume async -vv
```

### Что происходит при запуске

```
1. Worker подключается к RabbitMQ
2. Worker подписывается на очередь (basic_consume)
3. Бесконечный цикл:
   a. Получение сообщения из очереди
   b. Десериализация
   c. Прогон через middleware
   d. Вызов handler'а
   e. Ack (успех) или Nack (ошибка → retry/fail)
   f. Проверка лимитов (memory, time, count)
4. Worker завершается (supervisor перезапустит)
```

### Опции ограничения (ВАЖНО для продакшена!)

| Опция | Назначение |
|-------|------------|
| `--limit=N` | Обработать N сообщений и завершиться (против memory leak) |
| `--time-limit=N` | Работать N секунд и завершиться |
| `--memory-limit=N` | Завершиться при превышении N MB |
| `--sleep=N` | Пауза N сек при пустой очереди |

> **Рекомендация:** Всегда ставьте `--time-limit` и `--memory-limit`. Supervisor перезапустит worker. Это предотвращает memory leaks.

---

## 15. Supervisor — управление воркерами в продакшене

### Конфигурация Supervisor

```ini
; /etc/supervisor/conf.d/messenger-worker.conf

[program:messenger-consume-async]
command=php /var/www/app/bin/console messenger:consume async --time-limit=3600 --memory-limit=256 --limit=500
user=www-data
numprocs=4                          ; 4 параллельных воркера
process_name=%(program_name)s_%(process_num)02d
autostart=true
autorestart=true                    ; Перезапуск при завершении
startsecs=0
startretries=10
stopwaitsecs=60                     ; Ждать 60 сек до SIGKILL
stopasgroup=true
killasgroup=true
redirect_stderr=true
stdout_logfile=/var/log/messenger/async_%(process_num)02d.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=5
environment=
    APP_ENV="prod",
    MESSENGER_TRANSPORT_DSN="amqp://user:pass@rabbitmq:5672/%%2f/messages"

[program:messenger-consume-high-priority]
command=php /var/www/app/bin/console messenger:consume async_priority_high --time-limit=3600 --memory-limit=128
user=www-data
numprocs=2
process_name=%(program_name)s_%(process_num)02d
autostart=true
autorestart=true
startsecs=0
redirect_stderr=true
stdout_logfile=/var/log/messenger/high_priority_%(process_num)02d.log
```

```bash
# Управление
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start messenger-consume-async:*
sudo supervisorctl stop messenger-consume-async:*
sudo supervisorctl restart messenger-consume-async:*
sudo supervisorctl status
```

### При деплое

```bash
# 1. Остановить воркеры (graceful)
sudo supervisorctl stop messenger-consume-async:*
sudo supervisorctl stop messenger-consume-high-priority:*

# 2. Деплой кода
git pull
composer install --no-dev
bin/console cache:warmup --env=prod

# 3. Перезапустить воркеры
sudo supervisorctl start messenger-consume-async:*
sudo supervisorctl start messenger-consume-high-priority:*
```

---

## 16. Failure Transport (обработка ошибок)

### Конфигурация

```yaml
framework:
    messenger:
        failure_transport: failed

        transports:
            failed:
                dsn: 'doctrine://default?queue_name=failed'
```

### Что происходит при ошибке

```
Сообщение → Handler → Exception!
    │
    ├── Retry 1 (delay 1s) → Handler → Exception!
    ├── Retry 2 (delay 3s) → Handler → Exception!
    ├── Retry 3 (delay 9s) → Handler → Exception!
    │
    └── Max retries exceeded → Failure Transport
        (сохраняется в таблице messenger_messages)
```

### Управление failed сообщениями

```bash
# Показать failed сообщения
bin/console messenger:failed:show

# Показать конкретное
bin/console messenger:failed:show 42

# Повторить обработку одного сообщения
bin/console messenger:failed:retry 42

# Повторить все
bin/console messenger:failed:retry --force

# Удалить сообщение
bin/console messenger:failed:remove 42

# Удалить все
bin/console messenger:failed:remove --force --all
```

---

## 17. Retry механизм в Symfony Messenger

### Конфигурация retry

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    max_retries: 3
                    delay: 1000          # Базовая задержка (мс)
                    multiplier: 2        # Множитель
                    max_delay: 60000     # Максимальная задержка (мс)
                    # Jitter:
                    # Symfony 6.4+ автоматически добавляет jitter
```

### Вычисление задержки

```
delay * multiplier ^ (retry - 1)

Retry 1: 1000 * 2^0 = 1000ms  (1 секунда)
Retry 2: 1000 * 2^1 = 2000ms  (2 секунды)
Retry 3: 1000 * 2^2 = 4000ms  (4 секунды)
Retry 4: 1000 * 2^3 = 8000ms  (8 секунд, но max_delay=60000)
```

### Кастомная retry-стратегия

```php
<?php

declare(strict_types=1);

namespace App\Messenger;

use App\Message\SendNotification;
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Retry\RetryStrategyInterface;
use Symfony\Component\Messenger\Stamp\RedeliveryStamp;

final class CustomRetryStrategy implements RetryStrategyInterface
{
    public function isRetryable(Envelope $message, ?\Throwable $throwable = null): bool
    {
        $retryCount = RedeliveryStamp::getRetryCountFromEnvelope($message);

        // Не ретраить некоторые типы ошибок
        if ($throwable instanceof \InvalidArgumentException) {
            return false; // Невалидные данные — ретрай не поможет
        }

        // Maximum retries
        return $retryCount < 5;
    }

    public function getWaitingTime(Envelope $message, ?\Throwable $throwable = null): int
    {
        $retryCount = RedeliveryStamp::getRetryCountFromEnvelope($message);

        // Экспоненциальный backoff с jitter
        $delay = (int) (1000 * (2 ** $retryCount));
        $jitter = random_int(0, (int) ($delay * 0.1));

        return min($delay + $jitter, 300_000); // max 5 минут
    }
}
```

```yaml
# config/services.yaml
services:
    App\Messenger\CustomRetryStrategy: ~

# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                retry_strategy:
                    service: App\Messenger\CustomRetryStrategy
```

### Отключение retry для конкретных исключений

```php
use Symfony\Component\Messenger\Exception\UnrecoverableMessageHandlingException;

#[AsMessageHandler]
final class ProcessPaymentHandler
{
    public function __invoke(ProcessPayment $message): void
    {
        try {
            $this->paymentGateway->charge($message->amount);
        } catch (CardDeclinedException $e) {
            // Карта отклонена — retry не поможет
            throw new UnrecoverableMessageHandlingException(
                'Payment declined: ' . $e->getMessage(),
                previous: $e
            );
            // Сообщение сразу попадёт в failure transport (без retry)
        } catch (GatewayTimeoutException $e) {
            // Таймаут — retry может помочь
            throw $e; // Стандартный механизм retry
        }
    }
}
```

> 🔰 **Новичку, главное правило:** **не все ошибки одинаково полезно ретраить**.
> - **Retryable** (transient) — сетевые ошибки, таймауты, временная недоступность сервиса, deadlock в БД. Бросаем обычное исключение → Messenger ретраит с backoff.
> - **Unrecoverable** (permanent) — невалидные данные, бизнес-правило запрещает операцию, отсутствует сущность. Бросаем `UnrecoverableMessageHandlingException` → Messenger сразу кладёт сообщение в `failure_transport`, не тратя попытки. Это экономит ресурсы и ускоряет реакцию оператора. См. [Symfony docs — Stopping Retries](https://symfony.com/doc/6.4/messenger.html#avoiding-retrying).
>
> Есть ещё «двойник» — `RecoverableMessageHandlingException`: явно говорит Messenger «да, ретраи имеют смысл» (полезно, если вы хотите переопределить кастомную логику `isRetryable()` стратегии).

---

## 18. Multiple Consumers и приоритетные очереди

### Приоритетные очереди через отдельные транспорты

```yaml
framework:
    messenger:
        transports:
            async_high:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queues:
                        high_priority: ~
            async_low:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    queues:
                        low_priority: ~

        routing:
            App\Message\SendOrderConfirmation: async_high
            App\Message\SendNewsletter: async_low
```

```bash
# Consumer: сначала обрабатывает high, потом low
bin/console messenger:consume async_high async_low
# Если в async_high есть сообщения — они обрабатываются первыми
```

### Масштабирование consumer'ов

```ini
; Supervisor — больше воркеров для критичных очередей
[program:messenger-high]
command=php bin/console messenger:consume async_high --time-limit=3600
numprocs=4    ; 4 воркера для высокоприоритетных

[program:messenger-low]
command=php bin/console messenger:consume async_low --time-limit=3600
numprocs=2    ; 2 воркера для низкоприоритетных
```

---

## 19. Serialization сообщений

### Формат по умолчанию (PHP serialize)

```php
// По умолчанию Symfony Messenger использует PHP native serialization
// Быстро, но непереносимо между языками
```

### JSON-сериализация (рекомендуется для cross-service)

```yaml
framework:
    messenger:
        serializer:
            default_serializer: messenger.transport.symfony_serializer
            symfony_serializer:
                format: json
                context:
                    # Дополнительный контекст сериализации
```

### Кастомный Serializer

```php
<?php

declare(strict_types=1);

namespace App\Messenger\Serializer;

use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Serialization\SerializerInterface;
use Symfony\Component\Messenger\Stamp\BusNameStamp;

/**
 * Кастомный сериализатор для интеграции с внешними сервисами.
 * Позволяет получать сообщения от не-Symfony producer'ов.
 */
final class ExternalEventSerializer implements SerializerInterface
{
    public function decode(array $encodedEnvelope): Envelope
    {
        $body = json_decode($encodedEnvelope['body'], true, flags: JSON_THROW_ON_ERROR);
        $headers = $encodedEnvelope['headers'] ?? [];

        // Определяем тип сообщения по заголовку
        $type = $headers['type'] ?? $body['event_type'] ?? 'unknown';

        $message = match ($type) {
            'order.created'   => new \App\Message\External\OrderCreated(
                orderId: $body['order_id'],
                amount: $body['amount'],
            ),
            'user.registered' => new \App\Message\External\UserRegistered(
                userId: $body['user_id'],
                email: $body['email'],
            ),
            default => throw new \RuntimeException("Unknown message type: {$type}"),
        };

        return new Envelope($message, [new BusNameStamp('messenger.bus.default')]);
    }

    public function encode(Envelope $envelope): array
    {
        $message = $envelope->getMessage();

        return [
            'body'    => json_encode($message, JSON_THROW_ON_ERROR),
            'headers' => [
                'type'         => $message::class,
                'Content-Type' => 'application/json',
            ],
        ];
    }
}
```

```yaml
# Использование кастомного сериализатора для конкретного транспорта
framework:
    messenger:
        transports:
            external_events:
                dsn: '%env(EXTERNAL_RABBITMQ_DSN)%'
                serializer: App\Messenger\Serializer\ExternalEventSerializer
```

---

## 20. Graceful Shutdown и сигналы

### Как Symfony Messenger обрабатывает сигналы

```
SIGTERM / SIGINT → Worker получает сигнал
    │
    ├── Если сейчас обрабатывается сообщение:
    │   ├── Дождаться завершения обработки
    │   ├── Отправить ack/nack
    │   └── Завершить worker
    │
    └── Если worker idle:
        └── Завершить worker немедленно
```

Symfony Messenger **автоматически** обрабатывает сигналы через `pcntl_signal`:
- **SIGTERM** — graceful shutdown
- **SIGINT** (Ctrl+C) — graceful shutdown
- **SIGUSR1** — не используется по умолчанию

### Конфигурация Supervisor для graceful shutdown

```ini
[program:messenger-consume]
command=php bin/console messenger:consume async --time-limit=3600
stopwaitsecs=60    ; Ждать 60 сек после SIGTERM до SIGKILL
stopsignal=SIGTERM ; Отправлять SIGTERM (по умолчанию)
stopasgroup=true
killasgroup=true
```

### Ручная остановка consumer'ов

```bash
# Остановить все consumer'ы gracefully
bin/console messenger:stop-workers

# Это помещает сигнал остановки в кэш.
# Все воркеры проверяют этот сигнал между сообщениями и останавливаются.
# Требует общий кэш (например, Redis) для всех воркеров.
```

```yaml
# config/packages/cache.yaml
framework:
    cache:
        # Для messenger:stop-workers нужен общий кэш:
        app: cache.adapter.redis
```

---

## 21. Memory Leaks и долгоживущие процессы

### Проблема

Consumer — долгоживущий процесс. PHP не идеален для таких процессов: Doctrine Identity Map, логгеры, кэши — всё накапливается.

### Решение 1: Ограничения worker'а

```bash
bin/console messenger:consume async \
    --memory-limit=256 \   # Перезапуск при 256MB
    --time-limit=3600 \    # Перезапуск каждый час
    --limit=500            # Перезапуск после 500 сообщений
```

### Решение 2: Очистка EntityManager

```php
#[AsMessageHandler]
final class ProcessOrderHandler
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function __invoke(ProcessOrder $message): void
    {
        // Обработка
        $order = $this->em->find(Order::class, $message->orderId);
        // ... бизнес-логика ...
        $this->em->flush();

        // ВАЖНО: очищаем Identity Map после каждого сообщения
        $this->em->clear();
    }
}
```

### Решение 3: ResetInterface (Symfony автоматическая очистка)

```php
// Symfony Messenger автоматически вызывает reset() на сервисах,
// реализующих ResetInterface, после каждого сообщения.
// Doctrine EntityManager реализует ResetInterface → автоматически clear().

// Для собственных сервисов:
use Symfony\Contracts\Service\ResetInterface;

final class InMemoryCache implements ResetInterface
{
    private array $cache = [];

    public function get(string $key): mixed
    {
        return $this->cache[$key] ?? null;
    }

    public function reset(): void
    {
        $this->cache = []; // Очищаем при каждом сообщении
    }
}
```

### Решение 4: Middleware для очистки

```php
use Symfony\Component\Messenger\Middleware\MiddlewareInterface;
use Symfony\Component\Messenger\Middleware\StackInterface;
use Symfony\Component\Messenger\Envelope;

final class DoctrineCloseConnectionMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly EntityManagerInterface $em,
    ) {}

    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        try {
            return $stack->next()->handle($envelope, $stack);
        } finally {
            // Если соединение закрылось (таймаут) — переподключаемся
            if (!$this->em->getConnection()->isConnected()) {
                $this->em->getConnection()->connect();
            }
        }
    }
}
```

> **Symfony 6.4** по умолчанию включает `doctrine_close_connection` middleware для consumer'ов, который пингует/переоткрывает соединение.

---

## 22. Consumer в Docker и Kubernetes

### Docker Compose

```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:4-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: secret
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics -q ping
      interval: 10s
      timeout: 5s
      retries: 5

  php-worker:
    build: .
    command: php bin/console messenger:consume async --time-limit=3600 --memory-limit=256
    depends_on:
      rabbitmq:
        condition: service_healthy
    environment:
      MESSENGER_TRANSPORT_DSN: amqp://app:secret@rabbitmq:5672/%2f/messages
    deploy:
      replicas: 4  # 4 параллельных consumer'а
    restart: unless-stopped

volumes:
  rabbitmq_data:
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: messenger-worker
spec:
  replicas: 4
  selector:
    matchLabels:
      app: messenger-worker
  template:
    metadata:
      labels:
        app: messenger-worker
    spec:
      containers:
        - name: worker
          image: myapp:latest
          command:
            - php
            - bin/console
            - messenger:consume
            - async
            - --time-limit=3600
            - --memory-limit=256
          env:
            - name: MESSENGER_TRANSPORT_DSN
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: rabbitmq-dsn
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            exec:
              # Проверяем, что процесс жив. Полноценный health-check через
              # messenger:stats не подходит: он грузит весь Symfony kernel
              # на каждую пробу (дорого). Более адекватно — сам воркер
              # пишет timestamp "последней активности" в файл, а probe его читает:
              #   command: ["sh", "-c", "test $(($(date +%s) - $(cat /tmp/worker.heartbeat))) -lt 120"]
              command: ["sh", "-c", "pgrep -f 'messenger:consume' > /dev/null"]
            initialDelaySeconds: 30
            periodSeconds: 30
      terminationGracePeriodSeconds: 120  # Ждать 2 минуты до SIGKILL (graceful shutdown)
```

### HPA — автомасштабирование по глубине очереди

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: messenger-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: messenger-worker
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages_ready
          selector:
            matchLabels:
              queue: messages
        target:
          type: AverageValue
          averageValue: "50"  # Если > 50 msg на pod → scale up
```

---

## 23. Мониторинг RabbitMQ и Consumer'ов

### RabbitMQ Management UI

```
http://localhost:15672
# Default: guest / guest

# Ключевые метрики:
# - Queue depth (Messages Ready + Unacked)
# - Consumer count
# - Publish rate
# - Ack rate
# - Memory usage
```

### RabbitMQ HTTP API

```bash
# Список очередей
curl -u guest:guest http://localhost:15672/api/queues

# Статистика конкретной очереди
curl -u guest:guest http://localhost:15672/api/queues/%2f/orders

# Количество сообщений
curl -s -u guest:guest http://localhost:15672/api/queues/%2f/orders | jq '.messages'
```

### Prometheus + Grafana

```bash
# Установка RabbitMQ Prometheus plugin
rabbitmq-plugins enable rabbitmq_prometheus

# Метрики доступны:
# http://localhost:15692/metrics
```

### Ключевые метрики для алертинга

| Метрика | Порог | Действие |
|---------|-------|----------|
| `queue_messages_ready` | > 10000 | Очередь растёт — добавить consumer'ов |
| `queue_messages_unacked` | > 1000 | Consumer'ы не успевают ack'ать |
| `queue_consumers` | = 0 | Нет consumer'ов — critical alert! |
| `node_mem_used` | > 80% | RabbitMQ скоро заблокирует publish |
| `node_disk_free` | < 1GB | Диск заканчивается |
| `channel_count` | > 1000 | Утечка каналов |

### Symfony Messenger Stats

```bash
# Количество сообщений в транспорте (Symfony 6.4+)
bin/console messenger:stats

# Пример вывода:
# --------------- ------- 
#  Transport       Count  
# --------------- ------- 
#  async           1234   
#  async_priority  56     
#  failed          3      
# --------------- -------
```

---

## 24. Идемпотентность обработки сообщений

### Проблема

При at-least-once доставке сообщение может быть обработано **более одного раза** (consumer crash после обработки, но до ack).

### Решение: Idempotency Key

```php
<?php

declare(strict_types=1);

namespace App\Message;

final readonly class ProcessPayment
{
    public function __construct(
        public string $idempotencyKey,  // Уникальный ключ
        public int $orderId,
        public float $amount,
    ) {}
}

// При dispatch:
$bus->dispatch(new ProcessPayment(
    idempotencyKey: sprintf('payment_order_%d_%s', $orderId, uniqid()),
    orderId: $orderId,
    amount: 99.99,
));
```

```php
<?php

declare(strict_types=1);

namespace App\MessageHandler;

use App\Message\ProcessPayment;
use Symfony\Component\Messenger\Attribute\AsMessageHandler;

#[AsMessageHandler]
final class ProcessPaymentHandler
{
    public function __construct(
        private readonly PaymentGateway $gateway,
        private readonly Connection $connection,
    ) {}

    public function __invoke(ProcessPayment $message): void
    {
        // Проверяем, не обрабатывали ли мы уже это сообщение
        $exists = $this->connection->fetchOne(
            'SELECT 1 FROM processed_messages WHERE idempotency_key = ?',
            [$message->idempotencyKey]
        );

        if ($exists) {
            return; // Уже обработано — пропускаем (idempotent)
        }

        // Обработка + запись ключа в одной транзакции
        $this->connection->beginTransaction();
        try {
            $this->gateway->charge($message->amount);
            
            $this->connection->insert('processed_messages', [
                'idempotency_key' => $message->idempotencyKey,
                'processed_at'    => (new \DateTimeImmutable())->format('Y-m-d H:i:s'),
            ]);
            
            $this->connection->commit();
        } catch (\Throwable $e) {
            $this->connection->rollBack();
            throw $e;
        }
    }
}
```

---

## 25. Гарантии доставки

### At-Most-Once

Сообщение обрабатывается **не более одного раза**. Может быть потеряно.

```
Consumer получает сообщение → auto-ack → обработка → crash
Результат: сообщение потеряно (ack был до обработки)
```

```php
// auto-ack (no_ack=true)
$channel->basic_consume('queue', no_ack: true);
```

### At-Least-Once (рекомендуемый)

Сообщение обрабатывается **хотя бы один раз**. Может быть обработано дублём.

```
Consumer получает сообщение → обработка → ack
Если crash ДО ack → requeue → повторное получение → обработка → ack
Результат: гарантированная обработка, возможны дубли
```

```php
// Manual ack ПОСЛЕ обработки
$callback = function (AMQPMessage $msg) {
    processMessage($msg->body); // Сначала обрабатываем
    $msg->ack();                // Потом подтверждаем
};
$channel->basic_consume('queue', no_ack: false, callback: $callback);
```

**Symfony Messenger по умолчанию использует At-Least-Once.**

### Exactly-Once (теоретически)

Сообщение обрабатывается **ровно один раз**. На практике недостижимо в распределённых системах (Two Generals' Problem). Эмулируется через idempotency.

```
At-Least-Once + Idempotent Handler = Effectively Exactly-Once
```

> ⚠️ **Важно для собеседования:** RabbitMQ **не гарантирует** Exactly-Once на уровне протокола AMQP. Причина: между обработкой сообщения и отправкой `ack` consumer может упасть → сообщение будет доставлено повторно. Единственный способ получить «эффективный Exactly-Once» — реализовать **idempotency** на стороне consumer'а (unique ID сообщения + проверка в БД/Redis перед обработкой). Kafka, в отличие от RabbitMQ, предоставляет Exactly-Once Semantics (EOS) на уровне протокола через транзакции, но даже там это работает только в рамках Kafka-to-Kafka пайплайна.

---

## 26. Паттерны: CQRS, Event Sourcing, Saga

### CQRS через Symfony Messenger

```yaml
framework:
    messenger:
        default_bus: command.bus
        
        buses:
            command.bus:
                middleware:
                    - doctrine_transaction
            query.bus:
                middleware: []
            event.bus:
                default_middleware:
                    enabled: true
                    allow_no_handlers: true
```

```php
// Command
final readonly class CreateOrder
{
    public function __construct(
        public int $customerId,
        public array $items,
    ) {}
}

// Event (после успешной команды)
final readonly class OrderCreated
{
    public function __construct(
        public int $orderId,
    ) {}
}

// Command Handler
#[AsMessageHandler(bus: 'command.bus')]
final class CreateOrderHandler
{
    public function __invoke(CreateOrder $command): void
    {
        $order = Order::create($command->customerId, $command->items);
        $this->em->persist($order);
        $this->em->flush();
        
        // Публикуем event
        $this->eventBus->dispatch(new OrderCreated($order->getId()));
    }
}

// Event Handlers (могут быть async)
#[AsMessageHandler(bus: 'event.bus')]
final class SendConfirmationEmailOnOrderCreated
{
    public function __invoke(OrderCreated $event): void { /* ... */ }
}

#[AsMessageHandler(bus: 'event.bus')]
final class UpdateInventoryOnOrderCreated
{
    public function __invoke(OrderCreated $event): void { /* ... */ }
}
```

---

## 27. Отложенные сообщения (Delayed Messages)

### Через Stamps в Symfony Messenger

```php
use Symfony\Component\Messenger\Stamp\DelayStamp;

// Отправить через 5 минут
$bus->dispatch(new SendReminder($userId), [
    new DelayStamp(300_000), // 5 минут в мс
]);

// Отправить в конкретное время
$delay = $scheduledAt->getTimestamp() - time();
$bus->dispatch(new SendScheduledEmail($emailId), [
    new DelayStamp($delay * 1000),
]);
```

### Как это работает в AMQP

Symfony Messenger использует **RabbitMQ Delayed Message Exchange Plugin** или механизм DLX + TTL.

```bash
# Установка плагина (рекомендуется)
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
```

---

## 28. Тестирование Consumer'ов

### Unit-тест Handler'а

```php
<?php

declare(strict_types=1);

use App\Message\SendOrderConfirmation;
use App\MessageHandler\SendOrderConfirmationHandler;
use PHPUnit\Framework\TestCase;

final class SendOrderConfirmationHandlerTest extends TestCase
{
    public function testSendsEmailForExistingOrder(): void
    {
        $order = new Order(id: 42, total: 99.99);
        
        $orderRepo = $this->createMock(OrderRepository::class);
        $orderRepo->method('find')->with(42)->willReturn($order);
        
        $emailSender = $this->createMock(EmailSender::class);
        $emailSender->expects($this->once())
            ->method('sendOrderConfirmation')
            ->with(
                to: 'user@example.com',
                customerName: 'Николай',
                order: $order,
            );

        $handler = new SendOrderConfirmationHandler(
            $orderRepo, $emailSender, new NullLogger()
        );

        $handler(new SendOrderConfirmation(
            orderId: 42,
            customerEmail: 'user@example.com',
            customerName: 'Николай',
        ));
    }

    public function testSkipsNonExistentOrder(): void
    {
        $orderRepo = $this->createMock(OrderRepository::class);
        $orderRepo->method('find')->willReturn(null);
        
        $emailSender = $this->createMock(EmailSender::class);
        $emailSender->expects($this->never())->method('sendOrderConfirmation');

        $handler = new SendOrderConfirmationHandler(
            $orderRepo, $emailSender, new NullLogger()
        );

        $handler(new SendOrderConfirmation(42, 'test@test.com', 'Test'));
    }
}
```

### Функциональный тест с InMemory транспортом

```yaml
# config/packages/test/messenger.yaml
framework:
    messenger:
        transports:
            async: 'in-memory://'
            async_priority_high: 'in-memory://'
```

```php
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;
use Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport;

final class OrderControllerTest extends WebTestCase
{
    public function testCreateOrderDispatchesMessages(): void
    {
        $client = static::createClient();
        
        $client->request('POST', '/api/orders', content: json_encode([
            'customer_id' => 1,
            'items' => [['product_id' => 42, 'quantity' => 2]],
        ]));
        
        $this->assertResponseStatusCodeSame(201);

        /** @var InMemoryTransport $transport */
        $transport = self::getContainer()->get('messenger.transport.async_priority_high');
        
        $messages = $transport->getSent();
        $this->assertCount(1, $messages);
        $this->assertInstanceOf(
            SendOrderConfirmation::class,
            $messages[0]->getMessage()
        );
        
        // Можно проверить содержимое
        $msg = $messages[0]->getMessage();
        $this->assertSame(1, $msg->orderId);
    }
}
```

---

## 29. Типичные проблемы и решения

### Проблема: Consumer не видит новые сообщения

```bash
# Проверить подключение
rabbitmqctl list_consumers
# Проверить что consumer активен

# Проверить очередь
rabbitmqctl list_queues name messages_ready consumers
```

### Проблема: Сообщения "застревают" в Unacked

```bash
# Показать unacked
rabbitmqctl list_queues name messages_unacknowledged

# Причина: consumer получил сообщение, но не ack'нул (hang, deadlock)
# Решение: уменьшить prefetch_count, добавить таймауты
```

### Проблема: Consumer падает с "Broken pipe"

```text
MESSENGER_TRANSPORT_DSN=amqp://guest:guest@localhost/%2f?heartbeat=60
```

Или в конфигурации Symfony:

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'
                options:
                    heartbeat: 60  # Пинг каждые 60 секунд
```

### Проблема: Очередь растёт бесконтрольно

```
Причины:
1. Consumer'ы не успевают — добавить воркеров
2. Consumer'ы падают — проверить логи
3. Сообщения крупные — оптимизировать payload
4. Handler слишком медленный — профилировать

Решения:
1. Увеличить numprocs в Supervisor
2. Исправить ошибки в handler'е
3. Уменьшить размер сообщений (передавать ID, а не данные)
4. Оптимизировать handler (кэширование, batch)
```

### Проблема: Потеря сообщений при перезапуске RabbitMQ

```
Чек-лист:
✅ Queue: durable=true
✅ Message: delivery_mode=2 (persistent)
✅ Publisher confirms (для гарантии доставки в exchange)

В Symfony Messenger durable=true и persistent=true по умолчанию.
```

---

## 30. Чек-лист для продакшена

### RabbitMQ

- [ ] Кластер (минимум 3 узла) или managed RabbitMQ
- [ ] Quorum queues вместо classic (Rabbit 3.8+)
- [ ] Мониторинг (Prometheus + Grafana)
- [ ] Алерты на queue depth, consumer count, memory
- [ ] Политики (max-length, TTL, DLX)
- [ ] Отдельный vhost для приложения
- [ ] Dedicated user (не guest!)

> 🔰 **Classic vs Quorum queues:** с RabbitMQ 3.8+ и особенно 4.x рекомендуется использовать **quorum queues** — реплицированные очереди на основе алгоритма консенсуса Raft. Они надёжнее при отказе узлов (данные сохраняются, пока живо большинство реплик), чем старые `mirrored classic queues` (которые в RabbitMQ 4.0 **удалены**). Ограничения: quorum не поддерживает priority (`x-max-priority`), `x-message-ttl` применяется по-другому, потребляют больше диска. Для большинства продакшен-очередей — выбор по умолчанию. См. [Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues).
>
> **Почему «не guest»:** пользователь `guest/guest` по умолчанию может подключаться **только с localhost** ([RabbitMQ Access Control](https://www.rabbitmq.com/docs/access-control#loopback-users)). Для production нужен отдельный пользователь с правами только на нужный vhost и минимально необходимыми конфигурационными правами.

### Consumer (Symfony Messenger)

- [ ] `--time-limit` и `--memory-limit` установлены
- [ ] Supervisor с `autorestart=true`
- [ ] `numprocs` рассчитан по нагрузке
- [ ] `stopwaitsecs` ≥ максимального времени обработки
- [ ] Failure transport настроен
- [ ] Retry strategy с exponential backoff
- [ ] `UnrecoverableMessageHandlingException` для невосстановимых ошибок
- [ ] Idempotent handlers
- [ ] Логирование (success, failure, retry)
- [ ] Graceful shutdown при деплое

### Сообщения

- [ ] Минимальный payload (ID, а не полные объекты)
- [ ] JSON-сериализация (для cross-service)
- [ ] Durable queues + persistent messages
- [ ] Prefetch count настроен

---

## 31. Источники

1. **RabbitMQ Official Documentation:**  
   https://www.rabbitmq.com/docs

2. **RabbitMQ Tutorials:**  
   https://www.rabbitmq.com/tutorials

3. **AMQP 0-9-1 Reference:**  
   https://www.rabbitmq.com/amqp-0-9-1-reference

4. **RabbitMQ Consumer Prefetch:**  
   https://www.rabbitmq.com/docs/consumer-prefetch

5. **RabbitMQ Dead Letter Exchanges:**  
   https://www.rabbitmq.com/docs/dlx

6. **RabbitMQ Reliability Guide:**  
   https://www.rabbitmq.com/docs/reliability

7. **Symfony Messenger:**  
   https://symfony.com/doc/6.4/messenger.html

8. **Symfony Messenger — AMQP Transport:**  
   https://symfony.com/doc/6.4/messenger.html#amqp-transport

9. **Symfony Messenger — Consuming Messages:**  
   https://symfony.com/doc/6.4/messenger.html#consuming-messages-running-the-worker

10. **Symfony Messenger — Retries & Failures:**  
    https://symfony.com/doc/6.4/messenger.html#retries-failures

11. **Symfony Messenger — Multiple Buses:**  
    https://symfony.com/doc/6.4/messenger/multiple_buses.html

12. **RabbitMQ Monitoring:**  
    https://www.rabbitmq.com/docs/monitoring

13. **CloudAMQP — RabbitMQ Best Practices:**  
    https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html

---

## 32. Вопросы для самопроверки

> Вопросы структурированы по уровням сложности. Каждый ответ сопровождается Obsidian-ссылкой на раздел этого же документа, где тема разбирается подробно. Щёлкните по ссылке, чтобы перейти к объяснению.

### Блок A. Основы протокола AMQP и архитектуры

> [!question]- Что такое AMQP 0-9-1 и чем он отличается от AMQP 1.0? Какую версию использует RabbitMQ?
> AMQP 0-9-1 — бинарный сетевой протокол обмена сообщениями с моделью «producer → exchange → binding → queue → consumer». AMQP 1.0 — это фактически **другой** протокол с иной моделью (peer-to-peer links, без exchange'ей как в 0-9-1). RabbitMQ «из коробки» реализует **AMQP 0-9-1**; поддержка 1.0 доступна через отдельный плагин и используется редко.
>
> 🔗 [[RabbitMQ Consumer#1. Основы RabbitMQ: протокол AMQP|Раздел 1 — Основы AMQP]]

> [!question]- Зачем нужен message broker, если можно обойтись таблицей `jobs` в БД и cron'ом?
> Ключевые преимущества брокера: **push-доставка** (задержка в миллисекундах, а не «1 минута до следующего cron»), **горизонтальное масштабирование** без блокировок в БД, встроенные **retry + DLX + priority**, **гарантии доставки** на уровне протокола (ack/nack), богатая инфраструктура мониторинга. Cron + DB допустим только для мелких проектов, где задач мало и задержка до минуты приемлема.
>
> 🔗 [[RabbitMQ Consumer#1. Основы RabbitMQ: протокол AMQP|Раздел 1 — Зачем Consumer, а не Cron + Database]]

> [!question]- В чём разница между Connection и Channel? Почему нельзя открывать по одному Connection на каждое сообщение?
> Connection — это TCP-соединение (handshake, TLS, keep-alive — дорогой ресурс). Channel — логический подканал **внутри** connection, позволяющий мультиплексировать несколько независимых AMQP-диалогов по одному TCP. Открытие нового connection на каждое сообщение создаёт огромную нагрузку на брокер (CPU на handshake, сетевые порты, файловые дескрипторы). Правильно — одно долгоживущее connection на приложение и channel'ы по мере надобности.
>
> 🔗 [[RabbitMQ Consumer#1. Основы RabbitMQ: протокол AMQP|Раздел 1 — Участники AMQP]]

> [!question]- Может ли producer публиковать напрямую в очередь, минуя exchange?
> Формально — нет. Producer всегда публикует в exchange. Синтаксис «`basic_publish(exchange='', routing_key='orders')`» работает, потому что используется **default exchange** (безымянный direct-exchange), к которому каждая очередь автоматически привязана по своему имени. Это просто сокращение.
>
> 🔗 [[RabbitMQ Consumer#7. Типы Exchange|Раздел 7 — Default Exchange]]

> [!question]- Перечислите четыре типа exchange и приведите сценарий для каждого.
> - **Direct** — точное совпадение routing_key (например, `order.created` → очередь обработки заказов).
> - **Fanout** — broadcast во все привязанные очереди (событие `user.registered` → одновременно email, analytics, CRM).
> - **Topic** — паттерны с `*` (одно слово) и `#` (ноль+ слов): `order.*.eu` или `logs.#` для иерархических событий.
> - **Headers** — маршрутизация по заголовкам сообщения, а не routing_key (например, `{region: eu, type: order, x-match: all}`).
>
> 🔗 [[RabbitMQ Consumer#7. Типы Exchange|Раздел 7 — Типы Exchange]]

### Блок B. Consumer, Acknowledgement, QoS

> [!question]- Что такое Consumer и в каком режиме он получает сообщения по умолчанию?
> Consumer — долгоживущий процесс, подписанный на очередь. Основной режим — **push (Basic.Consume)**: consumer один раз подписывается, а RabbitMQ сам «пушит» сообщения по мере поступления. Режим **pull (Basic.Get)** — ручной запрос одного сообщения, используется редко (batch-сценарии), неэффективен из-за round-trip на каждое сообщение.
>
> 🔗 [[RabbitMQ Consumer#3. Что такое Consumer и как он работает|Раздел 3 — Consumer и режимы]]

> [!question]- Объясните разницу между `basic_ack`, `basic_nack` и `basic_reject`. Когда использовать каждый?
> - `basic_ack` — «обработано успешно», удалить сообщение из очереди. Может подтвердить пачку (`multiple=true`).
> - `basic_nack` — расширение RabbitMQ. «Не обработано». `requeue=true` → вернуть в очередь (возможен бесконечный цикл!). `requeue=false` → DLX или удалить. Поддерживает `multiple=true`.
> - `basic_reject` — стандартный AMQP, аналог `nack` для **одного** сообщения (нет `multiple`).
>
> В продакшене типичный кейс: на transient ошибке — `nack(requeue=false)` вместе с настроенным DLX + retry-стратегией.
>
> 🔗 [[RabbitMQ Consumer#5. Acknowledgement (ack/nack/reject)|Раздел 5 — Acknowledgement]]

> [!question]- Что произойдёт, если consumer упадёт, не отправив ack?
> RabbitMQ отслеживает unacked-сообщения по delivery_tag в рамках channel'а. При разрыве соединения все unacked-сообщения **автоматически** возвращаются в очередь (redelivered=true) и доставляются другому (или тому же после рестарта) consumer'у. Это и есть базовый механизм гарантии **at-least-once**.
>
> 🔗 [[RabbitMQ Consumer#5. Acknowledgement (ack/nack/reject)|Раздел 5 — Что происходит при disconnect без ack]]

> [!question]- Что такое prefetch_count и почему нельзя оставлять его равным 0 (unlimited)?
> `prefetch_count` — максимум unacked-сообщений, которые RabbitMQ отправит consumer'у до того, как тот подтвердит хоть одно. При `0` (unlimited) consumer «заберёт» всю очередь себе → раздуется память, остальные consumer'ы простаивают, а при крэше 10 000 сообщений придётся redeliver'ить. Оптимум — подбирается под время обработки: тяжёлые задачи (>1с) → 1–5, средние → 10–50, лёгкие → 100+.
>
> 🔗 [[RabbitMQ Consumer#6. Prefetch Count (QoS)|Раздел 6 — Prefetch Count]]

> [!question]- В php-amqplib метод `basic_qos` принимает три параметра. Как называется третий и что означает `true`/`false`?
> Параметр называется **`$a_global`** (не `$global`, потому что `global` — зарезервированное слово PHP). Значения в RabbitMQ:
> - `false` (обычный случай) — лимит применяется **к каждому consumer'у отдельно**;
> - `true` — лимит применяется **ко всем consumer'ам channel'а суммарно**.
>
> Именованный аргумент пишется так: `basic_qos(prefetch_size: 0, prefetch_count: 10, a_global: false)`.
>
> 🔗 [[RabbitMQ Consumer#6. Prefetch Count (QoS)|Раздел 6 — Решение: prefetch_count]]

### Блок C. DLX, Retry, TTL

> [!question]- Что такое Dead Letter Exchange и в каких трёх случаях сообщение туда попадает?
> DLX — обычный exchange, куда RabbitMQ автоматически отправляет «мёртвые» сообщения. Три причины:
> 1. `nack`/`reject` с `requeue=false`;
> 2. истёк TTL сообщения (`expiration` или `x-message-ttl`);
> 3. переполнение очереди (`x-max-length` / `x-max-length-bytes`).
>
> Причина фиксируется в заголовке `x-death` — массиве с историей «смертей» сообщения.
>
> 🔗 [[RabbitMQ Consumer#8. Dead Letter Exchange (DLX)|Раздел 8 — DLX]]

> [!question]- Почему `nack(requeue=true)` — антипаттерн? Как правильно реализовать отложенный retry?
> `requeue=true` мгновенно возвращает сообщение в голову очереди — если причина ошибки не ушла, consumer снова упадёт на том же сообщении. Получается бесконечный цикл с 100% CPU. Правильно — использовать паттерн **DLX + retry-queue с TTL**: при ошибке `nack(requeue=false)` → сообщение попадает в `retry_queue` с `x-message-ttl=30000` и `x-dead-letter-exchange=main` → через 30 секунд RabbitMQ автоматически возвращает его обратно в main. Для экспоненциального backoff — несколько retry-очередей с разными TTL.
>
> 🔗 [[RabbitMQ Consumer#10. Retry-стратегии|Раздел 10 — Retry-стратегии]]

> [!question]- Чем различаются `expiration` на сообщении и `x-message-ttl` на очереди?
> - `x-message-ttl` (аргумент очереди, **int**, миллисекунды) — применяется ко всем сообщениям, проверяется на голове очереди при попытке доставки.
> - `expiration` (свойство сообщения, **строка** с числом миллисекунд!) — индивидуальный TTL, проверяется при попытке доставки consumer'у.
>
> Если указаны оба — действует меньший. Типовой баг: передать `expiration` числом — библиотека ожидает строку.
>
> 🔗 [[RabbitMQ Consumer#9. TTL, Expiration, Priority|Раздел 9 — TTL]]

> [!question]- Почему приоритетные очереди в RabbitMQ — не всегда лучший выбор?
> `x-max-priority` потребляет больше памяти/CPU (внутри — суб-очереди на каждый приоритет), **несовместим с quorum queues**, и на больших объёмах уступает по пропускной способности обычным очередям. Рекомендуемая альтернатива — две отдельные очереди (`high`, `low`) и запуск `messenger:consume async_high async_low` — worker всегда обрабатывает high раньше low.
>
> 🔗 [[RabbitMQ Consumer#9. TTL, Expiration, Priority|Раздел 9 — Priority Queues]]

### Блок D. Symfony Messenger

> [!question]- Какую роль играют MessageBus, Transport и Handler в Symfony Messenger?
> - **MessageBus** — центральная точка, куда приложение «диспатчит» сообщения. Пропускает их через цепочку **middleware**.
> - **Transport** — адаптер, который сериализует сообщение и отправляет в хранилище (AMQP, Doctrine, Redis, in-memory). При consume — наоборот, читает и десериализует.
> - **Handler** — класс с атрибутом `#[AsMessageHandler]`, содержащий бизнес-логику. Вызывается, когда worker получает сообщение из transport и прогоняет через `HandleMessageMiddleware`.
>
> 🔗 [[RabbitMQ Consumer#11. Symfony Messenger + RabbitMQ|Раздел 11 — Архитектура Symfony Messenger]]

> [!question]- Что такое Envelope и Stamp? Приведите 3–4 примера встроенных Stamp'ов.
> **Envelope** — «конверт», в который Messenger оборачивает message при диспатче. На конверт навешиваются **Stamp'ы** — иммутабельные метаданные:
> - `BusNameStamp` — имя шины.
> - `TransportNamesStamp` — в какой транспорт отправить (переопределяет routing).
> - `DelayStamp($ms)` — отложенная доставка.
> - `RedeliveryStamp` — номер попытки retry.
> - `AmqpStamp` — AMQP-специфичные параметры (routing_key, exchange, priority).
> - `HandledStamp` — результат handler'а.
>
> Stamp'ы — способ прокинуть сквозную метаинформацию через middleware, не меняя сам message.
>
> 🔗 [[RabbitMQ Consumer#13. Создание Message и Handler|Раздел 13 — Envelope и Stamp]]

> [!question]- Что случится с сообщением после превышения `max_retries`? Как это настроить?
> Если retry-стратегия исчерпана, Messenger перекладывает сообщение в **failure_transport** (обычно Doctrine-транспорт с таблицей `messenger_messages`). Управление: `messenger:failed:show`, `messenger:failed:retry <id>`, `messenger:failed:remove <id>`. Настраивается в `framework.messenger.failure_transport` и блоком `retry_strategy` у транспорта (`max_retries`, `delay`, `multiplier`, `max_delay`).
>
> 🔗 [[RabbitMQ Consumer#16. Failure Transport (обработка ошибок)|Раздел 16 — Failure Transport]]

> [!question]- Когда использовать `UnrecoverableMessageHandlingException`, а когда обычное исключение?
> - **Обычное исключение** (сетевой таймаут, временная недоступность API, deadlock) — Messenger применит retry-стратегию с backoff. Используется для **transient** ошибок.
> - **`UnrecoverableMessageHandlingException`** — для **permanent** ошибок (битый payload, нарушение бизнес-правила, отсутствует сущность). Сообщение **сразу** уходит в failure_transport, минуя все retries. Это экономит ресурсы и ускоряет алертинг.
>
> 🔗 [[RabbitMQ Consumer#17. Retry механизм в Symfony Messenger|Раздел 17 — Отключение retry для конкретных исключений]]

> [!question]- Чем отличается PHP-сериализация от JSON-сериализации в Messenger? Когда выбирать каждую?
> - **PhpSerializer** (по умолчанию) — `serialize()/unserialize()`. Быстрый, сохраняет типы, но **непереносим** между языками и чувствителен к рефакторингу (переименовали класс → `unserialize` упадёт на старых сообщениях в очереди).
> - **Symfony Serializer (JSON)** — человекочитаемый формат, переносим между сервисами на разных языках, но требует соответствия структуры class ↔ JSON (нужен `type` в заголовках для десериализации).
>
> Выбор: внутри одного PHP-монолита — PHP serializer, для cross-service (polyglot microservices) — JSON.
>
> 🔗 [[RabbitMQ Consumer#19. Serialization сообщений|Раздел 19 — Serialization]]

> [!question]- Зачем опции `--time-limit` и `--memory-limit` у команды `messenger:consume`?
> Consumer — долгоживущий PHP-процесс. PHP плохо «чистит» память (Identity Map Doctrine, кэши, логгеры), со временем потребление растёт. `--time-limit=3600` заставит воркер корректно завершиться раз в час, `--memory-limit=256` — при превышении 256 МБ. Supervisor тут же перезапустит worker «чистым». Это типовая защита от memory leaks.
>
> 🔗 [[RabbitMQ Consumer#21. Memory Leaks и долгоживущие процессы|Раздел 21 — Memory Leaks]]

### Блок E. Прод, масштабирование, надёжность

> [!question]- Как реализуется graceful shutdown в Symfony Messenger? Какие сигналы он обрабатывает?
> Messenger через `pcntl_signal` ловит `SIGTERM` и `SIGINT`: если в момент сигнала воркер обрабатывает сообщение, он **дожидается** завершения обработки, отправляет ack/nack и только потом выходит. Supervisor/K8s должны давать достаточный `stopwaitsecs` / `terminationGracePeriodSeconds` (≥ максимального времени handler'а + запас). Команда `messenger:stop-workers` кладёт сигнал в shared cache — воркеры проверяют его между сообщениями (нужен общий cache, например Redis).
>
> 🔗 [[RabbitMQ Consumer#20. Graceful Shutdown и сигналы|Раздел 20 — Graceful Shutdown]]

> [!question]- Как обеспечить, чтобы сообщения не терялись при рестарте RabbitMQ?
> Нужны **три** условия одновременно:
> 1. Queue объявлена с `durable=true` — метаданные очереди сохранятся на диск.
> 2. Сообщение опубликовано с `delivery_mode=2` (persistent) — содержимое будет записано на диск.
> 3. Publisher confirms (`confirm_select` + ожидание ACK от брокера) — producer получает подтверждение, что брокер принял сообщение.
>
> В Symfony Messenger AMQP-транспорт по умолчанию использует `durable=true` и `persistent=true`. Для максимальной надёжности — quorum queues и кластер из 3+ узлов.
>
> 🔗 [[RabbitMQ Consumer#4. Жизненный цикл сообщения|Раздел 4 — Persistent Messages]]

> [!question]- Что такое идемпотентность и почему она обязательна при at-least-once доставке?
> Идемпотентность — свойство операции давать одинаковый результат при многократном применении с теми же входными данными. При at-least-once сообщение может прийти **дважды** (consumer упал **после** обработки, но **до** ack → сообщение redeliver'ят). Если handler не идемпотентен — возникнут двойные списания, дубли писем и т. п. Реализация: уникальный `idempotency_key` в сообщении + таблица `processed_messages` (или Redis SETNX) + атомарная запись ключа вместе с бизнес-операцией в одной транзакции.
>
> 🔗 [[RabbitMQ Consumer#24. Идемпотентность обработки сообщений|Раздел 24 — Идемпотентность]]

> [!question]- Почему Exactly-Once невозможно «честно» в распределённой системе? Как получается «effectively exactly-once»?
> Проблема двух генералов (Two Generals' Problem): нет способа надёжно договориться о факте доставки по ненадёжному каналу конечным числом сообщений. Между handler'ом и ack'ом всегда есть окно, где consumer может упасть → сообщение доставится ещё раз. RabbitMQ на уровне протокола гарантирует только at-least-once (или at-most-once при auto-ack). **Effectively exactly-once** = at-least-once + идемпотентный handler: дубли физически возможны, но бизнес-эффект применяется ровно один раз.
>
> 🔗 [[RabbitMQ Consumer#25. Гарантии доставки|Раздел 25 — Гарантии доставки]]

> [!question]- Чем classic queues отличаются от quorum queues? Когда какую выбрать?
> - **Classic** — оригинальные очереди RabbitMQ, работают на одном узле; репликация — через устаревшие `mirrored queues` (удалены в RabbitMQ 4.0). Поддерживают priority, простые в использовании.
> - **Quorum** (3.8+) — реплицированные очереди на алгоритме Raft, живут, пока живо большинство реплик; рекомендованы для production. Ограничения: **не поддерживают** `x-max-priority`, по-другому работают с TTL, потребляют больше диска.
>
> Правило: всё критичное (платежи, заказы) — quorum; быстрые эфемерные задачи (например, websocket presence) — classic.
>
> 🔗 [[RabbitMQ Consumer#30. Чек-лист для продакшена|Раздел 30 — Чек-лист: RabbitMQ]]

> [!question]- Как масштабировать consumer'ов по глубине очереди в Kubernetes?
> Через **HorizontalPodAutoscaler** на внешнюю метрику `rabbitmq_queue_messages_ready` (экспортируется `rabbitmq_prometheus` плагином). HPA реагирует на `messages_ready / pods > threshold` и добавляет pod'ы. Для этого нужен **Prometheus Adapter** (или KEDA с RabbitMQ-scaler'ом — более популярное решение), чтобы превратить метрику Prometheus в `external metric` Kubernetes API. `terminationGracePeriodSeconds` должен быть достаточен для graceful shutdown worker'а.
>
> 🔗 [[RabbitMQ Consumer#22. Consumer в Docker и Kubernetes|Раздел 22 — HPA]]

> [!question]- Какие метрики RabbitMQ обязательны для алертинга в проде?
> - `rabbitmq_queue_messages_ready` — рост = consumer'ы не успевают.
> - `rabbitmq_queue_messages_unacked` — застрявшие у consumer'а сообщения (hang/медленный handler).
> - `rabbitmq_queue_consumers == 0` — **critical**: нет воркеров.
> - `rabbitmq_resident_memory_used` > 80% watermark — брокер скоро заблокирует publish.
> - `rabbitmq_disk_free` < лимита — риск остановки кластера.
> - `rabbitmq_connections / channels` — утечки клиентов.
>
> 🔗 [[RabbitMQ Consumer#23. Мониторинг RabbitMQ и Consumer'ов|Раздел 23 — Мониторинг]]

> [!question]- Зачем нужны две очереди (high_priority + low_priority) и порядок транспортов в `messenger:consume`?
> `messenger:consume async_high async_low` — worker сначала опустошит первую очередь, и только когда она пуста — пойдёт во вторую. Это даёт приоритизацию **без** использования x-max-priority (которое несовместимо с quorum и тяжелее по ресурсам). Альтернатива — отдельные пулы worker'ов для разных типов нагрузки, чтобы медленные newsletter'ы не блокировали критичные подтверждения заказов.
>
> 🔗 [[RabbitMQ Consumer#18. Multiple Consumers и приоритетные очереди|Раздел 18 — Multiple Consumers]]

> [!question]- Что именно делает Doctrine-middleware в Messenger и зачем?
> - `doctrine_ping_connection` — перед обработкой проверяет/восстанавливает соединение с БД (закроется ли оно по таймауту между сообщениями).
> - `doctrine_close_connection` — после обработки закрывает соединение (полезно для транспортов с редкими сообщениями).
> - `doctrine_transaction` — оборачивает handler в DB-транзакцию: коммит при успехе, rollback при исключении.
>
> В Symfony 6.4 `doctrine_ping_connection` добавлен в worker'ы автоматически — защищает от ошибки «MySQL server has gone away».
>
> 🔗 [[RabbitMQ Consumer#21. Memory Leaks и долгоживущие процессы|Раздел 21 — Memory Leaks и долгоживущие процессы]]

> [!question]- Как отложить сообщение на 5 минут? Что нужно со стороны RabbitMQ?
> В коде — `$bus->dispatch($message, [new DelayStamp(300_000)])` (миллисекунды). Под капотом AMQP-транспорт Symfony Messenger использует **DLX + TTL**: создаёт delay-exchange и delay-queue с нужным TTL, через который сообщение возвращается в целевой exchange. Более простой и масштабируемый вариант — включить плагин `rabbitmq_delayed_message_exchange` и настроить специальный тип exchange. Оба варианта поддерживаются Messenger'ом; плагин рекомендуется при большом количестве разных delay.
>
> 🔗 [[RabbitMQ Consumer#27. Отложенные сообщения (Delayed Messages)|Раздел 27 — Delayed Messages]]

> [!question]- Ошибка «Broken pipe» / «connection closed unexpectedly» у worker'а — в чём причина и как лечить?
> Причина: TCP-соединение между consumer'ом и RabbitMQ закрылось по idle-таймауту (часто на L4-балансере, NAT или самим брокером). Лечение — **heartbeat**: `?heartbeat=60` в DSN или `options.heartbeat: 60`. Клиент и сервер периодически обмениваются «ping»-фреймами → соединение считается живым. Рекомендованное значение — половина от idle-timeout инфраструктуры (обычно 30–60 секунд).
>
> 🔗 [[RabbitMQ Consumer#29. Типичные проблемы и решения|Раздел 29 — Consumer падает с Broken pipe]]

> [!question]- Почему в `livenessProbe` Kubernetes **нельзя** запускать `bin/console messenger:stats`?
> Каждый вызов `bin/console` — это **полная загрузка Symfony kernel'а** (autoload, DI-контейнер, конфиги). Делать это каждые 30–60 секунд = постоянный оверхед по CPU и памяти, плюс риск false-positive при временной медленности. Правильные подходы:
> - `pgrep -f 'messenger:consume'` — убедиться, что процесс жив.
> - Worker пишет timestamp «последней обработки» в файл/shared-memory → probe читает и сравнивает с текущим временем.
> - Использовать `livenessProbe` очень консервативно; для consumer'ов часто достаточно `restartPolicy: Always` + graceful shutdown.
>
> 🔗 [[RabbitMQ Consumer#22. Consumer в Docker и Kubernetes|Раздел 22 — Kubernetes Deployment]]

### Блок F. Проверка понимания (ситуационные)

> [!question]- Очередь растёт на 1000 сообщений/минуту, consumer'ы не успевают. Алгоритм диагностики?
> 1. **Метрики:** `messages_ready` растёт, `messages_unacked` в норме → producer опережает consumer'ов.
> 2. **Consumer throughput:** сколько ack/sec? Сравнить с publish rate в Management UI.
> 3. **Handler профилирование:** Blackfire/XHProf — найти bottleneck (запросы к БД, внешний API).
> 4. **Горизонтальное масштабирование:** увеличить `numprocs` в Supervisor или replicas в K8s.
> 5. **prefetch_count:** если slow handler — уменьшить до 1–5; если fast — увеличить до 50+.
> 6. **Размер payload:** передавать ID, а не полные сущности.
> 7. **Batching:** если handler делает по 1 INSERT — объединить в `INSERT ... VALUES (...), (...), (...)`.
>
> 🔗 [[RabbitMQ Consumer#29. Типичные проблемы и решения|Раздел 29 — Очередь растёт бесконтрольно]]

> [!question]- После деплоя worker'ы стали падать с «Class App\Message\OldEvent not found». Что случилось и как предотвратить?
> В очереди лежали сообщения, сериализованные **старым кодом** (PhpSerializer), а после деплоя класс `OldEvent` удалили/переименовали. Профилактика:
> 1. Деплой — **двухфазный**: сначала раскатываем новый код, который умеет обрабатывать и старые, и новые сообщения, ждём очистки очереди, потом удаляем старый класс.
> 2. Использовать **JSON Serializer** + явное поле `type` в заголовке → можно роутить на нужный handler, старые сообщения не падают на unserialize.
> 3. Для критичных событий — версионирование (`UserRegisteredV1`, `UserRegisteredV2`).
> 4. Перед удалением класса — `messenger:stats`, чтобы убедиться, что очередь пуста.
>
> 🔗 [[RabbitMQ Consumer#19. Serialization сообщений|Раздел 19 — Serialization сообщений]]

> [!question]- Handler платежа: как обеспечить, чтобы клиент не был списан дважды, даже если RabbitMQ redeliver'ит сообщение?
> Комбинация трёх приёмов:
> 1. **Idempotency key** в сообщении (`payment_order_<id>_<uuid>`).
> 2. **Таблица `processed_messages`** с уникальным индексом на ключ; запись в неё + списание средств — **в одной транзакции БД**. При повторе сработает unique violation → ловим, возвращаем ack без повторной операции.
> 3. Либо — делегирование идемпотентности платёжному шлюзу (передавать ему тот же `idempotency_key` — почти все современные PSP это поддерживают). Шлюз гарантирует, что повторный запрос не приведёт к повторному списанию.
> 4. `UnrecoverableMessageHandlingException` на permanent-ошибках (карта отклонена) — чтобы не наматывать retries.
>
> 🔗 [[RabbitMQ Consumer#24. Идемпотентность обработки сообщений|Раздел 24 — Идемпотентность]]

---
