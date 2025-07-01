Конспект лекции
Apache Spark для распределенной пакетной обработки данных

## 1. Введение в распределенную обработку данных

### Проблема MapReduce 

MapReduce — парадигма программирования для обработки больших данных, представленная Google в 2004 году. Apache Hadoop содержит открытую реализация MapReduce.
Основные ограничения Hadoop MapReduce:
- Низкая производительность из-за записи/чтения промежуточных результатов на диск между задачами Map и Reduce.
- Сложность программирования: необходимость вручную  определять множество функций, типы их аргументов и результатов,  выстраивать цепочки заданий.
- Отсутствие поддержки интерактивного анализа данных и итерационных алгоритмов (важных для машинного обучения, алгоритмов на графах).

### Появление Spark
  - В 2009 году в Университете Калифорнии (Беркли) стартовал проект Spark как исследовательская работа по созданию быстрой и удобной платформы для обработки данных.
  - Spark хранит промежуточные результаты задач в памяти (JVM исполнителя не выключается после выполнения задачи).
  - Способ программирования - более простой и естественный - обработка коллекций в функциональном стиле, как в Java Streams.

**Пример: WordCount в Hadoop MapReduce (60+ строк кода)** vs **Spark (10 строк)**:
```python
// Hadoop MR
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
  public void map(Object key, Text value, Context context) { /* ... */ }
}
public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
  public void reduce(Text key, Iterable<IntWritable> values, Context context) { /* ... */ }
}

// Spark RDD
sc.textFile("file.txt")
  .flatMap(line => line.split(" "))
  .map(word => (word, 1))
  .reduceByKey(_ + _)
  .saveAsTextFile("output")
```

### Архитектура Spark

1. Cluster Manager (менеджер ресурсов):
   - Standalone (встроенный в Spark).
   - Apache Mesos (кластерный менеджер).
   - Hadoop YARN (если уже есть Hadoop-кластер).
   - Kubernetes (современный, через контейнеры Docker).

2. Driver Program (основная программа):
   - Ваш код на Java/python/Python (например, `SparkPi.python`).
   - Создаёт `SparkSession` (и `SparkContext`).
   - Планирует выполнение.

3. Executors (рабочие JVM-процессы на узлах кластера):
   - Выполняют задачи (Task, порция работы).
   - Хранят разделы/партишены (Partition, часть коллекции записей, фрагмент RDD/DataFrame).
   - Общаются с Driver по сети через Netty.

4. Основные объекты:
   - `SparkContext` (Spark 1.x) — создание RDD.
   - `SparkSession` (Spark 2.x+) — унифицированный контекст для DataFrame/SQL/Streaming, содержит в себе `SparkContext`.

Пример создания контекста:
```python
spark = SparkSession.builder()
  .appName("My Example")
  .master("local[4]") // 4 ядра локально
  .config("spark.executor.memory", "2g")
  .getOrCreate()
sc = spark.sparkContext // если нужен старый RDD API
```

В какой момент начинается обработка данных? Когда дальше откладывать уже невозможно: нужно результат обработки  записать в файл, показать в интерактивной оболочке, присвоить переменной в вызывающей программе, или хотя бы вернуть количество записей в результате. Такие операции, которые должны быть выполнены немедленно, называются действиями (англ. action), например, `saveAsText()`, `count()`, `collect()`, `take()`.
К этому времени предыдущие вызовы (трансформации) уже построили направленный ациклический граф задания (узлы - RDD):
   - `RDD A → map() → RDD B → filter() → RDD C → reduceByKey() → RDD D`
