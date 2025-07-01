Семинар 
Работа с Apache Spark (RDD, DataFrame, Spark SQL) на PySpark

**Цель занятия:**
1. Освоить создание и операции с **RDD** (Resilient Distributed Dataset).
2. Научиться работать с **DataFrame** и выполнять запросы через **Spark SQL**.
3. Применить базовые трансформации и действия (transformations/actions) в Spark.

**Подготовка:**
- Убедитесь, что PySpark установлен (`pip install pyspark`).
- Используйте **Spark 3.5.x** и **Python 3.7+**.
- Основные среды разработки: **Jupyter Notebook**, **VS Code** или **PySpark Shell**.

### **Часть 1: Демонстрационные примеры (преподаватель)**

#### **Пример 1: Основы RDD (Word Count)**

```python
from pyspark.sql import SparkSession

# Запуск SparkSession
spark = SparkSession.builder.appName("RDD Demo").master("local[*]").getOrCreate()
sc = spark.sparkContext

# Предполагается, что есть файл data.txt с текстом
# Например:
# hello spark world
# hello python world

# 1. Создаём текстовый RDD
textRDD = sc.textFile("data.txt") # Файл должен существовать!

# 2. Трансформации
# В Python используем lambda-функции
words = textRDD.flatMap(lambda line: line.split(" ")) # разбить строки на слова
pairs = words.map(lambda word: (word.lower(), 1))  # (слово, 1)
wordCounts = pairs.reduceByKey(lambda a, b: a + b) # сложить частоты

# 3. Action: сохраняем результат
# Сначала удалим папку, если она существует, чтобы избежать ошибки
# import shutil
# shutil.rmtree("output_wordcount", ignore_errors=True)
wordCounts.saveAsTextFile("output_wordcount")

# 4. Посмотрим топ-10 частых слов
# lambda t: (t[1], t[0]) меняет местами (count, word)
top_10_words = wordCounts.map(lambda t: (t[1], t[0])) \
                         .sortByKey(ascending=False) \
                         .take(10) # берём первые 10

for count, word in top_10_words:
    print(f"{word}: {count}")

# spark.stop() # опционально для скриптов
```
**Что показать:**
- Ленивые вычисления (ничего не происходит до `saveAsTextFile` или `take`).
- `.toDebugString()` для отображения плана выполнения (`print(wordCounts.toDebugString().decode())`).

#### **Пример 2: Переход к DataFrame**

```python
from pyspark.sql import SparkSession
# Продолжение из предыдущего примера...

# Берём тот же wordCounts RDD
# В PySpark нужно явно указать имена колонок
wordCountsDF = wordCounts.toDF(["word", "count"])
wordCountsDF.printSchema()
# root
# |-- word: string (nullable = true)
# |-- count: long (nullable = true)

# SQL-запросы поверх DataFrame
wordCountsDF.createOrReplaceTempView("words")
spark.sql("""
  SELECT word, count
  FROM words
  WHERE count > 10
  ORDER BY count DESC
  LIMIT 5
""").show()

# Или DSL-синтаксис (DataFrame API)
# Вместо $`count` используем col("count") или просто строковое выражение
from pyspark.sql.functions import col, desc

wordCountsDF.filter(col("count") > 10) \
            .orderBy(desc("count")) \
            .limit(5) \
            .show()
```
**Что обсудить:**
- Как автоматически определяется схема (`inferSchema`).
- Разница `spark.sql` vs `col()` выражения.
- Оптимизатор **Catalyst** (`.explain()` → Physical Plan). `wordCountsDF.explain()`

#### **Пример 3: Чтение CSV и агрегирующие запросы**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import avg, desc

spark = SparkSession.builder.appName("CSV Demo").master("local[*]").getOrCreate()

# tips.csv - стандартный датасет
df = spark.read \
  .option("header", "true") \
  .option("inferSchema", "true") \
  .csv("tips.csv") # Убедитесь, что файл существует

# 1. SQL
df.createOrReplaceTempView("tips")
spark.sql("""
  SELECT sex, AVG(tip) as avg_tip
  FROM tips
  GROUP BY sex
  ORDER BY avg_tip DESC
""").show()

# 2. DataFrame API (аналогично)
df.groupBy("sex") \
  .agg(avg("tip").alias("avg_tip")) \
  .orderBy(desc("avg_tip")) \
  .show()
```
**Акценты:**
- `inferSchema` vs ручное задание схемы.
- `agg()` с разными функциями (`sum`, `max`, `countDistinct`).

---

### **Часть 2: Задачи для самостоятельной работы**

#### **Задача 1: Анализ логов веб-сервера (RDD)**

Дан файл `access.log`. **Требуется:**
1. Найти топ-5 **IP-адресов** по числу запросов.
2. Посчитать количество **ошибок (4xx, 5xx статусы)**.
3. Вывести все запросы к `/api` и сгруппировать по HTTP-методам (GET, POST...).

**Шаблон кода (PySpark):**
```python
# sc - SparkContext из предыдущих примеров
logs = sc.textFile("access.log")

