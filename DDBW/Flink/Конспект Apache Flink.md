Конспект лекции
Apache Flink

## Введение

Мы изучаем, как данные хранятся и управляются в распределенных системах (распределенные БД, NoSQL, Data Lakes, Data Warehouses). Данные не только *хранятся* (Data at Rest), но и постоянно *генерируются и перемещаются* (Data in Motion). Необходимо обрабатывать данные *по мере их поступления* (в реальном или почти реальном времени).

Традиционные подходы (обработка запросов к БД, пакетная обработка) не подходят для задач, требующих низкой задержки и обработки бесконечных потоков данных.
Примеры: обнаружение мошенничества, мониторинг систем, анализ логов в реальном времени, персонализация рекомендаций, ETL в реальном времени.
- Бесконечный поток данных.
- Данные приходят не по порядку (out-of-order).
- Необходимость поддерживать *состояние* (state) между событиями (например, подсчет суммы за последний час).
- Обеспечение надежности и отказоустойчивости (чтобы не потерять данные или состояние).
- Обработка с разными гарантиями доставки (At-most-once, At-least-once, Exactly-once).

## Введение в Apache Flink

Что такое Flink? Распределенный движок для *потоковой* (стриминговой) обработки данных с открытым исходным кодом.
Ключевое отличие: Flink – это *настоящий* стриминговый движок. Он обрабатывает события *одно за другим* или небольшими группами (не микро-батчами, как некоторые другие системы).
Способен обрабатывать как *неограниченные* (unbounded), так и *ограниченные* (bounded) потоки данных. Фактически, батчевая обработка в Flink – это частный случай стриминга (обработка ограниченного потока).
Разработан в TU Berlin, вырос в компанию Data Artisans (теперь Ververica).
Основные преимущества:
- Низкая задержка и высокая пропускная способность.
- Управление состоянием (State Management).
- Отказоустойчивость и гарантии обработки.
- Гибкое управление временем (Event Time, Processing Time).
- Поддержка сложных операций (окна, соединения потоков).

## Фундаментальные Концепции Flink 

1.  **Потоки Данных (Data Streams):**
Представляют собой последовательность *событий* (events).
События: единичные записи данных, поступающие из источника (например, клик пользователя, показание датчика, запись в логе).
Неограниченные потоки: данные поступают непрерывно, поток никогда не заканчивается. Основной фокус Flink.
Ограниченные потоки: поток данных имеет конечное начало и конец (например, чтение файла). Flink обрабатывает их как специальные случаи неограниченных потоков.
2.  **Время в Потоковой Обработке:**
Одна из ключевых сложностей. События могут приходить не по порядку из-за сетевых задержек, буферизации и т.д.
**Processing Time:** Время, когда событие *обрабатывается* оператором Flink. Самое простое, но наименее точное для анализа событий, т.к. зависит от скорости обработки.
**Ingestion Time:** Время, когда событие *попадает* в Flink из источника. Чуть лучше, но все еще зависит от задержки между источником и Flink.
**Event Time:** Время, когда событие *произошло* в источнике (указано в самом событии). Наиболее точное для анализа, но требует обработки out-of-order событий. Flink excels here.
3.  **Watermarks:**
Механизм в Flink для работы с Event Time и обработки out-of-order событий.
Watermark с меткой `T` означает, что Flink ожидает, что больше не увидит событий с Event Time *меньше или равным* `T` из данного источника/партиции.
Используются для определения "границы" в Event Time и сигнализирования операторам (особенно окнам), что можно завершать обработку за прошедший период времени.
Как генерируются: Источники данных или специальные операторы вставляют Watermarks в поток. Могут быть эвристическими (основаны на наблюдении за временем событий) или детерминированными (если источник гарантирует порядок).
Поздние события (Late Events): События, пришедшие после Watermark, который "закрыл" их временной интервал. Flink предоставляет механизмы для их обработки (например, помещение в "side output").

**Демонстрация:**
Простая Flink job, читающая из сокета или файла.
Визуализация Job Graph в Flink Web UI.

## Управление Состоянием

