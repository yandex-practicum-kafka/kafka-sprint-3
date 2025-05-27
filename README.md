1. Подготовка: Создание проекта с `docker-compose.yml`  

Создайте директорию для проекта, например `kafka-partition-reassignment`. В ней создайте файл `docker-compose.yml` со следующим содержанием:  

```
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka-1:
    image: confluentinc/cp-kafka:latest
    hostname: kafka-1
    container_name: kafka-1
    ports:
      - "9092:9092"
      - "9991:9991"  # JMX Port
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: kafka-1:29092
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'

  kafka-2:
    image: confluentinc/cp-kafka:latest
    hostname: kafka-2
    container_name: kafka-2
    ports:
      - "9093:9093"
      - "9992:9991"  # JMX Port
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29092,PLAINTEXT_HOST://localhost:9093
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: kafka-2:29092
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'

  kafka-3:
    image: confluentinc/cp-kafka:latest
    hostname: kafka-3
    container_name: kafka-3
    ports:
      - "9094:9094"
      - "9993:9991" # JMX Port
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29092,PLAINTEXT_HOST://localhost:9094
      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: kafka-3:29092
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'

  kafka-setup:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    entrypoint: ["/bin/sh", "-c"]
    command: |
      "
      # Wait for Kafka brokers to be ready
      kafka-topics --bootstrap-server kafka-1:29092 --list

      # Create the '__consumer_offsets' topic with desired replication factor.
      kafka-topics --bootstrap-server kafka-1:29092 --create --if-not-exists --topic __consumer_offsets --partitions 50 --replication-factor 3

      # Create the 'balanced_topic' with the specified partitions and replication factor.
      kafka-topics --bootstrap-server kafka-1:29092 --create --if-not-exists --topic balanced_topic --partitions 8 --replication-factor 3
      "
    volumes:
      - ./reassignment.json:/tmp/reassignment.json

  kafka-cli:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - kafka-1
    entrypoint: ["/bin/sh", "-c"]
    tty: true
    stdin_open: true
```  
Описание `docker-compose.yml`:  

•  `zookeeper`: Один экземпляр ZooKeeper.  
•  `kafka-1, kafka-2, kafka-3`: Три брокера Kafka. Важно, чтобы KAFKA_BROKER_ID был уникальным для каждого брокера.   `KAFKA_ADVERTISED_LISTENERS` определяет, как брокер объявляет о себе клиентам. PLAINTEXT использует внутреннее имя (например, `kafka-1`), а `PLAINTEXT_HOST` использует `localhost`, чтобы вы могли подключаться к Kafka извне Docker сети.   `CONFLUENT_METRICS_REPORTER_*` настраивают Confluent Metrics Reporter для мониторинга.  
•  `kafka-setup`: Сервис, который создает топик `balanced_topic` и `topic __consumer_offsets` при старте кластера. kafka-topics используется для создания топиков с 8 партициями и фактором репликации 3. Монтируется `reassignment.json` в контейнер.  
•  `kafka-cli`: Контейнер с Kafka CLI для выполнения команд. Это удобный способ взаимодействия с кластером Kafka изнутри Docker.  

Важно:  

•  Убедитесь, что порты (`9092, 9093, 9094`) не заняты на вашей машине.  
•  Этот пример использует `localhost` для доступа к Kafka извне контейнеров. В production-среде вам потребуется настроить `KAFKA_ADVERTISED_LISTENERS` и DNS/сеть соответствующим образом.  

2. Запуск кластера Kafka  

В терминале перейдите в директорию, где находится docker-compose.yml, и выполните:  

```
docker-compose up -d
```

Это запустит Zookeeper и три брокера Kafka в detached режиме (в фоновом режиме).  

3. Создание топика `balanced_topic`  

Топик `balanced_topic` создается автоматически сервисом `kafka-setup` при старте кластера. Убедитесь, что топик создан успешно, выполнив следующую команду в контейнере `kafka-cli`:  

```
docker-compose exec kafka-cli kafka-topics --bootstrap-server kafka-1:29092 --list
```

Вы должны увидеть в списке топиков `balanced_topic` и `__consumer_offsets`.  

4. Определение текущего распределения партиций  

Используем `kafka-topics` с опцией `--describe` для получения информации о распределении партиций топика `balanced_topic`. Выполните следующую команду в контейнере `kafka-cli`:  

```
docker-compose exec kafka-cli kafka-topics --bootstrap-server kafka-1:29092 --describe --topic balanced_topic
```

Вывод будет похож на:  