# 1. Топ-5 IP
# .split(" ")[0] - взять первый элемент (IP)
ips = logs.map(lambda line: (line.split(" ")[0], 1)) \
         .reduceByKey(lambda a, b: a + b) \
         .map(lambda t: (t[1], t[0])) \
         .sortByKey(ascending=False) \
         .take(5)

print("Топ-5 IP:")
for count, ip in ips:
    print(f"{ip}: {count}")

# 2. Ошибки 4xx/5xx
def get_status_code(line):
    try:
        # Более надежный парсинг
        return int(line.split('"')[2].strip().split(" ")[0])
    except (IndexError, ValueError):
        return 0

errors = logs.filter(lambda line: 400 <= get_status_code(line) < 600)
print(f"\nErrors count: {errors.count()}")

# 3. /api запросы
def get_method(line):
    try:
        return line.split('"')[1].split(" ")[0]
    except IndexError:
        return "UNKNOWN"

apiRequests = logs.filter(lambda line: "/api" in line) \
                  .map(lambda line: (get_method(line), 1)) \
                  .reduceByKey(lambda a, b: a + b)

print("\nAPI requests by method:")
for method, count in apiRequests.collect():
    print(f"{method}: {count}")
```

#### **Задача 2: Анализ продаж (DataFrame + Spark SQL)**

Дан `sales.csv`. **Требуется:**
1. Найти **общую выручку** (`amount * price`) по каждому продукту.
2. Вывести топ-3 пользователей (`user_id`) по сумме покупок.
3. Определить средний чек для каждого пользователя.

**Шаблон кода (DataFrame API):**
```python
from pyspark.sql.functions import sum, col, desc, countDistinct

sales = spark.read \
  .option("header", "true") \
  .option("inferSchema", "true") \
  .csv("sales.csv")

# 1. Выручка по продуктам
sales.groupBy("product") \
     .agg(sum(col("amount") * col("price")).alias("revenue")) \
     .orderBy(desc("revenue")) \
     .show()

# 2. Топ-3 пользователей по сумме покупок
sales.groupBy("user_id") \
     .agg(sum(col("amount") * col("price")).alias("total_spent")) \
     .orderBy(desc("total_spent")) \
     .limit(3) \
     .show()

# 3. Средний чек пользователя
userStats = sales.groupBy("user_id") \
  .agg(
    sum(col("amount") * col("price")).alias("total_spent"),
    countDistinct("order_id").alias("orders_count")
  )

userStats.withColumn("avg_order", col("total_spent") / col("orders_count")) \
         .orderBy(desc("avg_order")) \
         .show()
```

#### **Задача 3: Join-операции**

Даны `users.csv` и `orders.csv`. **Задание:**
1. Выведите все заказы с именами пользователей (JOIN).
2. Найдите города, где суммарный оборот (`total_sum`) больше 10000.
3. Определите клиентов, не сделавших ни одного заказа (LEFT ANTI JOIN).

**Пример решения (Spark SQL):**
```python
users = spark.read.option("header", "true").csv("users.csv")
orders = spark.read.option("header", "true").option("inferSchema", "true").csv("orders.csv")

users.createOrReplaceTempView("users")
orders.createOrReplaceTempView("orders")

# 1. Заказы с именами
spark.sql("""
  SELECT u.name, o.order_id, o.total_sum
  FROM users u JOIN orders o ON u.user_id = o.user_id
""").show()

# 2. Города с оборотом > 10к
spark.sql("""
  SELECT u.city, SUM(o.total_sum) as total_revenue
  FROM users u JOIN orders o ON u.user_id = o.user_id
  GROUP BY u.city
  HAVING total_revenue > 10000
""").show()

# 3. Клиенты без заказов (LEFT ANTI JOIN)
# В SQL это `LEFT ANTI JOIN` или `LEFT JOIN ... WHERE o.user_id IS NULL`
spark.sql("""
  SELECT u.name
  FROM users u LEFT ANTI JOIN orders o ON u.user_id = o.user_id
""").show()

# То же самое через DataFrame API
# 1. Join
users.join(orders, on="user_id", how="inner").select("name", "order_id", "total_sum").show()

# 2. Города с оборотом > 10к
users.join(orders, on="user_id", how="inner") \
    .groupBy("city") \
    .agg(sum("total_sum").alias("total_revenue")) \
    .filter(col("total_revenue") > 10000) \
    .show()

# 3. Клиенты без заказов
users.join(orders, on="user_id", how="left_anti").show()
```