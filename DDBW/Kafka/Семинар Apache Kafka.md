Семинар 
Apache Kafka

## Демонстрация 1: Запуск окружения и знакомство с Kafka UI

#### 1. Запуск окружения
```bash
# Запускаем все сервисы: Kafka, ZooKeeper, Schema Registry, Kafka UI, ksqlDB
# Используем docker-compose up (без -d), чтобы видеть логи в реальном времени
docker-compose up
```
#### 2. Проверка запущенных контейнеров
```bash
# Проверяем, что все контейнеры работают
docker-compose ps
```

#### 3. Знакомство с Kafka UI
Откройте в браузере `http://localhost:8080`.

## Демонстрация 2: Создание топиков и работа с ними через CLI

#### 1. Создание топика
```bash
# Создаем топик с именем "demo-topic", 3 партициями и фактором репликации 1
# Используем docker exec для выполнения команды внутри контейнера 'kafka'
docker exec kafka kafka-topics --create --topic demo-topic --bootstrap-server kafka:29092 --partitions 3 --replication-factor 1
```

#### 2. Просмотр списка топиков
```bash
# Выводим список всех топиков
docker exec kafka kafka-topics --list --bootstrap-server kafka:29092
```
> **Примечание для преподавателя:** Обновите страницу Kafka UI, чтобы показать, что новый топик появился и там.

#### 3. Просмотр детальной информации о топике
```bash
# Выводим детальную информацию о топике: партиции, лидер, реплики
docker exec kafka kafka-topics --describe --topic demo-topic --bootstrap-server kafka:29092
```

#### 4. Удаление топика
```bash
# Удаляем топик
docker exec kafka kafka-topics --delete --topic demo-topic --bootstrap-server kafka:29092
```

## Демонстрация 3: Работа с продюсерами и консьюмерами через командную строку

#### 1. Пересоздание топика для демонстрации
```bash
docker exec kafka kafka-topics --create --topic demo-topic --bootstrap-server kafka:29092 --partitions 3 --replication-factor 1
```

#### 2. Запуск консольного продюсера
Откройте новый терминал.
```bash
# Запускаем консольный продюсер в интерактивном режиме
docker exec -it kafka kafka-console-producer --topic demo-topic --bootstrap-server kafka:29092
```

#### 3. Запуск консольного консьюмера
Откройте еще один новый терминал.
```bash
# Запускаем консольный консьюмер для чтения всех сообщений с начала
docker exec kafka kafka-console-consumer --topic demo-topic --bootstrap-server kafka:29092 --from-beginning
```

#### 4. Работа с группами потребителей
```bash
# Откройте два новых терминала
# В первом запустите консьюмер в группе 'my-group'
docker exec kafka kafka-console-consumer --topic demo-topic --bootstrap-server kafka:29092 --group my-group

# Во втором запустите еще один консьюмер в той же группе
docker exec kafka kafka-console-consumer --topic demo-topic --bootstrap-server kafka:29092 --group my-group
```

## Демонстрация 4: Разработка простого приложения с использованием Kafka

### Предварительные требования
- Java (JDK 11 или выше)
- Maven
- IDE (IntelliJ IDEA, Eclipse и т.д.)

#### 1. Создание Maven-проекта и добавление зависимостей в `pom.xml`
```xml
<dependencies>
    <!-- Kafka клиент -->
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>3.3.1</version> <!-- Версия, соответствующая кластеру -->
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-simple</artifactId>
        <version>1.7.36</version>
    </dependency>
</dependencies>
```

#### 2. Создание класса продюсера
```java
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;
import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class SimpleProducer {
    public static void main(String[] args) throws InterruptedException {
        Properties props = new Properties();
        // Подключаемся к Kafka, запущенной в Docker, по порту, открытому на localhost
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        
        try (Producer<String, String> producer = new KafkaProducer<>(props)) {
            for (int i = 0; i < 10; i++) {
                String key = "key-" + i;
                String value = "value-" + i;
                ProducerRecord<String, String> record = new ProducerRecord<>("demo-topic", key, value);
                RecordMetadata metadata = producer.send(record).get();
                
                System.out.printf("Sent record(key=%s value=%s) meta(partition=%d, offset=%d)\n",
                        key, value, metadata.partition(), metadata.offset());
                
                Thread.sleep(500);
            }
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

#### 3. Создание класса консьюмера
```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.time.Duration;
import java.util.Collections;
import java.util.Properties;

