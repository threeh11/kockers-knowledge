> Apache Avro — это система для настройки вызова удалённых процедур и сериализации данных, созданная в рамках проекта Apache Hadoop в 2009 году.

Она использует JSON для определения типов данных и протоколов, а сами данные преобразует в компактный бинарный формат. Ниже представлена схема JSON-формата.
![[Pasted image 20250519233459.png]]
![[Pasted image 20250519233709.png]]
![[Pasted image 20250519233732.png]]
![[Pasted image 20250519233752.png]]
Schema registry — это веб-сервер, разработанный на Java и входящий в платформу Confluent CP. Он предоставляет REST API для управления схемами: добавления новых, получения версий, проверки совместимости, удаления и других манипуляций.

# Apache Avro + Symfony Integration

## Установка зависимостей
```bash
composer require flix-tech/avro-serde-php php-avro/php-avro
```

## Схема Avro (`config/avro/schemas/user.avsc`)
```json
{
  "type": "record",
  "name": "User",
  "namespace": "app.avro",
  "fields": [
    {"name": "id", "type": "int"},
    {"name": "username", "type": "string"},
    {"name": "email", "type": "string"},
    {"name": "createdAt", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "isActive", "type": "boolean", "default": true}
  ]
}
```

## Сервис AvroSerializer (`src/Service/AvroSerializer.php`)

```PHP
<?php

namespace App\Service;

use FlixTech\AvroSerializer\Objects\RecordSerializer;
use FlixTech\SchemaRegistryApi\Registry\CacheRegistry;
use FlixTech\SchemaRegistryApi\Registry\CachedRegistry;
use FlixTech\SchemaRegistryApi\Registry\PromisingRegistry;
use GuzzleHttp\Client;

class AvroSerializer
{
    private RecordSerializer $serializer;
    private CachedRegistry $registry;

    public function __construct(string $schemaRegistryUrl)
    {
        $client = new Client(['base_uri' => $schemaRegistryUrl]);
        $this->registry = new CacheRegistry(new PromisingRegistry($client));
        $this->serializer = new RecordSerializer($this->registry);
    }

    public function encode(string $subject, string $avroSchemaPath, array $data): string
    {
        $avroSchema = \AvroSchema::parse(file_get_contents($avroSchemaPath));
        return $this->serializer->encodeRecord($subject, $avroSchema, $data);
    }

    public function decode(string $subject, string $avroSchemaPath, string $avroEncodedData): array
    {
        $avroSchema = \AvroSchema::parse(file_get_contents($avroSchemaPath));
        return $this->serializer->decodeRecord($subject, $avroSchema, $avroEncodedData);
    }
}
```

## Конфигурация сервисов (`config/services.yaml`)

```yaml
services:
  App\Service\AvroSerializer:
    arguments:
      $schemaRegistryUrl: '%env(SCHEMA_REGISTRY_URL)%'
  
  avro.user.schema: '%kernel.project_dir%/config/avro/schemas/user.avsc'
```

## Пример контроллера

```PHP
<?php

namespace App\Controller;

use App\Service\AvroSerializer;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;

class AvroExampleController extends AbstractController
{
    private AvroSerializer $avroSerializer;
    private string $userSchemaPath;

    public function __construct(AvroSerializer $avroSerializer, string $avro.user.schema)
    {
        $this->avroSerializer = $avroSerializer;
        $this->userSchemaPath = $avro.user.schema;
    }

    #[Route('/avro/encode', methods: ['POST'])]
    public function encodeExample(Request $request): JsonResponse
    {
        $userData = [
            'id' => 123,
            'username' => 'john_doe',
            'email' => 'john@example.com',
            'createdAt' => (new \DateTime())->getTimestamp() * 1000,
            'isActive' => true
        ];

        $avroEncoded = $this->avroSerializer->encode('users', $this->userSchemaPath, $userData);
        
        return $this->json([
            'base64_encoded' => base64_encode($avroEncoded),
            'hex_encoded' => bin2hex($avroEncoded)
        ]);
    }

    #[Route('/avro/decode', methods: ['POST'])]
    public function decodeExample(Request $request): JsonResponse
    {
        $avroEncoded = base64_decode($request->request->get('avro_data'));
        $decodedData = $this->avroSerializer->decode('users', $this->userSchemaPath, $avroEncoded);
        
        return $this->json($decodedData);
    }
}
```

## Пример для работы с Kafka

```PHP
<?php

namespace App\Service;

use Enqueue\RdKafka\RdKafkaConnectionFactory;

class KafkaAvroProducer
{
    private RdKafkaConnectionFactory $connectionFactory;
    private AvroSerializer $avroSerializer;
    private string $userSchemaPath;

    public function __construct(
        AvroSerializer $avroSerializer,
        string $avro.user.schema,
        string $kafkaBrokers
    ) {
        $this->avroSerializer = $avroSerializer;
        $this->userSchemaPath = $avro.user.schema;
        $this->connectionFactory = new RdKafkaConnectionFactory([
            'global' => [
                'group.id' => 'my_group',
                'metadata.broker.list' => $kafkaBrokers,
                'enable.auto.commit' => 'false'
            ],
            'topic' => [
                'auto.offset.reset' => 'earliest'
            ]
        ]);
    }

    public function sendUserMessage(array $userData, string $topic): void
    {
        $context = $this->connectionFactory->createContext();
        $avroEncoded = $this->avroSerializer->encode('users', $this->userSchemaPath, $userData);
        
        $message = $context->createMessage($avroEncoded);
        $message->setKey((string)$userData['id']);
        
        $topic = $context->createTopic($topic);
        $context->createProducer()->send($topic, $message);
    }
}
```

## Конфигурация окружения (`.env`)
```ENV
SCHEMA_REGISTRY_URL=http://localhost:8081
```

### Как развернуть Schema Registry

#### 1. Вариант через Docker (самый простой)

Добавьте в `docker-compose.yml`:
```YAML
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    depends_on:
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: "kafka:9092"
    ports:
      - "8081:8081"
```

### 4. Производительность

**Прирост:**
- Бинарный формат Avro + Registry дает до **60% экономии** трафика по сравнению с JSON
- В 5-10 раз меньше CPU нагрузка при сериализации
- В 2-3 раза меньше памяти чем Protocol Buffers

**Тестовые данные:**

|Формат|Размер 1000 сообщений|CPU usage|
|---|---|---|
|JSON|1.8 MB|100%|
|Avro|0.4 MB|23%|
|Avro+SR|0.38 MB (+ID схемы)|25%|

### 5. Безопасность
- Гарантия, что потребитель получит данные в ожидаемом формате
- Защита от "схемного дрифта" (когда схемы незаметно расходятся)
- Аудит изменений (кто, когда и как поменял схему)