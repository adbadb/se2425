Семинар: 
Apache Spark Structured Streaming

**Цель семинара:** Научиться создавать, запускать и отлаживать базовые потоковые приложения с использованием PySpark. Закрепить на практике ключевые концепции: источники, приёмники, режимы вывода, работа с состоянием, окна.

#### **Краткое повторение теоретических концепций:**

Прежде чем приступить к коду, давайте быстро вспомним ключевые идеи из лекции:

1.  **Поток как бесконечная таблица:** Мы пишем запросы к потоку так же, как к обычной статической таблице (DataFrame).
2.  **Архитектура:** Источник (Source) -> Преобразование (Transformation) -> Приёмник (Sink).
3.  **Режимы вывода (Output Modes):**
    - `append`: Только новые строки (для stateless-преобразований).
    - `complete`: Вся таблица результатов (для агрегаций без окон).
    - `update`: Только обновлённые строки (для агрегаций с окнами).
4.  **Триггеры (Triggers):** Определяют, как часто запускается микро-батч.
5.  **Контрольные точки (Checkpoints):** **Обязательны** для откагацией по времени (оконные функции) и обработкой опоздавших данных (водяные знаки).
4.  **Понять** критическую роль контрольных точек (checkpoints).
5.  **Получить** навыки отладки потоковых приложений с помощью вывода в консоль.

**Необходимые инструменты и подготовка:**
- Python 3.7+
- Установленная библиотека PySpark: `pip install pyspark`
- Утилита `netcat` (`nc`) для генерации потока данных.
    - В Linux/macOS: обычно установлена по умолчанию.
    - В Windows: можно использовать WSL (Windows Subsystem for Linux) или найти аналог.
- Создайте рабочую директорию для проекта.

### **Часть 1: Разминка – "Hello, Stream!" (Классический Word Count)**

**Задача:** Написать приложение, которое в реальном времени читает строки текста из сетевого сокета (порта), считает частоту встречаемости каждого слова и выводит обновляемую таблицу в консоль.

**Шаг 1: Подготовка окружения**

1.  Откройте **первый терминал**. Он будет имитировать источник данных. Запустите `netcat`, чтобы он слушал порт 9999:
    ```bash
    nc -lk 9999
    ```
    Теперь этот терминал будет ждать входящих соединений и отправлять всё, что вы в него введёте, подключившемуся клиенту (нашему Spark-приложению).

2.  Создайте Python-файл, например, `wordcount_stream.py`.

**Шаг 2: Пишем код на PySpark**

```python
# wordcount_stream.py

from pyspark.sql import SparkSession
from pyspark.sql.functions import explode, split

# 1. Инициализация SparkSession
# local[2] означает, что Spark будет работать локально, используя 2 ядра.
# Одно ядро для приёма данных, второе для их обработки.
spark = SparkSession \
    .builder \
    .appName("StructuredNetworkWordCount") \
    .master("local[2]") \
    .getOrCreate()

# Убираем лишние логи, чтобы видеть только результат
spark.sparkContext.setLogLevel("ERROR")

# 2. Создание потокового DataFrame
# Мы читаем данные из сокета. Spark будет автоматически подключаться к localhost:9999
lines = spark \
    .readStream \
    .format("socket") \
    .option("host", "localhost") \
    .option("port", 9999) \
    .load()

# DataFrame `lines` имеет одну колонку "value" типа string.

# 3. Трансформации
# Разбиваем строки (lines.value) на отдельные слова.
# split() возвращает массив, explode() превращает каждый элемент массива в отдельную строку.
words = lines.select(
   explode(
       split(lines.value, " ")
   ).alias("word")
)

# 4. Агрегация с состоянием (Stateful Aggregation)
# Группируем по словам и считаем их количество.
# Это stateful-операция, т.к. Spark должен помнить счётчики для каждого слова между батчами.
wordCounts = words.groupBy("word").count()

# 5. Настройка и запуск потокового запроса (Query)
# Мы хотим выводить результат в консоль.
query = wordCounts \
    .writeStream \
    .outputMode("complete") # Выводить всю таблицу результатов каждый раз
    .format("console") \
    .trigger(processingTime='5 seconds') \
    .start() # Запускаем поток!

print("=== Потоковый запрос запущен. Вводите текст в терминале с netcat. ===")

# Ожидаем завершения потока (в данном случае, вечно, пока не прервём вручную)
query.awaitTermination()
```