1.  **Что такое Состояние в Flink?**
Состояние – это данные, которые оператор Flink сохраняет между обработкой отдельных событий или групп событий.
Примеры: счетчик событий для ключа, сумма значений, буфер для оконной агрегации, информация о последнем событии от конкретного пользователя.
Состояние критически важно для большинства осмысленных потоковых приложений (агрегации, соединения, обнаружение паттернов).
2.  **Типы Состояния:**
**Keyed State:** Состояние, связанное с конкретным *ключом* в потоке. Доступно только операторам, которые работают с *ключированными потоками* (`KeyedStream`, после операции `keyBy()`). Это наиболее часто используемый тип состояния.
Примеры Keyed State: `ValueState<T>`, `ListState<T>`, `ReducingState<T>`, `AggregatingState<I, O>`, `MapState<UK, UV>`. Flink управляет сериализацией, памятью и отказоустойчивостью для этого состояния.
**Operator State:** Состояние, связанное с конкретной *экземпляром оператора*. Каждый параллельный экземпляр оператора имеет свое состояние. Используется реже, например, для буферизации элементов перед отправкой в sink или для координации между параллельными экземплярами.
Примеры Operator State: `ListState<T>` (часто используется для распределения состояния между экземплярами при изменении параллелизма).
3.  **Управляемое Состояние (Managed State) vs. Сырое Состояние (Raw State):**
**Managed State:** Состояние, которым управляет Flink (Keyed State и большинство Operator State). Flink заботится о его хранении, сериализации, отказоустойчивости и масштабировании. Разработчик использует предоставляемые Flink интерфейсы (`ValueState`, `ListState` и т.д.). **Рекомендуемый подход.**
**Raw State:** Состояние, которым разработчик управляет сам (например, хранит объекты в памяти оператора). Flink не знает о внутренней структуре этого состояния и просто сохраняет его байты при чекпоинтах. Требует ручного управления сериализацией и не рекомендуется, кроме очень специфичных случаев.
4.  **State Backends:**
Определяют, *как* Flink физически хранит управляемое состояние и метаданные чекпоинтов во время выполнения.
**MemoryStateBackend:** Состояние хранится в памяти TaskManager'а. Чекпоинты сохраняются в памяти JobManager'а или на файловой системе. Быстрый, но ограничен объемом памяти, не подходит для большого состояния или продакшена.
**FsStateBackend (или HDFSStateBackend):** Состояние хранится в памяти TaskManager'а, но чекпоинты записываются на файловую систему (HDFS, S3, GCS и т.д.). Лучше для продакшена, чем Memory, но восстановление может быть медленнее, т.к. нужно загружать состояние из файлов.
**RocksDBStateBackend:** Состояние хранится в RocksDB – высокопроизводительной встроенной key-value базе данных – на диске TaskManager'а. Чекпоинты сохраняются на файловой системе. Лучший выбор для *большого* состояния, т.к. не ограничено памятью TaskManager'а. Обеспечивает низкую задержку при доступе к состоянию. **Настоятельно рекомендуется для большинства продакшен-сценариев с большим состоянием.**

## Отказоустойчивость и Гарантии Обработки 

1.  **Проблема Отказов:**
В распределенной системе компоненты (TaskManagers, сеть) могут выходить из строя.
Как гарантировать, что обработка данных продолжится корректно после сбоя, без потери или дублирования данных?
2.  **Механизм Checkpointing (Контрольные точки):**
Основной механизм отказоустойчивости в Flink.
Создает *глобальные, согласованные снимки* (consistent snapshots) *состояния всех операторов* приложения в определенный момент времени.
Использует разновидность алгоритма Chandy-Lamport для создания распределенного снимка без остановки потока данных ("barrier alignment").
**Процесс чекпоинта:** JobManager инициирует чекпоинт. У источника вставляются специальные "барьеры" (checkpoint barriers). Барьеры текут по графу операторов вместе с данными. Каждый оператор, получив барьер, сохраняет свое состояние (для этого чекпоинта) в State Backend и передает барьер дальше. Как только JobManager получает барьеры от всех источников и всех операторов, чекпоинт считается завершенным и атомарно коммитится.
3.  **Восстановление после Сбоя:**
Если TaskManager или JobManager выходит из строя, Flink перезапускает сбойные части приложения.
При перезапуске, операторы загружают свое состояние из *последнего успешно завершенного чекпоинта*.
Обработка данных возобновляется *с момента* этого чекпоинта. Источники данных (если они поддерживают "replay" с определенной позиции, как Kafka) сбрасываются на соответствующую позицию.
4.  **Savepoints (Точки сохранения):**
Мануально инициируемые снимки состояния.
Похожи на чекпоинты, но предназначены для *плановых* операций (обновление версии Flink job, изменение логики, перенос кластера, A/B тестирование).
Не удаляются автоматически (в отличие от старых чекпоинтов).
Позволяют запустить новую версию job из состояния старой версии.
5.  **Гарантии Обработки (Processing Guarantees):**
**At-most-once:** Каждое событие обрабатывается не более одного раза. В случае сбоя данные в процессе обработки могут быть потеряны. Самая низкая гарантия.
**At-least-once:** Каждое событие обрабатывается *хотя бы* один раз. В случае сбоя события могут быть обработаны повторно, что может привести к дублированию результатов. Требует идемпотентных операций или дедупликации downstream.
**Exactly-once:** Каждое событие обрабатывается *ровно* один раз. Это самая сильная гарантия и является одной из ключевых фич Flink.
Как Flink достигает Exactly-once? Комбинация:
Надежные чекпоинты (гарантируют согласованное состояние).
Источники данных, поддерживающие "replay" (например, Kafka, Kinesis).
Sinks (приемники данных), поддерживающие *транзакционную запись* или *атомарный коммит* результатов, связанных с чекпоинтом (например, двухфазный коммит). Flink интегрируется с такими sinks, чтобы результаты чекпоинта были видимы только *после* успешного завершения чекпоинта.
**Важно:** Гарантия Exactly-once в Flink часто означает Exactly-once *обновление состояния внутри Flink* и Exactly-once *доставку в транзакционный sink*. Если sink не транзакционный, полной end-to-end Exactly-once гарантии достичь сложнее без дополнительных мер.