Планировщик (DAG Scheduler) выявляет в графе  этапы (англ. stage) - группы соседних RDD, которые можно выполнять вместе. Например, несколько проверок разных условий разумно объединить. Некоторые операции требуют перегруппировки данных (англ. shuffle), по ним будет проходить граница стадий. 
   - Узкие преобразования (Narrow Transformations) - которые могу выполняться независимо (можно последовательно, можно параллельно) для разных партишенов - например,`map` или `filter` - объединяются в один этап.
   - Широкие преобразования (Wide Transformations) - для каждого партишена результата может понадобиться собрать данные из нескольких испходных партишенов - например, `groupBy` и  `join`, - требуют перегруппировки данных (Shuffle), а значит  планирования еще одного этапа.
4. Планировщик задач (Task Scheduler) делит этап на задачи (по числу партишнов):
   - Если RDD разбит на 8 партишнов, будет 8 задач в Stage.
5. Задачи отправляются на свободные исполнители (Executors) и там выполняются (каждая задача читает данные → обрабатывает → отдает результат).
6. Результат возвращается в Driver (или пишется в HDFS/S3).

**Пример: План выполнения `rdd.map().filter().groupByKey()`**
```
Stage 1 (Narrow):
  Task 1 (map/filter на partition 1) → Executor 1
  Task 2 (map/filter на partition 2) → Executor 2

Stage 2 (Wide, Shuffle):
  [SHUFFLE] 
  Task 3 (groupByKey partition A) ← данные от Task 1,2
  Task 4 (groupByKey partition B) ← данные от Task 1,2
```

## RDD (Resilient Distributed Dataset)

- Неизменяемая (Immutable) коллекция объектов.
- Распределённая (разбита на партишены, лежит на разных узлах).
- Восстанавливаемая (Resilient) — если узел упал, пропавшие партишены RDD автоматически пересчитываются по истории преобразований.

### Создание RDD

1. Из коллекции в памяти:
   ```python
   numbers = sc.parallelize([1, 2, 3, 4, 5], numSlices = 4)
   ```
2. Из внешнего источника:
   ```python
   text = sc.textFile("hdfs://namenode/path/to/log.txt", minPartitions = 10)
   ```

### Операции над RDD

1. Преобразования (ленивые, не запускают вычисления):
   - `map()`, `flatMap()`, `filter()`, `distinct()`, `groupByKey()`, `join()`.
   ```python
   words = text.flatMap(lambda line: line.split(" ")) // RDD[String]
   pairs = words.map(lambda word: (word, 1))          // RDD[(String, Int)]
   ```

2. Действия (запускают вычисления и возвращают результат):
   - `collect()`, `count()`, `first()`, `take(n)`, `saveAsTextFile()`, `foreach()`.
   ```python
   wordCounts = pairs.reduceByKey(add) // RDD[(String, Int)]
   wordCounts.collect().foreach(println)     // действие запускает весь процесс
   ```

### Кэширование RDD (закрепление в памяти)

- По умолчанию RDD пересчитывается заново при каждом `action()`.
- `rdd.cache()` → закрепить в RAM при ближайшем следующем действии (пока хватает памяти).
- `rdd.persist(StorageLevel.MEMORY_AND_DISK)` → если не влезло в RAM, сбросит на диск.

## DataFrame и Spark SQL

### Что такое DataFrame

- Распределённая таблица с типизированными столбцами (как Pandas DataFrame).
- RDD[Row] + Schema (метаданные о столбцах).
- Catalyst Optimizer → автоматически оптимизирует SQL-запросы.

**Пример создания DataFrame:**
```python
df = spark.read
  .format("csv")
  .option("header", "true")
  .option("inferSchema", "true")
  .load("users.csv")

df.printSchema() // выводит типы столбцов
// root
// |-- id: integer (nullable = true)
// |-- name: string (nullable = true)
// |-- age: integer (nullable = true)
```

### Основные операции с DataFrame

1. Фильтрация:
   ```python
   df.filter($"age" > 25).show()
   // SELECT * FROM df WHERE age > 25
   ```

2. Группировка:
   ```python
   df.groupBy("country")
     .agg(count("*").as("cnt"), avg("age").as("avg_age"))
     .orderBy("cnt".desc)
     .show()
   // SELECT country, COUNT(*), AVG(age)
// FROM df GROUP BY country ORDER BY cnt DESC
   ```

