RabbitMQ ‒ это брокер сообщений. Его основная цель ‒ принимать и отдавать сообщения.

**Producer** (поставщик) ‒ программа, отправляющая сообщения.
![[Pasted image 20250418212852.png]]
**Queue** (очередь) ‒ имя «почтового ящика». Она существует внутри RabbitMQ. Хотя сообщения проходят через RabbitMQ и приложения, хранятся они только в очередях. Очередь не имеет ограничений на количество сообщений, она может принять сколь угодно большое их количество ‒ можно считать ее бесконечным буфером. Любое количество поставщиков может отправлять сообщения в одну очередь, также любое количество подписчиков может получать сообщения из одной очереди.
![[Pasted image 20250418212931.png]]
**Consumer** (подписчик) ‒ программа, принимающая сообщения. Обычно подписчик находится в состоянии ожидания сообщений.
![[Pasted image 20250418212954.png]]

```
sudo rabbitmqctl list_queues // Какие очереди существуют в RabbitMQ
//апнутый скрипт
watch 'sudo /usr/sbin/rabbitmqctl list_queues name messages_unacknowledged messages_ready messages durable auto_delete consumers | grep -v "\.\.\." | sort | column -t;'
```

#### Установка в Symfony
composer require php-amqplib/rabbitmq-bundle

### **Подключение к RabbitMQ (connections)**
```yaml
connections:
    default:
        host: '%env(RABBITMQ_HOST)%'
        port: '%env(RABBITMQ_PORT)%'
        user: '%env(RABBITMQ_USER)%'
        password: '%env(RABBITMQ_PASSWORD)%'
        vhost: '%env(RABBITMQ_VHOST)%'
```
### **2. Producer (отправитель сообщений)**
```yaml
producers:
    task_producer:
        connection: default
        exchange_options: {name: '', type: direct}
```
- `task_producer` — это сервис, который будет отправлять сообщения в RabbitMQ.
- `exchange_options`:
    - `name: ''` — используется default exchange (как в оригинальном коде).
    - `type: direct` — тип обменника (но для default exchange это не имеет значения).
      
### **Consumer (получатель сообщений)**
```yaml
consumers:
    task_worker:
        connection: default
        exchange_options: {name: '', type: direct}
        queue_options: {name: 'task_queue', durable: true}
        callback: App\MessageHandler\TaskProcessor
        qos_options: {prefetch_size: 0, prefetch_count: 1, global: false}
```

- `name: 'task_queue'` — имя очереди (как в оригинальном коде).
- `durable: true` — очередь будет сохраняться после перезагрузки RabbitMQ
- App\MessageHandler\TaskProcessor - Указывает класс, который будет обрабатывать сообщения (аналог функции `$callback` в оригинальном коде).
- `prefetch_count: 1` — Worker получит только 1 сообщение за раз и не будет брать новое, пока не подтвердит обработку текущего (fair dispatch).
  
### **Введение в Publish/Subscribe паттерн**

В Work Queue каждое задание доставлялось ровно одному работнику (worker), а в Publish/Subscribe сообщение будет доставляться множеству потребителей (consumers)
## Основные концепции

### Exchange (Обменник)
Основная идея модели сообщений в RabbitMQ:
- **Producer (Продюсер)** - приложение, которое отправляет сообщения
- **Queue (Очередь)** - буфер, который хранит сообщения
- **Consumer (Консьюмер)** - приложение, которое получает сообщения
**Главное отличие**: продюсер никогда не отправляет сообщения напрямую в очередь. Часто продюсер даже не знает, будет ли сообщение доставлено в какую-либо очередь 3.
Вместо этого продюсер может отправлять сообщения только в **exchange (обменник)**. Обменник - это очень простая сущность:
- С одной стороны он получает сообщения от продюсеров
- С другой стороны он отправляет эти сообщения в очереди

Обменник должен точно знать, что делать с полученным сообщением
- Добавить его в конкретную очередь?
- Добавить в несколько очередей?
- Или отбросить?

Правила для этого определяются **типом обменника**
### Типы Exchange
Существует несколько типов обменников:
1. `direct` - сообщение отправляется в очередь с точно совпадающим routing key
2. `topic` - сообщение отправляется в очередь, где routing key соответствует шаблону
3. `headers` - сообщение отправляется в очередь на основе заголовков
4. `fanout` - сообщение отправляется во все связанные очереди
ПРИМЕР:
```yaml
producers:
        logs_producer:
            connection: default
            exchange_options: {name: 'logs', type: fanout}
    consumers:
        logs_consumer:
            connection: default
            exchange_options: {name: 'logs', type: fanout}
            queue_options: {name: 'logs_queue', durable: false, auto_delete: true}
            callback: App\MessageHandler\LogsHandler
```

[Publisher] --> (Exchange) --> [Queue 1] --> [Subscriber 1]
            \               \-> [Queue 2] --> [Subscriber 2]
             \-> [Queue 3] --> [Subscriber 3]

Каждый subscriber получает свою копию сообщения, как разные люди слушают одну и ту же радиопередачу.