**Демонстрация:**
Настройка чекпоинтов для простой job.
Имитация сбоя TaskManager'а (если возможно в тестовой среде).
Показ восстановления job из чекпоинта.
Создание и использование Savepoint'а для перезапуска job.

## API и Основные Операции **

1.  **DataStream API:**
Низкоуровневый API, основной для потоковой обработки.
Позволяет строить графы операторов над потоком данных (`DataStream`).
Предоставляет множество трансформаций:
`map`, `filter`, `flatMap`: Элементарные преобразования событий.
`keyBy(KeySelector)`: Логически партиционирует поток по ключу. **Критично для использования Keyed State и параллельной обработки по ключам.**
`reduce`, `aggregate`: Агрегация по ключу или в окне.
`connect`, `union`: Объединение двух или более потоков.
`split`, `select` (устарело), `sideOutput`: Разделение потока.
`window(WindowAssigner)` + `trigger`, `evictor`: Оконные операции.
Разработчик пишет код, который реализует логику операторов (например, `MapFunction`, `FilterFunction`, `ProcessFunction`).
2.  **Windowing (Оконные функции):**
Необходимы для выполнения агрегаций над *ограниченными* группами событий в неограниченном потоке (например, подсчет кликов за последнюю минуту).
Окно определяется *WindowAssigner'ом*.
Типы окон:
**Tumbling Windows (Перекатывающиеся):** Фиксированный размер, не перекрываются. `[0, 5), [5, 10), [10, 15) ...`
**Sliding Windows (Скользящие):** Фиксированный размер, перекрываются. Определяются размером окна и интервалом сдвига. `[0, 10), [5, 15), [10, 20) ...`
**Session Windows (Сессионные):** Определяются интервалом неактивности (gap). Окно закрывается, если в течение заданного времени не поступает новых событий. Размер окна переменный. Часто используются для пользовательских сессий.
**Global Windows:** Одно окно для всего потока (без ограничения по времени/количеству). Требует кастомного триггера.
После определения окна, применяются агрегации (`sum`, `count`, `reduce`, `aggregate`) или более мощный `process` метод.
Триггеры (Triggers): Определяют, когда результаты окна должны быть вычислены и отправлены ниже по потоку. По умолчанию для Event Time окон используется EventTimeTrigger на основе Watermarks.
Эвикторы (Evictors): (Опционально) Удаляют элементы из окна перед или после срабатывания триггера.
3.  **Joins (Соединения потоков):**
Соединение двух потоков данных.
Могут быть основаны на окнах (Window Joins) или на интервалах времени (Interval Joins).
Требуют управления состоянием для хранения элементов из одного потока в ожидании соответствующих элементов из другого.
4.  **ProcessFunction / CoProcessFunction:**
Низкоуровневые операторы, предоставляющие максимальную гибкость.
Позволяют получить доступ к Event Time, Watermarks, метаданным событий.
Главное: предоставляют доступ к Timers (таймеры), которые позволяют запланировать выполнение кода в будущем Event Time или Processing Time. Критично для реализации сложных паттернов, обнаружения бездействия и т.д.
Могут работать с Keyed State.
5.  **Table API & SQL API:**
Высокоуровневые абстракции над DataStream API.
Представляют потоки как динамические таблицы, которые постоянно обновляются.
Позволяют писать запросы к потокам данных с использованием реляционных операций (SELECT, FROM, WHERE, GROUP BY, JOIN) на Scala, Java, Python (Table API) или стандартном SQL.
Удобны для ETL/ELT задач в потоковом режиме, аналитики.
Flink преобразует запросы Table API/SQL в эффективные программы DataStream API.