3. JOIN:
   ```python
   orders = spark.read.csv("orders.csv")
   result = df.join(orders, "user_id", "inner")
   ```

### Spark SQL (запросы как в БД)

Можно писать **чистый SQL** прямо в коде:
  ```pyhton
  df.createOrReplaceTempView("people")
  spark.sql("""
    SELECT name, age 
    FROM people 
    WHERE age = 18
  """)
  ```

### Отличия DataFrame от Dataset в python

| **Критерий**        | **DataFrame**                     | **Dataset**                          |
|---------------------|-----------------------------------|--------------------------------------|
| **Тип данных**      | `RDD[Row]` (нет типизации)        | `RDD[ТвойКласс]` (JVM-типизация)    |
| **Схема таблицы**   | Определяется динамически (`inferSchema`) | Задаётся через `case class`         |
| **Производительность** | Чуть быстрее (меньше сериализации) | Почти как DataFrame (Catalyst Opt.) |
| **Пример кода**     | `df.filter($"age" > 25)`          | `ds.filter(_.age > 25)`             |
| **Использование**   | Для SQL/аналитики/агрегаций       | Для типизированной бизнес-логики   |

**Пример Dataset:**
```python
case class Person(name: String, age: Int)
val ds = spark.read.json("people.json").as[Person]
ds.filter(_.age >= 18).map(_.name).show()
// Типизированная фильтрация!
```

### Catalyst Optimizer (как Spark оптимизирует запросы)

1. Logical Plan (абстрактный план запроса):
   - `df.filter("age" > 25).groupBy("city").count()`
   - Превращается в дерево операций: `Filter → Aggregate → Project`

2. Optimizer (правила оптимизации):
   - Predicate Pushdown: `WHERE age > 25` выполниьб как можно раньше, сразу после чтения данных.
   - Projection Pushdown: читать только нужные столбцы (`age, city`).
   - Join Reordering: в `A JOIN B`, поменять таблицы местами, если `B` меньше.

3. Physical Plan (конкретные операции):
   - `Scan parquet → Filter(age > 25) → HashAggregate(city) → Exchange (shuffle)`

4. Code Generation (генерация байткода):
   - Вместо интерпретации запроса на лету — компилирует в Java bytecode.
   - В 10-20 раз быстрее Hadoop MapReduce за счёт JIT-компиляции.

**Посмотреть план выполнения запроса:**
```python
df.filter("age" > 25).groupBy("city").count().explain(true)
// == Parsed Logical Plan ==
// == Analyzed Logical Plan ==
// == Optimized Logical Plan ==
// == Physical Plan ==
// *(2) HashAggregate(keys=[city#1], functions=[count(1)])
// +- Exchange hashpartitioning(city#1, 200)
//    +- *(1) HashAggregate(keys=[city#1], functions=[partial_count(1)])
//       +- *(1) Filter (isnotnull(age#0) && (age#0 > 25))
```
Что тут важно?
- `Exchange` = shuffle (данные пересылаются между узлами).
- `HashAggregate` = группировка в памяти.
- `200 партишнов` (по умолчанию) после `groupBy`.

## Чтение и запись данных (Sources API)

### Чтение из файлов (Parquet, CSV, JSON)

Parquet (бинарный колоночный формат, лучший выбор):
```python
df = spark.read.parquet("hdfs://path/to/users.parquet")
```
Почему Parquet?
- Кодирует и сжимает данные (до 75% экономии места).
- Читает только нужные столбцы (predicate pushdown).
- Поддерживает схемы данных (эволюция таблиц).

CSV/JSON:
```python
df_csv = spark.read
  .option("header", "true")
  .option("inferSchema", "true") // автоопределение типов
  .csv("data.csv")

df_json = spark.read.json("data.json")
// JSON может быть многострочным:
spark.read.option("multiLine", true).json("complex.json")
```