```
Topic: balanced_topic PartitionCount: 8 ReplicationFactor: 3 Configs:
 Topic: balanced_topic Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
 Topic: balanced_topic Partition: 1 Leader: 2 Replicas: 2,3,1 Isr: 2,3,1
 Topic: balanced_topic Partition: 2 Leader: 3 Replicas: 3,1,2 Isr: 3,1,2
 Topic: balanced_topic Partition: 3 Leader: 1 Replicas: 1,3,2 Isr: 1,3,2
 Topic: balanced_topic Partition: 4 Leader: 2 Replicas: 2,1,3 Isr: 2,1,3
 Topic: balanced_topic Partition: 5 Leader: 3 Replicas: 3,2,1 Isr: 3,2,1
 Topic: balanced_topic Partition: 6 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
 Topic: balanced_topic Partition: 7 Leader: 2 Replicas: 2,3,1 Isr: 2,3,1

```

Эта информация показывает:  

•  `PartitionCount`: Количество партиций (8).  
•  `ReplicationFactor`: Фактор репликации (3).  
•  `Leader`: Брокер, который является лидером для данной партиции.  
•  `Replicas`: Список брокеров, содержащих реплики данной партиции. Первый брокер в списке - лидер.  
•  `Isr`: Список брокеров, входящих в In-Sync Replicas (ISR) – реплики, которые синхронизированы с лидером и готовы к принятию роли лидера в случае его отказа.  

5. Создание JSON-файла `reassignment.json` для перераспределения партиций  

Этот файл определяет желаемое новое распределение партиций. Создайте файл reassignment.json в той же директории, что и `docker-compose.yml`. Вот пример содержимого:  

```
{
  "version": 1,
  "partitions": [
    {"topic": "balanced_topic", "partition": 0, "replicas": [2, 3, 1]},
    {"topic": "balanced_topic", "partition": 1, "replicas": [3, 1, 2]},
    {"topic": "balanced_topic", "partition": 2, "replicas": [1, 2, 3]},
    {"topic": "balanced_topic", "partition": 3, "replicas": [2, 1, 3]},
    {"topic": "balanced_topic", "partition": 4, "replicas": [3, 2, 1]},
    {"topic": "balanced_topic", "partition": 5, "replicas": [1, 3, 2]},
    {"topic": "balanced_topic", "partition": 6, "replicas": [2, 3, 1]},
    {"topic": "balanced_topic", "partition": 7, "replicas": [3, 1, 2]}
  ]
}
```

Важно:  

•  `version`: Указывает версию формата файла.  
•  `partitions`: Массив, содержащий информацию о каждой партиции, которую вы хотите перераспределить.  
  •  `topic`: Имя топика.  
  •  `partition`: Номер партиции.  
  •  `replicas`: Список брокеров, которые должны содержать реплики этой партиции в указанном порядке (первый брокер будет лидером).
•  Убедитесь, что все брокеры, указанные в replicas, существуют в вашем кластере. В данном примере используются брокеры с ID 1, 2 и 3.  

Примечание: В этом примере мы меняем порядок реплик, чтобы более равномерно распределить лидерство между брокерами. Вы можете настроить `reassignment.json` в соответствии с вашими потребностями.  

6. Генерация плана перераспределения  

Прежде чем выполнять перераспределение, сгенерируйте план, используя --generate. Это позволит вам увидеть, какие изменения будут внесены. Выполните следующую команду в контейнере `kafka-cli`:  

```
docker-compose exec kafka-cli kafka-reassign-partitions --bootstrap-server kafka-1:29092 --generate --topics-to-move-json-file /tmp/reassignment.json --broker-list "1,2,3" --out-file /tmp/reassignment.json
```

Эта команда:  

•  `kafka-reassign-partitions`: Инструмент для перераспределения партиций.  
•  `--bootstrap-server`: Адрес одного из брокеров для начального подключения.  
•  `--generate`: Генерирует план перераспределения.  
•  `--topics-to-move-json-file`: Путь к файлу JSON, содержащему список топиков и партиций для перераспределения. В нашем случае, это /tmp/reassignment.json, так как мы смонтировали локальный файл в контейнер.  
•  `--broker-list`: Список брокеров, которые будут использоваться для перераспределения.  
•  `--out-file`: Путь к файлу, в который будет записан сгенерированный план перераспределения. В данном случае перезаписываем файл `/tmp/reassignment.json`.  

7. Перераспределение партиций  

Теперь выполните перераспределение партиций, используя сгенерированный план. Выполните следующую команду в контейнере `kafka-cli`:  

```
docker-compose exec kafka-cli kafka-reassign-partitions --bootstrap-server kafka-1:29092 --execute --reassignment-json-file /tmp/reassignment.json
```
Эта команда:  

