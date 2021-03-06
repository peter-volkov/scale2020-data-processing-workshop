# Yandex Scale 2020: Обработка поточных данных с использованием Managed Service for Apache Kafka® и Data Proc

В этом репозитории содержится код и памятка для практикума по работе с [Managed Service for Apache Kafka®](https://cloud.yandex.ru/services/managed-kafka) и [Data Proc](https://cloud.yandex.ru/services/data-proc) в Яндекс.Облаке.

[Ссылка на мероприятие](https://cloud.yandex.ru/events/scale-2020/workshops#real-time-processing)

## Стуктура репозитория
В этом репозитории есть исходные файлы:
 * `Makefile` -- файл для создания нужного окружения на vm, для запуска producer.py
 * `producer.py` --  скрипт для загрузки данных в Kafka сырых событий.
 * `streaming.py` -- pyspark задача для обработки поточных сырых событий.
 * `clickhouse.sql` -- DDL для ClickHouse, для экспорта обработанных данных в витрину.

## Основные шаги практикума:
#### 1. Создайте ssh-keys
Для работы с виртуальными машинами нам потребуется ssh ключ.
Если у вас еще нет пары ключей, то вы можете [создать новые ssh-keys](cloud.yandex.ru/docs/compute/operations/vm-connect/ssh#creating-ssh-keys) по документации Яндекс.Облака.

#### 2. Создайте новую сеть в Virtual Private Network
1. Перейдите в сервис VPC и нажмите кнопку `Создать сеть`.
2. Укажите уникальное имя, выставьте флаг `Создать подсети` и создайте сеть.
3. Перейдтие в созданную сеть и для новых подсетей через настройки нажмите `Включить NAT в интернет`. NAT нужен для Data Proc кластера для доступа к Object Storage и отправки своего статуса.

#### 3. Создайте сервисный аккаунт
1. Вернитесь в каталог и нажмите слева на вкладку `Сервисные пользователи`.
2. Нажмите `Создать сервисный аккаунт` справа.
3. Введите имя сервисного аккаунта для кластера Data Proc, например `streaming-service-account`.
4. Добавьте роли `mdb.dataproc.agent` для отправки метрик и `storage.uploader`, `storage.viewer` для работы с Object Storage.

#### 4. Создайте бакет в Объектом Хранилище.
1. Вернитесь в каталог и нажмите на `Object Storage`.
2. Нажмите на кнопку `Создать бакет`.
3. Укажите уникальное имя, для уникальности можно добавить суффикс своего аккаунта и нажмите кнопку `Создать бакет`.

#### 5. Создайте кластер Kafka
1. Вернитесь в каталог и нажмите на `Managed Service for Kafka` и кнопку `Создать кластер`.
2. Укажите имя.
3. Укажите *класс хоста* `s2.micro` в 2 ядра и 8ГБ памяти.
4. В `сетевых настройках` укажите используемую зону доступности, чтобы создать только 1 инстанс Kafka.
5. Количество брокеров в зоне отавьте равным `1`, чтобы создать только 1 инстанс кафки в одной зоне доступности.
6. Нажмите кнопку `Создать`.

#### 6. Создайте кластер Data Proc
1. Вернитесь в каталог и нажмите на `Data Proc` и кнопку `Создать кластер`.
2. Укажите имя.
3. Выберете последнюю доступную версию, на момент практикума это `1.3`.
4. Укажите необходимые сервисы. Нам потребуется `HDFS`, `YARN`, `SPARK` и `ZEPPELIN`. Код будем запускать на pyspark, а Zeppelin пользоваться как IDE.
5. Укажите публичную часть ключа в поле `ssh-ключ`.
6. Сервисный аккаунт должен быть выбран автоматически, укажите созданный вручную, если он не указан.
7. Выставьте зону доступности, для которой вы включили `NAT`. В нашем практикуме мы будем создавать все мощности только в одной из зон доступности, выберите по желанию.
8. Бакет должен быть выбран автоматически, укажите созданный вручную, если он не указан.
9. Выставьте флаг `UI Proxy` для проксирования интерфейсов кластера через отдельный endpoint. Он потребуется для написания кода на pyspark и отладки работы spark задания.
10. В поле подкластеров у вас будет 2 подкластера с ролями `Мастер` и `Data`.
11. Для мастер-подкластера через настройки можете задать имя.
12. Укажите *класс хоста* `s2.micro` в 2 ядра и 8ГБ памяти.
13. Размер хранилища можно уменьшить до 32ГБ и сохраните изменения.
14. Для `Data` подкластера так же нажмите на редактирование.
15. Выставьте 2 хоста.
16. Выставьте *класс хоста* в 2 ядра и 8ГБ памяти.
17. Хранилище можно переключить так же на `network-ssd` и уменьшить его до 64 ГБ.
18. Сохраните изменения.
19. Создайте кластер.

#### 7. Создайте кластер ClickHouse
1. Вернитесь в каталог и нажмите на `Managed Service for ClickHouse` и кнопку `Создать кластер`.
2. Укажите имя.
3. Выставьте последнюю доступную версию `20.6`.
4. Укажите *класс хоста* `s2.micro` в 2 ядра и 8ГБ памяти.
5. Укажите имя базы данных, например, `scale2020`.
6. Укажите имя пользователя, например, `clickhouse`.
7. Укажите пароль для пользователя.
8. Выставьте используемую зону доступности, публичный доступ не нужен.
9. Выставьте в дополнительных настройках флаги `Доступ из DataLens` и `Доступ из консоли управления`.
10. В настройках СУБД нажмите кнопку `Настроить`.
11. Пролистайте настройки до секции `Kafka` и раскройте их.
12. Укажите:
  * `SASL_MECHANISM` как `SCRAM-SHA-512`,
  * `SASL_PASSWORD` как пароль, с которым ClickHouse будет ходить в Kafka.
  * `SASL_USERNAME` логин, с которым ClickHouse будет ходить в Kafka.
  * `SECURITY_PROTOL`, как `SASL_PLAINTEXT`.
13. Нажмите на `Сохранить` и `Создать кластер`.

#### 8. Создайте топики в Kafka
1. Вернитесь в ваш кластер Kafka.
2. Откройте вкладку `Топики` и нажмите на `Создать топик`.
3. Создайте топик `raw` для сырых данных. Фактор репликации нужно оставить в `1`, т.к. у нас всего 1 инстанс. Количество разделов (партиций) лучше уменьшить до `3`, они нам сейчас не очень нужны.
4. В настройках можно выставить политику очистки в `DELETE` и `время жизни сегмента` в `3600000` мс, но это опционально.
5. Создайте такой же топик `combined` для общения Data Proc и ClickHouse.

#### 9. Создайте пользователей в Kafka
1. Переключитесь на вкладку `Пользователи` и нажмите на `Создать пользователя`.
2. Создайте первого пользователя с именем `ingestion`, который будет публиковать события в топик `raw`. Для этого выдайте на топик `raw` право `ACCESS_ROLE_PRODUCER`.
3. Создайте пользователя `dataproc`, который будет вычитывать данные из топика `raw` и записывать в `combined`. Для этого выдайте на топик `raw` право `ACCESS_ROLE_CONSUMER` и на топик `combined` право `ACCESS_ROLE_PRODUCER`.
2. Создайте первого пользователя с именем `clickhouse`, который будет вычитывать события из топика `raw` в ClickHouse. Для этого выдайте на топик `combined` право `ACCESS_ROLE_CONSUMER`.

#### 10. Создайте виртуальную машину, которая будет импортировать новые события.
1. Вернитесь в каталог и переключитесь на сервис `Compute Cloud`.
2. Нажмите `Создать ВМ`.
3. Укажите имя, выберите ОС `ubuntu-20.04 lts`.
4. Диск можно уменьшить до 8ГБ.
5. Гарантированную долю CPU можно уменьшить до 20%.
6. RAM можно уменьшить до 1ГБ.
7. Публичный адрес выставить `Автоматически`.
8. Выставить логин `ubuntu` и публичную часть вашего ssh-ключа.
9. Создать.

#### 11. Импортирование потока событий
1. Откройте терминал и зайдите на виртуальную машину по ssh.
2. Выполните команду `sudo apt update && sudo apt install git make screen python3-dev python-venv`, которая установит команды, необходимые для запуска скрипта.
3. Выполните `git clone git@github.com:epikhinm/scale2020-data-processing-workshop.git; cd scale2020-data-processing-workshop`
4. Установите зависимости с помощью команды `make venv`.
5. Запустите `screen` и в нем выполните `. venv/bin/activate`, для того чтобы начать использовать python и зависимости из собранного venv.
6. Скопируйте адрес хоста kafka из UI и выполните в терминале команду: `KAFKA_BROKERS="<kafka_host>:9091" KAFKA_PASS="ingestion_password" KAFKA_USER="ingestion" python producer.py`

У вас запустился скрипт, который в цикле забирает события и отправялет их в Kafka.

### 12. Отладка и запуск pyspark задания, по обработке потока
1. Вернитесь в ваш Data Proc кластер.
2. Откройте ссылку `Zeppelin Web UI`.
3. У вас откроется Web IDE, где удобно разрабатывать и отлаживать код. Нажмите `Create new note`.
4. Скопируйте код и файла [streaming.py](https://github.com/epikhinm/scale2020-data-processing-workshop/blob/master/streaming.py), впишите в note первой строкой `%pyspark` и вставьте содержимое `streaming.py`.
5. Запустите заметку кнопкой `Play`.

### 13. ClickHouse DDL
1. Вернитесь в ваш ClickHouse кластер.
2. Откройте вкладку `SQL` и подключитесь.
3. Для создания таблицы, которая будет импортировать данные из Kafka создайте таблицу со следующим кодом (замените broker на актульный адрес Kafka):
  ```sql
CREATE TABLE scale2020.queue (
    timestamp DateTime,
    location_id UInt32,
    latitude Float64,
    longitude Float64,
    country FixedString(2),
    temperature Nullable(Float64) DEFAULT NULL,
    humidity Nullable(Float64) DEFAULT NULL,
    pressure Nullable(Float64) DEFAULT NULL,
    P1 Nullable(Float64) DEFAULT NULL,
    P2 Nullable(Float64) DEFAULT NULL
) ENGINE = Kafka
  SETTINGS kafka_broker_list = 'broker:9092',
           kafka_topic_list = 'combined',
           kafka_group_name = 'queue',
           kafka_format = 'CSV',
           kafka_skip_broken_messages = 1,
           kafka_num_consumers = 1,
           kafka_max_block_size = 1048576;
```
4. Для создания таблицы с историческими данными, создается вторую таблицу:
```sql
CREATE TABLE scale2020.air_quality (
    timestamp DateTime,
    location_id UInt32,
    latitude Float64,
    longitude Float64,
    country FixedString(2),
    temperature Nullable(Float64) DEFAULT NULL,
    humidity Nullable(Float64) DEFAULT NULL,
    pressure Nullable(Float64) DEFAULT NULL,
    P1 Nullable(Float64) DEFAULT NULL,
    P2 Nullable(Float64) DEFAULT NULL
) ENGINE = MergeTree
PARTITION BY toYYYYMM(timestamp)
ORDER BY (timestamp);
```
5. Для того чтобы в фоне ClickHouse перекладывал данные из очереди Kafka в историческую таблицу создайте `MATERIALIZED VIEW`:
```sql
CREATE MATERIALIZED VIEW scale2020.air_quality_mv TO scale2020.air_quality AS
SELECT * FROM scale2020.queue;
```

6. Теперь в таблицу `scale2020.air_quality` будут все собранные и очищенные записи.