### Запись данных (режимы сохранения)

1. Overwrite (перезаписать поверх):
   ```python
   df.write.mode("overwrite").parquet("output.parquet")
   ```
2. Append (дописать в конец):
   ```python
   df.write.mode("append").saveAsTable("mytable") // Hive/HDFS
   ```
3. Ignore (не писать, если папка не пуста):
   ```python
   df.write.mode("ignore").json("path")
   ```
4. ErrorIfExists (бросает исключение по умолчанию):
   ```python
   df.write.parquet("path") // кидает ошибку, если папка занята
   ```

### JDBC (Чтение/Запись в PostgreSQL/MySQL)**

**Чтение таблицы из БД:**
```python
dbDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql://host:5432/mydb")
  .option("dbtable", "public.users")
  .option("user", "myuser")
  .option("password", "mypass")
  .load()
```
**Фильтрация на стороне БД (predicate pushdown):**
```python
dbDF.filter("age" > 30).explain(true)
// Spark отправит SQL: `SELECT * FROM users WHERE age > 30`
```

**Запись в БД (batch insert):**
```python
df.write
  .format("jdbc")
  .option("url", "jdbc:mysql://host:3306/mydb")
  .option("dbtable", "reports")
  .option("batchsize", "10000") // коммит каждые 10к строк
  .save()
```

### HDFS (Hadoop Distributed File System)

- Блок = 128 MB (по умолчанию).
- `sc.textFile("hdfs://namenode:8020/data/*.log")`.
- Малые файлы — проблема (NameNode хранит метаданные в RAM).
  - Решение: `spark.sql.files.maxPartitionBytes` (сливать мелкие файлы).

**Пример чтения логов из HDFS:**
```python
logs = sc.textFile("hdfs://namenode:8020/logs/access.log")
errors = logs.filter(line => line.contains("404") || line.contains("500"))
errors.saveAsTextFile("hdfs://namenode:8020/logs/errors")
```

### S3 (Amazon Simple Storage Service)

- Spark прозрачно работает с S3 как с HDFS:
  ```python
  data = spark.read.parquet("s3a://mybucket/users/")
  ```
- s3a:// (новейший коннектор, поддерживает до 5Тб файлы).
- Настройки производительности:
  ```python
  spark.conf.set("fs.s3a.fast.upload", true)
  spark.conf.set("fs.s3a.connection.maximum", 100)
  ```

### Kafka (Streaming из очереди сообщений)

**Чтение потока из Kafka топика:**
```python
val kafkaStream = spark.readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", "kafka1:9092,kafka2:9092")
  .option("subscribe", "user_events")
  .load()

kafkaStream.selectExpr("CAST(value AS STRING)")
  .writeStream
  .format("console")
  .outputMode("append")
  .start()
  .awaitTermination()
```
Как работает?
1. Spark подключается к Kafka-брокерам.
2. Читает оффсеты (позиции) из топика.
3. Поток разбивается на микробатчи (по 100-500 мс).
4. Обрабатывает RDD/DataFrame → выводит в sink (консоль/HDFS/ES).

### Hive Metastore и совместимость с Spark

Что такое Hive?

- Hadoop SQL-движок (2008 год).
- Хранит метаданные таблиц (схемы, локации, партиции).
- Транслирует SQL → MapReduce (теперь Tez/Spark).

**Пример Hive-таблицы:**
```sql
CREATE TABLE logs (
  ip STRING,
  url STRING,
  status INT
)
PARTITIONED BY (year INT, month INT)
STORED AS PARQUET
LOCATION 'hdfs://path/to/logs/';
```

### Как Spark работает с Hive Metastore?

1. spark-shell запускаем с Hive-опциями:
   ```bash
   spark-shell --conf spark.sql.catalogImplementation=hive
   ```