**Шаг 3: Запуск и проверка**

1.  Откройте **второй терминал** в вашей рабочей директории.
2.  Запустите Python-скрипт:
    ```bash
    python wordcount_stream.py
    ```
3.  Вернитесь в **первый терминал** (`netcat`) и начните вводить слова, разделяя их пробелами. Нажимайте Enter после каждой строки.
    ```
    hello spark world
    hello big data
    spark is cool
    ```
4.  Наблюдайте за выводом во **втором терминале**. Каждые 5 секунд Spark будет выводить обновленную таблицу:

    ```
    -------------------------------------------
    Batch: 1
    -------------------------------------------
    +-------+-----+
    |   word|count|
    +-------+-----+
    |  hello|    2|
    |  spark|    2|
    |  world|    1|
    |   cool|    1|
    |    big|    1|
    |   data|    1|
    |     is|    1|
    +-------+-----+
    ```

- **`local[2]`**: Почему минимум 2 ядра? (Приёмник + обработка).
- **`outputMode('complete')`**: Почему именно этот режим? (Потому что у нас агрегация, и мы хотим видеть всю таблицу, а не только новые/обновленные строки). Что было бы, если бы мы использовали `append`? (Spark выдал бы ошибку, т.к. `append` не поддерживается для агрегаций без водяных знаков).
- **Состояние (State)**: Где Spark хранит промежуточные подсчеты слов? (В памяти драйвера/экзекьюторов). Что произойдет, если мы остановим скрипт и запустим заново? (Состояние потеряется, подсчет начнется с нуля). Это подводит нас к следующей теме.

### **Часть 2: Работа с файлами, окнами и водяными знаками**

**Задача:** Создать приложение, которое обрабатывает поток JSON-файлов с данными от IoT-сенсоров. Нам нужно посчитать среднюю температуру за каждую 10-секундную временную "корзину" (окно), корректно обрабатывая данные, которые могут прийти с опозданием.

**Сценарий:** Система пишет данные о температуре в JSON-файлы и складывает их в директорию. Каждый файл содержит одно или несколько событий.

**Шаг 1: Подготовка**

1.  Создайте в рабочей директории три поддиректории:
    - `input_data` — сюда мы будем "бросать" наши JSON-файлы.
    - `checkpoint_dir` — сюда Spark будет сохранять свое состояние.
    - `output_dir` (опционально) — для записи результата в файлы.

2.  Создайте новый Python-файл, например `sensor_stream.py`.

**Шаг 2: Пишем код**

```python
# sensor_stream.py

from pyspark.sql import SparkSession
from pyspark.sql.functions import window, avg
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, TimestampType

# 1. Инициализация SparkSession
spark = SparkSession \
    .builder \
    .appName("StructuredSensorStream") \
    .master("local[*]") \
    .getOrCreate()

spark.sparkContext.setLogLevel("ERROR")

# 2. Определение схемы данных
# В потоковых приложениях ВСЕГДА лучше явно определять схему.
# Это надежно и позволяет избежать лишних затрат на ее вывод.
schema = StructType([
    StructField("deviceId", StringType(), True),
    StructField("temperature", DoubleType(), True),
    # Важнейшее поле - время, когда событие произошло!
    StructField("event_time", TimestampType(), True)
])

# 3. Создание потокового DataFrame из файлового источника
# Spark будет "следить" за директорией input_data и обрабатывать новые json-файлы.
streaming_df = spark \
    .readStream \
    .schema(schema) \
    .option("maxFilesPerTrigger", 1) # Обрабатывать не более 1 файла за раз
    .json("input_data")

# 4. Агрегация с окнами и водяными знаками
# withWatermark говорит Spark: "Мы готовы ждать опоздавшие данные не более 20 секунд".
# Все, что придет позже этого порога, будет проигнорировано.
# Это позволяет Spark очищать старое состояние и не хранить его вечно.
windowed_avg_temp = streaming_df \
    .withWatermark("event_time", "20 seconds") \
    .groupBy(
        # Группируем по 10-секундным непересекающимся окнам (tumbling window)
        window("event_time", "10 seconds")
    ) \
    .agg(avg("temperature").alias("avg_temp"))

# 5. Запуск запроса
query = windowed_avg_temp \
    .writeStream \
    .outputMode("update") # Выводить только обновленные окна
    .format("console") \
    .option("truncate", "false") # Не обрезать длинные строки в выводе
    .trigger(processingTime='5 seconds') \
    .option("checkpointLocation", "checkpoint_dir") # !!! КРИТИЧЕСКИ ВАЖНО !!!
    .start()

print("=== Потоковый запрос запущен. Создавайте JSON-файлы в директории 'input_data'. ===")

query.awaitTermination()
```