public class SimpleConsumer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "demo-app-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        
        try (Consumer<String, String> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(Collections.singletonList("demo-topic"));
            
            // Обработка корректного завершения
            final Thread mainThread = Thread.currentThread();
            Runtime.getRuntime().addShutdownHook(new Thread(() -> {
                System.out.println("Starting exit...");
                consumer.wakeup();
                try {
                    mainThread.join();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }));

            while (true) {
                ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("Received record(key=%s, value=%s) at offset=%d\n",
                            record.key(), record.value(), record.offset());
                }
            }
        } catch (Exception e) {
            // WakeupException будет перехвачен здесь при завершении
            System.out.println("Consumer loop finished.");
        }
    }
}
```

#### 4. Компиляция и запуск приложения из IDE

## Демонстрация 5: Простой пример Kafka Streams

#### 1. Добавление зависимости Kafka Streams в `pom.xml`
```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-streams</artifactId>
    <version>3.3.1</version>
</dependency>
```

#### 2. Создание класса для обработки потоков
```java
// Код класса WordCountApplication (идентичен коду из лекции)
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Produced;
import java.util.Arrays;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;

public class WordCountApplication {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "wordcount-application");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> textLines = builder.stream("streams-input-topic");
        KTable<String, Long> wordCounts = textLines
            .flatMapValues(value -> Arrays.asList(value.toLowerCase().split("\\W+")))
            .groupBy((key, word) -> word)
            .count();
        wordCounts.toStream().to("streams-output-topic", Produced.with(Serdes.String(), Serdes.Long()));

        final KafkaStreams streams = new KafkaStreams(builder.build(), props);
        final CountDownLatch latch = new CountDownLatch(1);
        Runtime.getRuntime().addShutdownHook(new Thread("streams-shutdown-hook") {
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });

        try {
            streams.start();
            latch.await();
        } catch (Throwable e) {
            System.exit(1);
        }
        System.exit(0);
    }
}
```

#### 3. Создание входного и выходного топиков
```bash
docker exec kafka kafka-topics --create --topic streams-input-topic --bootstrap-server kafka:29092
docker exec kafka kafka-topics --create --topic streams-output-topic --bootstrap-server kafka:29092
```

#### 4. Запуск приложения Streams из IDE
Запустите класс `WordCountApplication`.

#### 5. Отправка данных во входной топик
В новом терминале:
```bash
docker exec -it kafka kafka-console-producer --topic streams-input-topic --bootstrap-server kafka:29092
```
Введите несколько предложений:
`Hello Kafka Streams`
`Kafka is great`
`Hello world`

#### 6. Просмотр результатов в выходном топике
В еще одном терминале:
```bash
docker exec kafka kafka-console-consumer --topic streams-output-topic --bootstrap-server kafka:29092 \
--from-beginning \
--property print.key=true \
--property print.value=true \
--property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
--property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```
## Задания для самостоятельной работы

### Задание 1: Настройка Kafka и создание топиков

**Цель задания:** Освоить базовые операции по установке, настройке Kafka и созданию топиков с различными параметрами.

**Задачи:**
1. Развернуть Apache Kafka, используя предоставленный `docker-compose.yml` из инструкции по развертыванию.
2. Настроить сервер Kafka с нестандартными параметрами (путем изменения `docker-compose.yml` или `server.properties` внутри контейнера):
   - Изменить порт для внешнего доступа с 9092 на 9093.
   - Установить количество партиций по умолчанию равным 2.
   - Настроить период хранения сообщений (retention period) в 2 дня.
3. Создать три топика с разными конфигурациями с помощью консольных утилит:
   - `users-topic` с 3 партициями и фактором репликации 1.
   - `orders-topic` с 5 партициями и фактором репликации 1.
   - `logs-topic` с 2 партициями и фактором репликации 1.
4. Написать скрипт, который выводит информацию о созданных топиках в удобочитаемом формате.

**Ожидаемый результат:**
- Работающий кластер Kafka с нестандартными настройками.
- Три созданных топика с указанными параметрами.
- Скрипт для вывода информации о топиках.

**Критерии оценки:**
- Корректность настройки кластера Kafka.
- Правильность создания топиков с заданными параметрами.
- Качество и информативность скрипта для вывода информации.

### Задание 2: Разработка простого продюсера и консьюмера

**Цель задания:** Научиться разрабатывать базовые приложения для отправки и получения сообщений в Kafka.

**Задачи:**
1. Разработать консольное приложение-продюсер на Java или Python, которое:
   - Принимает на вход имя топика и текстовое сообщение.
   - Отправляет сообщение в указанный топик.
   - Выводит информацию об успешной отправке (партиция, смещение).
2. Разработать консольное приложение-консьюмер, которое:
   - Принимает на вход имя топика и имя группы потребителей.
   - Читает и выводит все сообщения из указанного топика.
   - Корректно обрабатывает сигналы завершения (например, Ctrl+C), используя `ShutdownHook` или `signal handler`.
3. Протестировать работу приложений на топиках, созданных в задании 1.

**Ожидаемый результат:**
- Работающие приложения продюсера и консьюмера.
- Демонстрация обмена сообщениями между приложениями через Kafka.

**Критерии оценки:**
- Корректность работы продюсера и консьюмера.
- Обработка ошибок и исключительных ситуаций.
- Реализация корректного завершения работы консьюмера.
- Качество кода и документации.

### Задание 3: Мониторинг и анализ работы Kafka

**Цель задания:** Освоить инструменты мониторинга и анализа работы Kafka.

**Задачи:**
1. Настроить JMX-мониторинг для Kafka, используя JMX Exporter для Prometheus.
2. Установить и настроить Prometheus и Grafana для сбора и визуализации метрик.
3. Создать дашборд в Grafana, отображающий ключевые метрики Kafka:
   - Количество сообщений в секунду (throughput).
   - Задержка (latency) для Produce-запросов.
   - Использование диска (размер логов по топикам).
   - Количество активных контроллеров.
4. Провести нагрузочное тестирование с помощью `kafka-producer-perf-test` и `kafka-consumer-perf-test`.
5. Проанализировать результаты и составить отчет.

**Ожидаемый результат:**
- Настроенная система мониторинга Kafka.
- Дашборд в Grafana с ключевыми метриками.
- Отчет о результатах нагрузочного тестирования.

**Критерии оценки:**
- Корректность настройки мониторинга.
- Информативность и удобство дашборда.
- Качество анализа результатов тестирования.

### Задание 4: Разработка системы логирования с использованием Kafka

**Цель задания:** Разработать систему централизованного логирования с использованием Kafka в качестве брокера сообщений.

**Задачи:**
1. Разработать приложение-генератор логов, которое:
   - Генерирует логи разных уровней (INFO, WARNING, ERROR).
   - Отправляет логи в Kafka-топик `application-logs`.
   - Поддерживает различные форматы логов (JSON, Plain Text).
2. Разработать приложение-обработчик логов, которое:
   - Читает логи из Kafka-топика.
   - Фильтрует логи по уровню.
   - Сохраняет логи в файлы, разделенные по уровню и дате.
3. Настроить партиционирование топика `application-logs` по уровню логирования, реализовав собственный `Partitioner`.
4. Реализовать механизм ротации файлов логов по размеру.

**Ожидаемый результат:**
- Работающая система логирования с использованием Kafka.
- Демонстрация работы системы с генерацией и обработкой логов разных уровней.

**Критерии оценки:**
- Корректность работы системы логирования.
- Эффективность партиционирования и обработки логов.
- Удобство использования и расширяемость системы.

### Задание 5: Реализация системы обработки событий с гарантированной доставкой

**Цель задания:** Разработать систему обработки событий с гарантированной доставкой сообщений (exactly-once semantics).

**Задачи:**
1. Разработать приложение-генератор событий, которое:
   - Генерирует события с уникальными идентификаторами.
   - Отправляет события в Kafka-топик `events-topic` с использованием транзакций Kafka.
2. Разработать приложение-обработчик событий, которое:
   - Читает события из Kafka-топика с уровнем изоляции `read_committed`.
   - Обрабатывает события и сохраняет результаты в базу данных (например, PostgreSQL).
   - Использует паттерн "идемпотентный потребитель" для предотвращения повторной обработки (проверка по ID события в БД).
3. Реализовать механизм восстановления после сбоев для обоих приложений.
4. Провести тестирование системы в условиях сбоев (например, принудительное завершение процессов).

**Ожидаемый результат:**
- Работающая система обработки событий с гарантированной доставкой.
- Демонстрация корректной работы системы в условиях сбоев.

**Критерии оценки:**
- Корректность реализации exactly-once семантики (транзакционный продюсер, идемпотентный консьюмер).
- Продемонстрировать отсутствие дубликатов в БД при перезапуске консьюмера в середине обработки пачки сообщений.
- Устойчивость системы к сбоям.
- Эффективность механизма восстановления.

### Задание 6: Разработка системы потоковой обработки данных с Kafka Streams

**Цель задания:** Освоить Kafka Streams API для разработки приложений потоковой обработки данных.

**Задачи:**
1. Разработать приложение для анализа активности пользователей, которое:
   - Читает события пользователей из топика `user-events`.
   - Агрегирует события по пользователям и типам действий.
   - Вычисляет статистику активности в реальном времени (например, количество действий каждого типа за последний час).
   - Записывает результаты в топик `user-stats`.
2. Реализовать оконные операции для анализа трендов активности.
3. Реализовать соединение (join) потока событий с таблицей пользователей (KTable) для обогащения данных.
4. Разработать простой веб-интерфейс для визуализации статистики в реальном времени (например, с использованием WebSocket).

**Ожидаемый результат:**
- Работающее приложение потоковой обработки на базе Kafka Streams.
- Веб-интерфейс для визуализации статистики активности пользователей.

**Критерии оценки:**
- Корректность реализации потоковой обработки (агрегации, оконные операции, join).
- Эффективность использования Kafka Streams API.
- Информативность и удобство веб-интерфейса.