•  `--execute`: Запускает процесс перераспределения.  
•  `--reassignment-json-file`: Путь к файлу JSON, содержащему план перераспределения (сгенерированный на предыдущем шаге).  

8. Проверка статуса перераспределения  

Чтобы проверить статус перераспределения, выполните следующую команду в контейнере kafka-cli:  

```
docker-compose exec kafka-cli kafka-reassign-partitions --bootstrap-server kafka-1:29092 --verify --reassignment-json-file /tmp/reassignment.json
```

Вывод должен показать, что перераспределение выполняется. После завершения вы должны увидеть сообщение об успешной проверке.  

9. Подтверждение успешного выполнения перераспределения  

После завершения перераспределения повторите команду --verify. Если все прошло успешно, вы увидите сообщение:  

```
Status of partition reassignment:
Reassignment of partition [balanced_topic,0] completed successfully.
Reassignment of partition [balanced_topic,1] completed successfully.
Reassignment of partition [balanced_topic,2] completed successfully.
Reassignment of partition [balanced_topic,3] completed successfully.
Reassignment of partition [balanced_topic,4] completed successfully.
Reassignment of partition [balanced_topic,5] completed successfully.
Reassignment of partition [balanced_topic,6] completed successfully.
Reassignment of partition [balanced_topic,7] completed successfully.
```

Также, вы можете повторно выполнить команду `--describe` (см. шаг 4), чтобы убедиться, что распределение партиций соответствует содержимому вашего `reassignment.json` файла.  

10. Моделирование сбоя брокера  

Остановите брокер `kafka-1`:  

```
docker-compose stop kafka-1
```

11. Проверка состояния топиков после сбоя  

Снова выполните команду `--describe` в контейнере `kafka-cli`:  

```
docker-compose exec kafka-cli kafka-topics --bootstrap-server kafka-2:29092 --describe --topic balanced_topic
```

Важно: Обратите внимание, что мы теперь подключаемся к `kafka-2`, так как `kafka-1` недоступен.  

Вы должны увидеть, что лидером некоторых партиций, которые ранее находились на `kafka-1`, стал один из оставшихся брокеров (`kafka-2` или `kafka-3`). Также, обратите внимание на столбец `Isr`. Он может содержать меньше брокеров, чем `Replicas`, пока `kafka-1` не вернется в строй и не синхронизирует свои реплики.  

12. Запуск брокера заново  

Запустите брокер `kafka-1` снова:  

```
docker-compose start kafka-1
```

13. Проверка, восстановилась ли синхронизация реплик  

Подождите некоторое время, чтобы `kafka-1` успел восстановиться и синхронизировать свои реплики. Затем снова выполните команду `--describe` (подключившись к любому живому брокеру):  

```
docker-compose exec kafka-cli kafka-topics --bootstrap-server kafka-2:29092 --describe --topic balanced_topic
```

Убедитесь, что:  

•  `kafka-1` снова присутствует в списке `Isr` для тех партиций, где он является репликой.  
•  Все лидеры и реплики соответствуют вашему `reassignment.json` файлу (если вы не вносили изменений после перераспределения).  

Если `Isr` содержит все ожидаемые реплики, это означает, что синхронизация восстановлена.  

Дополнительные советы и диагностика:  

•  **Логи Kafka**: Смотрите логи брокеров Kafka (с помощью `docker logs kafka-1`, `docker logs kafka-2`, `docker logs kafka-3`) для получения дополнительной информации об ошибках и процессах перераспределения.  
•  **JMX**: Используйте JMX (порты `9991, 9992, 9993`) для мониторинга метрик Kafka, таких как количество активных контроллеров, состояние репликации и использование ресурсов брокерами. Существуют различные инструменты для просмотра JMX метрик, например, VisualVM.  
•  **Увеличение скорости перераспределения**: Вы можете увеличить скорость перераспределения, настроив параметры `throttle` в Kafka.   
•  **Проблемы с Zookeeper**: Если возникают проблемы с подключением к Zookeeper, убедитесь, что Zookeeper запущен и доступен по указанному адресу. Проверьте логи Zookeeper (docker logs zookeeper) для выявления проблем.  
•  **Размер сообщений**: Большой размер сообщений может замедлить процесс репликации и перераспределения. Можно рассмотреть возможность оптимизации размера сообщений или увеличения параметров, связанных с размером сообщений, в конфигурации Kafka.  

Заключение:  

Этот пример демонстрирует основные шаги балансировки партиций и диагностики кластера Kafka. В реальной среде мониторинг и диагностика должны быть автоматизированы с использованием специализированных инструментов.