**Шаг 3: Запуск и симуляция потока данных**

1.  Запустите скрипт `sensor_stream.py`.
2.  В отдельном терминале начните создавать файлы в директории `input_data`. Выполняйте команды с небольшой задержкой.

    ```bash
    # Первые данные
    echo '{"deviceId": "d1", "temperature": 25.5, "event_time": "2023-11-10T12:00:05Z"}' > input_data/data1.json
    sleep 2
    echo '{"deviceId": "d2", "temperature": 26.5, "event_time": "2023-11-10T12:00:08Z"
	```
	
### **Часть 3: Самостоятельная работа**

**Задача:** Проанализировать поток событий от сервиса такси.

**Входные данные:** JSON-файлы в директории `taxi_events` со следующей схемой:
- `event_id` (string)
- `event_type` (string): `start_ride` или `end_ride`
- `fare` (double): стоимость поездки (только для `end_ride`)
- `event_time` (timestamp)

**Требуется:**
1.  Настроить потоковое чтение из директории `taxi_events`.
2.  Используя 1-минутные окна, посчитать для каждого окна:
    - Общее количество начавшихся поездок (`start_ride`).
    - Общее количество завершенных поездок (`end_ride`).
    - Суммарную выручку по завершенным поездкам.
3.  Использовать водяной знак в 2 минуты для обработки опоздавших событий.
4.  Выводить результат в консоль в режиме `update`.
5.  **Обязательно** использовать checkpointing.

```python
# taxi_stream_task.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import window, col, sum, when, lit
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, TimestampType

spark = SparkSession.builder.appName("TaxiStreamTask").master("local[*]").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

schema = StructType([
    StructField("event_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("fare", DoubleType(), True),
    StructField("event_time", TimestampType(), True)
])

streaming_df = spark.readStream.schema(schema).json("taxi_events")

# >>> ВАШ КОД ДОЛЖЕН БЫТЬ ЗДЕСЬ <<<
# 1. Добавьте водяной знак
# 2. Сгруппируйте по 1-минутному окну
# 3. Используйте .agg() для вычисления трех метрик.
#    Подсказка: используйте `sum(when(condition, 1).otherwise(0))` для условного подсчета.


# >>> ЗАПУСК ЗАПРОСА <<<
# query = ...
# query.awaitTermination()
```

```python
# taxi_stream_solution.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import window, col, sum, when, lit
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, TimestampType

spark = SparkSession.builder.appName("TaxiStreamTask").master("local[*]").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

schema = StructType([
    StructField("event_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("fare", DoubleType(), True),
    StructField("event_time", TimestampType(), True)
])

streaming_df = spark.readStream.schema(schema).json("taxi_events")

# Решение
analysis_df = streaming_df \
    .withWatermark("event_time", "2 minutes") \
    .groupBy(window("event_time", "1 minute")) \
    .agg(
        sum(when(col("event_type") == "start_ride", 1).otherwise(0)).alias("started_rides"),
        sum(when(col("event_type") == "end_ride", 1).otherwise(0)).alias("ended_rides"),
        sum(when(col("event_type") == "end_ride", col("fare")).otherwise(0)).alias("total_fare")
    )

query = analysis_df.writeStream \
    .outputMode("update") \
    .format("console") \
    .option("truncate", "false") \
    .option("checkpointLocation", "taxi_checkpoint") \
    .trigger(processingTime='10 seconds') \
    .start()

print("=== Taxi analysis stream started. Create JSON files in 'taxi_events' directory. ===")
query.awaitTermination()
```