**Демонстрация:**
Пример кода с использованием `keyBy` и `count` в окне (Tumbling Window).
(Опционально) Пример использования `ProcessFunction` с таймером.
(Опционально) Пример простого SQL-запроса к потоку.

## Архитектура и развертывание

1.  **Компоненты Архитектуры:**
**JobManager:** Главный координатор. Отвечает за:
- Прием Flink job от клиента.
- Оптимизацию и преобразование job graph в execution graph.
- Планирование задач на TaskManagers.
- Координацию чекпоинтов.
- Мониторинг TaskManagers.
- Предоставление Web UI.
**TaskManagers:** Рабочие узлы. Отвечают за:
- Выполнение задач (tasks) из execution graph.
- Управление *слотами* (slots) – единицами ресурсов (память, CPU).
- Хранение и управление состоянием через State Backend.
- Взаимодействие друг с другом для пересылки данных между операторами.
**Client:** Отправляет Flink job в JobManager.
2.  **Parallelism (Параллелизм):**
Flink job разбивается на подзадачи (subtasks), которые выполняются параллельно на разных TaskManager'ах.
Уровень параллелизма можно настроить для каждого оператора или для всей job.
`keyBy()` гарантирует, что все события с одним ключом обрабатываются одним и тем же subtask'ом оператора, что важно для корректного управления Keyed State.
3.  **Slots (Слоты):**
TaskManager делит свои ресурсы на слоты.
Один Task Slot может выполнять одну или несколько subtask'ов одной job (или разных job, в зависимости от конфигурации).
Позволяют эффективно использовать ресурсы TaskManager'а, запуская несколько задач параллельно в рамках одного процесса JVM.
4.  **Развертывание (Deployment Modes):**
**Standalone:** Запуск Flink на выделенных машинах. Простой для тестирования.
**YARN:** Развертывание Flink job на кластере Apache Hadoop YARN. Позволяет динамически выделять ресурсы.
**Kubernetes:** Развертывание Flink job в контейнерах Kubernetes. Современный и гибкий подход.
**Cloud Provider Specific:** Интеграции с облачными платформами (например, AWS Kinesis Analytics for Apache Flink, Google Cloud Dataflow - хотя это скорее конкурент, но показывает интеграцию концепций).
**Native Flink Kubernetes Operator:** Упрощает развертывание и управление Flink job в Kubernetes.


## Сравнение с альтернативами

**Apache Spark Streaming / Structured Streaming:**
Spark Streaming: Микро-батчевый подход. Обрабатывает данные небольшими пакетами. Выше задержка по сравнению с Flink, менее эффективное управление состоянием и Exactly-once гарантии сложнее достичь end-to-end.
Spark Structured Streaming: Более современный API, ближе к Flink в концепциях (Event Time, Watermarks, State). Но по-прежнему основан на микро-батчах под капотом (хотя и скрывает это). Flink часто имеет более низкую задержку и более зрелое управление состоянием для сложных сценариев.

**Apache Storm:**
Один из первых стриминговых движков.
Более низкоуровневый API.
Управление состоянием и Exactly-once гарантии сложнее реализовать и поддерживать по сравнению с Flink.

**Kafka Streams:**
Библиотека для потоковой обработки, *тесно интегрированная с Apache Kafka*.
Проще в использовании для базовых задач обработки потоков из Kafka.
Ограничена Kafka как источником/приемником. Менее гибкая в плане развертывания и выбора State Backend (использует RocksDB или память).
Flink – более универсальный, мощный и гибкий движок для потоковой обработки из *разных* источников с *разным* состоянием и *разными* режимами развертывания.

## Сценарии Использования

Мониторинг и оповещения в реальном времени.
Обнаружение мошенничества.
Анализ кликов и поведения пользователей в реальном времени.
ETL/ELT конвейеры с низкой задержкой.
Обработка данных с IoT-устройств.
Построение постоянно обновляющихся материализованных представлений или витрин данных.
Системы рекомендаций в реальном времени.
