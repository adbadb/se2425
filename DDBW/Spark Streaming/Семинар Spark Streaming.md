Семинар
Apache Spark Streaming

### 1. Вводная часть 

#### Демонстрация 1: Запуск простого стриминг-приложения

- **Описание:** Подсчёт количества слов в реальном времени из текстового потока TCP.
- **Действия:**
  1. Запустить сервер, генерирующий текстовый поток:
     ```bash
     nc -lk 9999
     ```
  2. Запустить следующий скрипт на Python (PySpark):

     ```python
     from pyspark import SparkContext
     from pyspark.streaming import StreamingContext

     sc = SparkContext("local[2]", "NetworkWordCount")
     ssc = StreamingContext(sc, 1)

     lines = ssc.socketTextStream("localhost", 9999)
     words = lines.flatMap(lambda line: line.split(" "))
     wordCounts = words.map(lambda word: (word, 1)).reduceByKey(lambda a, b: a + b)
     wordCounts.pprint()
     ssc.start()
     ssc.awaitTermination()
     ```

  3. Ввести в терминале с `nc` любую строку (например, `hello world hello`), посмотреть результат в Spark.

#### Демонстрация 2: Оконные операции

- **Описание:** Подсчёт количества слов за последние 10 секунд с обновлением каждые 5 секунд.

  ```python
  windowedCounts = words.map(lambda word: (word, 1)).reduceByKeyAndWindow(
      lambda a, b: a + b, 
      windowDuration=10,  # 10 секунд
      slideDuration=5     # обновление каждые 5 секунд
  )
  windowedCounts.pprint()
  ```

### 2. Самостоятельная работа студентов

#### Задание 1 (Простое): Подсчёт уникальных слов в потоке

- **Задача:** Напишите приложение, которое выводит количество уникальных слов, поступивших в каждом батче.
- **Подсказка:** используйте методы `distinct()` и `count()`.

**Решение:**

```python
uniqueWordCount = words.transform(lambda rdd: rdd.distinct().count())
uniqueWordCount.pprint()
```

**Правильный ответ:**  
В каждом батче в консоли выводится число уникальных слов (например, если введено "hello world hello", результат — 2).


#### Задание 2 (Средней сложности): Подсчёт уникальных IP-адресов в логах

- **Задача:** Пусть поток состоит из логов вида `192.168.0.1 - - [date] "GET /index.html" ...`. Необходимо подсчитать количество уникальных IP-адресов за последние 30 секунд с обновлением каждые 10 секунд.

**Решение:**

```python
import re

def extract_ip(line):
    match = re.match(r"^([\d\.]+)", line)
    return match.group(1) if match else None

ips = lines.map(extract_ip).filter(lambda x: x is not None)
windowed_ips = ips.window(windowDuration=30, slideDuration=10)
unique_ips_count = windowed_ips.transform(lambda rdd: rdd.distinct().count())
unique_ips_count.pprint()
```

**Правильный ответ:**  
В каждом окне (30 секунд) показывается число уникальных IP-адресов.


#### Задание 3 (Сложное): Реализация подсчёта частоты слов с сохранением состояния (stateful processing)

- **Задача:** Реализуйте подсчёт общего количества каждого слова, встретившегося во всём стриме с момента запуска (используйте `updateStateByKey`).

**Решение:**

```python
def updateFunction(new_values, running_count):
    return sum(new_values) + (running_count or 0)

ssc.checkpoint("checkpoint")  # указать директорию для checkpoint

runningCounts = words.map(lambda word: (word, 1)).updateStateByKey(updateFunction)
runningCounts.pprint()
```

**Правильный ответ:**  
В консоли отображается общее количество каждого слова, агрегированное по всему времени работы приложения.

#### Задание 4 (Дополнительно, выше среднего): Фильтрация "вредных" слов и подсчёт частоты "чистых" слов

- **Задача:** Допустим, есть список запрещённых ("вредных") слов. Необходимо фильтровать их из потока и подсчитывать частоту оставшихся слов за последние 60 секунд (с обновлением каждые 20 секунд).

**Решение:**

```python
bad_words = {"spam", "abuse", "scam"}

filtered_words = words.filter(lambda word: word not in bad_words)
windowedCounts = filtered_words.map(lambda word: (word, 1)).reduceByKeyAndWindow(
    lambda a, b: a + b,
    windowDuration=60,
    slideDuration=20
)
windowedCounts.pprint()
```

**Правильный ответ:**  
В окне отображаются только "чистые" слова и их частота.


### 4. Домашнее задание (по желанию)

- Реализовать стриминг-приложение, агрегирующее события из нескольких источников (например, TCP и файловая система), с записью результатов в файл.