Важно:** файлы нужно сначала создать с временным именем, а потом переименовать, чтобы Spark не начал читать не до конца записанный файл.
    ```bash
    # Терминал 2
    echo '{"deviceId": "d-101", "temperature": 85.5, "timestamp": "2023-10-27T12:00:00Z"}' > spark_seminar/data_input/event1.json
    sleep 10
    echo '{"deviceId": "d-102", "temperature": 95.1, "timestamp": "2023-10-27T12:00:10Z"}' > spark_seminar/data_input/event2.json
    sleep 10
    echo '{"deviceId": "d-103", "temperature": 99.9, "timestamp": "2023-10-27T12:00:20Z"}' > spark_seminar/data_input/event3.json
    ```
3.  Наблюдайте, как в консоли PySpark-приложения появляются только события с температурой > 90.0.
4.  Загляните в папку `spark_seminar/checkpoints/file_stream`. Вы увидите там служебные файлы, которые хранят состояние потока.

#### **Задание 3: Агрегация по времени с окнами и водяными знаками**

**Цель:** Понять, как использовать `window` для агрегации по времени и `withWatermark` для обработки опоздавших данных и очистки состояния.

**Сценарий:** Анализируем поток кликов пользователей на сайте. Нужно посчитать, сколько кликов каждого типа (`click`, `view`, `purchase`) происходит в 10-секундных интервалах. Мы готовы ждать опоздавшие данные не более 5 секунд.

**Шаги:**

1.  Создайте файл `window_stream_app.py`.
2.  Используйте тот же файловый источник, что и в Задании 2.

```python
# window_stream_app.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import window
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

# 1. Инициализация и схема
spark = SparkSession.builder.appName("WindowedStream").master("local[*]").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")

schema = StructType([
    StructField("event_time", TimestampType(), True),
    StructField("action", StringType(), True)
])

# 2. Создание потока
events_df = spark.readStream.schema(schema).json("spark_seminar/data_input")

# 3. Применение водяного знака и окон
# Мы готовы ждать опоздавшие данные 5 секунд.
# Агрегируем по 10-секундным непересекающимся (tumbling) окнам.
windowed_counts = events_df \
    .withWatermark("event_time", "5 seconds") \
    .groupBy(
        window("event_time", "10 seconds"),
        "action"
    ).count()

# 4. Запуск запроса
# Используем режим "update", т.к. результаты для окон могут обновляться
query = windowed_counts \
    .writeStream \
    .outputMode("update") \
    .format("console") \
    .option("truncate", "false") \
    .trigger(processingTime='5 seconds') \
    .option("checkpointLocation", "spark_seminar/checkpoints/window_stream") \
    .start()

print("Считаем события в 10-секундных окнах...")
query.awaitTermination()
```

**Как запустить и проверить:**

1.  Очистите директорию `spark_seminar/data_input`.
2.  Запустите скрипт: `python window_stream_app.py`.
3.  В другом терминале будем симулировать события, включая опоздавшие:
    ```bash
    # Первое окно (12:00:00 - 12:00:10)
    echo '{"event_time": "2023-10-27T12:00:02Z", "action": "click"}' > spark_seminar/data_input/e1.json
    sleep 2
    echo '{"event_time": "2023-10-27T12:00:04Z", "action": "view"}' > spark_seminar/data_input/e2.json
    # Наблюдайте за результатом в консоли. Появится первое окно.

    sleep 10
    # Второе окно (12:00:10 - 12:00:20) и ОПОЗДАВШЕЕ событие для первого окна
    echo '{"event_time": "2023-10-27T12:00:11Z", "action": "click"}' > spark_seminar/data_input/e3.json
    sleep 2
    # Это событие опоздало, но попадает в рамки водяного знака (5 сек)
    echo '{"event_time": "2023-10-27T12:00:08Z", "action": "click"}' > spark_seminar/data_input/e4.json
    # Наблюдайте, как обновится счетчик для первого окна!

    sleep 10
    # Это событие опоздало СЛИШКОМ сильно и будет проигнорировано водяным знаком
    echo '{"event_time": "2023-10-27T12:00:01Z", "action": "purchase"}' > spark_seminar/data_input/e5.json
    ```