2. Читаем Hive-таблицы как Spark DataFrame:
   ```python
   spark.sql("SELECT * FROM myhive_db.logs WHERE year = 2023").show()
   ```
3. Записываем в Hive-таблицу:
   ```python
   df.write.mode("overwrite").saveAsTable("mydb.newtable")
   ```

## Оптимизация и тюнинг Spark

### Как Spark выполняет Job (повторение)

1. **Driver** строит **DAG** (из `map`, `filter`, `join`…).
2. **DAG Scheduler** делит на **Stages** (где нужен shuffle — новый Stage).
3. **Task Scheduler** запускает Tasks (по числу партишнов).

### Ключевые настройки производительности

| **Параметр**                          | **Описание**                                                                 | **Пример значения**   |
|---------------------------------------|------------------------------------------------------------------------------|------------------------|
| `spark.executor.memory`               | Память одного исполнителя (JVM heap)                                        | `4g`, `8g`             |
| `spark.executor.cores`                | Ядра CPU на executor (больше = больше задач параллельно)                   | `2`, `4`               |
| `spark.sql.shuffle.partitions`        | Число партишнов **после** `groupBy`, `join` (по умолчанию 200)              | `500` (для больших JOIN) |
| `spark.default.parallelism`           | Партишны **до** shuffle (по умолчанию = cores \* 2-3)                       | `100`                  |
| `spark.memory.fraction`               | Доля памяти под **кэш** (остальное под вычисления)                          | `0.6` (60% RAM на кэш) |
| `spark.serializer`                    | `org.apache.spark.serializer.KryoSerializer` (быстрее JavaSerializer)      | `kryo`                 |

**Пример `spark-submit`:**
```bash
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 10 \
  --executor-memory 8G \
  --executor-cores 4 \
  --conf spark.sql.shuffle.partitions=500 \
  --conf spark.yarn.maxAppAttempts=2 \
  --class com.myapp.Main \
  target/python-2.12/myapp.jar \
  hdfs://input/data.csv hdfs://output/result
```

### Memory Management (управление памятью)

1. **Execution Memory** (под вычисления: `groupBy`, `sort`).
2. **Storage Memory** (под `cache()`, `persist()`).
3. **User Memory** (служебная память JVM).

**Если ошибки:**
- **`GC overhead limit exceeded`** → увеличь `spark.executor.memoryOverhead`.
- **`Container killed by YARN`** → уменьш `spark.executor.memory`.

**Мониторинг:**
- **Spark UI** (порт 4040) → Jobs, Stages, Tasks, Memory.
- **Ganglia / Prometheus** (метрики кластера).

**Пример полной конфигурации `spark-submit`:**
```bash
spark-submit \
  --master yarn \
  --deploy-mode cluster \
  --num-executors 10 \
  --executor-memory 8G \
  --executor-cores 4 \
  --driver-memory 4G \
  --conf spark.sql.shuffle.partitions=500 \
  --conf spark.dynamicAllocation.enabled=true \
  --conf spark.yarn.maxAppAttempts=2 \
  --class com.myapp.Main \
  target/python-2.12/myapp.jar \
  hdfs://input/data.csv hdfs://output/result
```

**12.4. Лучшие практики написания Spark-кода**

1. **Избегай `collect()` на больших данных** (тащит все в Driver → OOM).
   - Вместо этого: `write.parquet()` или `count()`.
2. **Кэшируй промежуточные результаты** (если повторно используются):
   ```python
   val df = spark.read.parquet("data").cache() // закрепить в RAM
   df.filter("age" > 25).show()
   df.groupBy("city").count().show() // повторно использует кэш
   ```
3. **Используй `explain()`** (проверяй план выполнения):
   ```python
   df.filter("age" > 25).groupBy("city").count().explain(true)
   ```
4. **Меньше партишнов → дольше обработка; слишком много → оверхед**.
   - Оптимум: `2-4 партишна на каждое ядро CPU`.

