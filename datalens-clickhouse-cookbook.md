# Кукбук: DataLens + ClickHouse

Черновик. Цель — короткая дорожка, позволяющая обойти типовые ошибки при 
использовании ClickHouse как источника данных для DataLens.

Все примеры выполняются на одном сквозном наборе данных: таблица
`financial_transactions` на 50 млн строк.

> **Об условиях замеров.** Все `-- N rows in set. Elapsed...` в этом файле
> получены на реальном инстансе Managed Service for ClickHouse минимальной
> конфигурации (8 vCPU / 32 GB, как рекомендовано в 0.1). Это ориентировочные
> цифры, а не результат строгого бенчмарка:
> - кэши (mark, uncompressed, filesystem) между сравниваемыми запросами не
>   сбрасывались — `SYSTEM DROP CACHE` не выполнялся;
> - на запрос — один прогон, без warm-up и без усреднения по нескольким
>   попыткам;
> - инстанс шэренный (managed-сервис), в моменте замеров конкурентная
>   нагрузка от чужих запросов была низкой, но не нулевой.
>
> Ориентируйтесь на порядок величин и на то, какой путь чтения выбрал
> оптимизатор, а не на конкретные секунды — на своих данных и своём железе
> числа будут другими.

## Оглавление

- 00. Почему под BI берут колоночную СУБД
- 0. Развернуть ClickHouse
  - 0.1 Managed Service for ClickHouse
  - 0.2 Своя установка
  - 0.3 Версии и обновления
- 1. Проектирование и подготовка таблиц
  - 1.1 Где выполнять работы
  - 1.2 Широкая денормализованная таблица
  - 1.3 ORDER BY и PRIMARY KEY
  - 1.4 Дата в ключе сортировки
  - 1.5 Партиционирование
  - 1.6 Практика: создание таблицы и наполнение
  - 1.7 Практика: два запроса для сравнения
- 2. Настройка таблиц - проекции
  - 2.1 Что бывает
  - 2.2 Ограничение: проекция живёт внутри парта
  - 2.3 DISTINCT и селекторы DataLens
  - 2.4 Нюансы движков
  - 2.5 Практика: четыре проекции
  - 2.6 Практика: проверка использования
- 3. Настройка таблиц - скип-индексы
  - 3.1 Когда работают
  - 3.2 Практика: minmax индексы для числовых колонок
  - 3.3 Практика: bloom filter и OR по двум колонкам
- 4. Использование таблиц - не страшный JOIN и словари
  - 4.1 Справочник и «внешний ключ»
  - 4.2 Словарь — более эффективный метод присоединения семантики
  - 4.3 JOIN против dictGet
  - 4.4 Практика: замеры
- 5. Использование таблиц - представления и оконные функции
  - 5.1 Представление
  - 5.2 Стоимость окон
  - 5.3 Запрос с одной оконной функцией
  - 5.4 Селекторы поверх представления
- 6. Производные таблицы - материализованные представления
  - 6.1 Инкрементальный MV — триггер на вставку
  - 6.2 AggregatingMergeTree и состояния агрегатов
  - 6.3 Refreshable MV: REPLACE и APPEND
  - 6.4 Что выбирать
- 7. Мониторинг BI-нагрузки
- 8. Использование таблиц и витрин в Datalens
  - 8.1 Практика: датасет и чарт на представлении с оконными функциями
  - 8.2 Задел: параметры датасета и параметризированные представления в БД

---

## 00. Почему под BI берут колоночную СУБД

- Профиль нагрузки BI — OLAP: агрегации по сотням миллионов строк, при этом в
  запросе участвуют единицы колонок из десятков имеющихся. Так устроен почти
  любой график: одно-два измерения, одна-две меры.
- Строковая СУБД читает строку целиком, включая ненужные запросу колонки.
  Колоночная читает только затронутые колонки — объём чтения падает
  пропорционально доле используемых полей.
- В пределах колонки данные однородны, поэтому общие алгоритмы сжатия (LZ4,
  ZSTD) дают заметно лучший результат, чем на строковом представлении, а
  специализированные кодеки (Delta, DoubleDelta, Gorilla, T64) дожимают
  монотонные ряды. Эффект двойной: экономия диска и ускорение чтения.
- Обработка идёт блоками (векторизация), с эффективным использованием кэшей CPU.
- Итог: на одинаковых ресурсах разница со строковой СУБД измеряется не
  процентами, а кратами.

ClickHouse — одна из лучших реализаций этого подхода: open source, линейное
масштабирование, нативный коннектор в DataLens.

---

## 0. Развернуть ClickHouse

Если у вас уже есть установленный ClickHouse версии не ранее 26.3.16, доступный для 
тестовых операций с параметрами не менее 8CPU/32RAM, переходите к пункту 0.3.

### 0.1 Managed Service for ClickHouse

Подходит, если нет навыков самостоятельного развёртывания и администрирования
либо самой цели держать свою инфраструктуру. Глубоко вникать в настройки не
потребуется: кластер разворачивается уже с конфигурацией, подходящей для
большинства задач. Значения по умолчанию без причины не менять.

Второй аргумент — гибкость ресурсов. Класс хоста и объём диска меняются в
несколько кликов, поэтому можно начать пилотный проект на базовых ресурсах и
расширяться по мере роста проекта и объёма данных, а не закладывать
инфраструктуру «на вырост» сразу.

Порядок действий в консоли Yandex Cloud:

1. Managed Service for ClickHouse → **Создать кластер**.
2. **Версия** — см. 0.3.
3. **Класс хоста** — минимальный рабочий вариант по рекомендациям разработчиков
   ClickHouse: 8 vCPU / 32 GB RAM.
4. **Хранилище** — от пары сотен ГБ. Помимо запаса под данные и мержи, в
   Managed Services размер диска напрямую связан со скоростью чтения:
   производительность сетевого диска растёт вместе с объёмом.
5. **Хосты** — один хост для пилота и типовых задач. При требовании повышенной
   отказоустойчивости — кластерная конфигурация из нескольких хостов с
   ClickHouse Keeper.
6. **База данных и пользователь** — создать на этом же шаге. Для BI завести
   отдельного пользователя, только на чтение нужной БД.
7. **Настройки доступа** — включить доступ из DataLens и доступ через
   WebSQL / консоль управления.

### 0.2 Своя установка

Если ClickHouse ставится самостоятельно — официальный репозиторий (`apt`/`yum`)
или официальный docker-образ. Дальше в кукбуке разницы нет: всё описанное
одинаково работает и в Managed Service, и на своей инсталляции.

### 0.3 Версии и обновления

Ставьте свежие версии и следите за обновлениями. Релизы выходят ежемесячно, и
заметная доля изменений — это оптимизации именно тех сценариев, которые
генерирует BI: агрегации, JOIN, DISTINCT, фильтрация по ключу сортировки,
работа проекций. Разрыв в год-полтора по версиям означает, что часть проблем с
производительностью вы решаете руками там, где новая версия решает их сама.

Но «свежая» не значит «только что вышедшая»:

- Брать линию LTS, с релиза которой прошло 2-3 месяца — за это время
  устраняются основные проблемы релиза. То есть не 26.3.1, а 26.3.17 на текущий
  момент.
- Продуктив — плановое обновление на следующую LTS по той же логике.
- Перед обновлением просматривать чейнджлог, в первую очередь раздел backward
  incompatible changes:
  <https://github.com/ClickHouse/ClickHouse/tree/master/docs/changelogs>

---

## 1. Проектирование и подготовка таблиц

### 1.1 Где выполнять работы

Проектирование и проверку выполнять не в BI-интерфейсе, а там, где сразу видна
цена запроса:

- **clickhouse-client** — после каждого запроса выводит время выполнения,
  количество прочитанных строк и объём. Основной рабочий инструмент.
- **WebSQL в консоли Yandex Cloud** — те же данные доступны во вкладке
  метаданных.

Дашборд как измерительный прибор не годится: в нём не видно, сколько строк
реально прочитано и какой план был выбран.

### 1.2 Широкая денормализованная таблица

Основа подхода — денормализованные широкие таблицы, содержащие и колонки
семантики (справочные данные, измерения), и множество колонок показателей.

Причина: сильная сторона ClickHouse — сканирование и агрегация одной таблицы, а
JOIN остаётся относительно дорогой операцией. При этом BI генерирует SQL сам, и
вручную оптимизировать его план возможности нет.

Разумные пределы:

- Избыточное количество семантических колонок усложняет оптимизацию структуры:
  сложнее подобрать `ORDER BY`, обслуживающий все сценарии.
- Меняющаяся семантика справочных данных иногда не позволяет внести их в широкую
  таблицу: записанные в факт значения окажутся зафиксированными на момент
  вставки. Если справочник переименовывают, переклассифицируют или
  реорганизуют, а отчётность должна показывать актуальное состояние, семантику
  оставляют снаружи — в справочной таблице или словаре (раздел 4).
- Массивы и JSON бывают удобны (переменный набор атрибутов, параметры событий,
  теги), но обходятся дороже при выборке и требуют в DataLens дополнительных
  вычисляемых полей на распаковку. Точечно, а не как способ не проектировать
  схему.
- Дублирование справочных значений в широкой таблице обычно не проблема:
  низкокардинальные строки хорошо сжимаются (колонки с менее чем 10 000
  уникальных значений — хорошие кандидаты для LowCardinality).

### 1.3 ORDER BY и PRIMARY KEY

В ClickHouse нет первичных и внешних ключей в привычном смысле, нет индексов
вроде b-tree, нет ограничений целостности. Первичный ключ ничего не гарантирует
по уникальности.

Основа — семейство движков MergeTree, а его краеугольный камень — сортировка
`ORDER BY`. Она определяет, как данные физически хранятся на диске, и то, как
система оптимизирует их выборку: отсечение засечек при фильтрации,
эффективность сжатия соседних значений, возможность агрегации без пересортировки.

`PRIMARY KEY` — подмножество-префикс `ORDER BY`: та часть ключа сортировки, по
которой строится разреженный индекс, хранимый в оперативной памяти. Разреженный
— потому что содержит по одной записи на гранулу (по умолчанию 8192 строки).
Разделять `PRIMARY KEY` и `ORDER BY` имеет смысл, когда сортировать нужно
глубже, чем индексировать: хвостовые колонки улучшают сжатие и локальность, но
не раздувают индекс в памяти.

Базовый вариант выбора порядка — поля измерений по возрастанию кардинальности:
Страна → Город → Улица → Дом. Получается вложенность, как в структуре папок.

Но отталкиваться нужно от сценариев использования, а не от кардинальности как
таковой:

- Если поле очень редко участвует в условиях отбора, его размещение впереди даёт
  ненужные накладные расходы. Если выборка всегда строится по названию Города,
  нет смысла проверять его наличие в каждой Стране.
- Поля можно исключать из `ORDER BY` совсем, если они не участвуют ни в
  фильтрах, ни в группировках.
- Поля можно менять местами вопреки кардинальности, если этого требует типовой
  фильтр.
- Предпочитайте колонки, которые исключают наибольший процент строк при
  фильтрации, а не просто имеют наименьшую кардинальность.
- Ключ держать коротким: 3-5 колонок. Каждая следующая всё меньше влияет на
  отсечение и всё больше — на стоимость мержей.
- Один `ORDER BY` не обслужит все сценарии. Если сценарии конфликтуют — это
  повод не на компромисс, а на проекцию (раздел 2).

В процессе жизненного цикла BI-системы подход к организации сортировок и прочих
конструкций оптимизации выборки стоит пересматривать вслед за меняющимися
сценариями использования.

### 1.4 Дата в ключе сортировки

 Большинстве случаев аналитическая обработка связана с интервалами дат и времени и поля с ними - очевидный кандидат в ORDER BY.

Типовая ошибка.

- Поле типа `Date` на первой позиции оправдано только если на каждую дату
  приходятся миллионы записей с разными значениями других измерений.
- Если это дата со временем, первая позиция может стать фатальной.
  Кардинальность близка к числу строк, каждая гранула получает уникальный
  диапазон, и все последующие колонки ключа перестают работать на отсечение.
  Сжатие остальных колонок тоже ухудшается.
- Разумное решение — помещать в начало не саму дату, а её огрубление:
  `dateTrunc('month', ...)`, `toStartOfWeek(...)`, `toYYYYMM(...)`. Точное время
  остаётся отдельной колонкой и может стоять в хвосте ключа.

Исключения, когда таймстемп или уникальный идентификатор на первом месте
оправдан:

- временные ряды — запрос по интервалу времени и есть основной сценарий;
- быстрый поиск по конкретному id — точечная выборка, а не аналитика по срезам.

В BI такие случаи встречаются, но они не типовые. Не проектируйте под них
таблицу, которая обслуживает дашборды.

### 1.5 Партиционирование

Партиционирование давно перестало быть инструментом оптимизации выборок. Это
часть механизма управления данными: перенос и отсоединение партиций, удаление
старых данных, обработка по партициям, бэкапы.

Для ускорения запросов `toYYYYMM(dt)` в начале ключа сортировки, как правило,
полезнее, чем такое же выражение в `PARTITION BY`. Логика та же, что со
странами: партиция — дополнительный верхний уровень, по которому система
вынуждена ходить. Каждая партиция хранится отдельно, и запрос, охватывающий
несколько периодов, заглядывает в каждую из них. При мелком партиционировании
получается множество мелких кусков, лишние накладные расходы на планирование и
мержи — и никакого выигрыша, которого не дал бы `ORDER BY`. При этом, вполне 
допустимо сделать партицию по полю даты с округлением и не помещать это поле в 
`ORDER BY`.

Практика: партиций немного и они крупные. По месяцам — норма; по дням — только
если действительно нужно посуточно перекладывать данные.

### 1.6 Практика: создание таблицы и наполнение

```sql
-- ============================================================
-- 1. СОЗДАНИЕ ТАБЛИЦЫ
-- ============================================================
-- PRIMARY KEY — компактный индекс в RAM (только 2 колонки)
-- ORDER BY   — физическая сортировка на диске (4 колонки)
-- PRIMARY KEY должен быть префиксом ORDER BY
-- Порядок ключей выбран по логике запросов, НЕ по кардинальности:
--   merchant_category (~10)  — самый частый фильтр в аналитике
--   country           (~15)  — второй по частоте фильтр
--   month             (~36)  — диапазонные фильтры по периоду
--   account_id        (~1M)  — точечные запросы по клиенту
-- payment_method и status (кардинальность 4) НЕ стоят первыми —
-- фильтр WHERE payment_method = 'card' захватывает ~25% строк
-- и почти не помогает сократить объём чтения, тогда как
-- merchant_category = 'Travel' сразу отсекает ~90% данных.
-- Правило: предпочитайте колонки которые исключают наибольший
-- процент строк при фильтрации, а не просто имеют наименьшую кардинальность.

CREATE TABLE financial_transactions
(
    -- Измерения (dimensions)
    transaction_id    UInt64,
    account_id        UInt32,                 -- ~1M уникальных (высокая кардинальность)
    merchant_category LowCardinality(String), -- ~10 категорий  (низкая кардинальность)
    payment_method    Enum8(
                          'card'     = 1,
                          'transfer' = 2,
                          'cash'     = 3,
                          'crypto'   = 4
                      ),                      -- 4 значения (очень низкая кардинальность)
    country           LowCardinality(String), -- ~15 стран      (средняя кардинальность)
    currency          LowCardinality(String), -- ~7 валют       (низкая кардинальность)
    transaction_date  Date,                   -- ~1095 дней     (средняя кардинальность)
    status            Enum8(
                          'completed' = 1,
                          'pending'   = 2,
                          'failed'    = 3,
                          'refunded'  = 4
                      ),                      -- 4 значения (очень низкая кардинальность)

    -- Показатели (metrics)
    amount            Decimal(18, 2),
    fee               Decimal(10, 4),
    exchange_rate     Float32,
    cashback_amount   Decimal(10, 4),
    risk_score        UInt8                   -- 0–100
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(transaction_date)
PRIMARY KEY (merchant_category, country)
ORDER BY (merchant_category, country, dateTrunc('month', transaction_date), account_id);
```

```sql
-- ============================================================
-- 2. ЗАПОЛНЕНИЕ ДАННЫМИ
-- ============================================================
INSERT INTO financial_transactions
SELECT
    rowNumberInAllBlocks() + 1                                AS transaction_id,
    toUInt32(rand() % 1000000)                                AS account_id,
    arrayElement(
        ['Retail', 'Food & Beverage', 'Travel', 'Healthcare',
         'Entertainment', 'Utilities', 'E-commerce', 'Finance',
         'Education', 'Automotive'],
        (rand() % 10) + 1
    )                                                         AS merchant_category,
    arrayElement(
        ['card', 'transfer', 'cash', 'crypto'],
        (rand() % 4) + 1
    )                                                         AS payment_method,
    arrayElement(
        ['US', 'DE', 'FR', 'GB', 'JP', 'CN', 'BR', 'IN', 'RU', 'AU',
         'CA', 'MX', 'KR', 'IT', 'ES'],
        (rand() % 15) + 1
    )                                                         AS country,
    arrayElement(
        ['USD', 'EUR', 'GBP', 'JPY', 'CNY', 'RUB', 'BRL'],
        (rand() % 7) + 1
    )                                                         AS currency,
    today() - 1095 + toInt32(rand() % 1095)                   AS transaction_date,
    arrayElement(
        ['completed', 'pending', 'failed', 'refunded'],
        (rand() % 4) + 1
    )                                                         AS status,
    toDecimal64(10 + (rand() % 9999000) / 100.0, 2)           AS amount,
    toDecimal64((rand() % 500) / 10000.0, 4)                  AS fee,
    toFloat32(0.5 + (rand() % 200) / 100.0)                   AS exchange_rate,
    toDecimal64((rand() % 200) / 10000.0, 4)                  AS cashback_amount,
    toUInt8(rand() % 101)                                     AS risk_score
FROM numbers(50000000);
```

```sql
-- Конкретные значения кардинальности можно получить запросом
SELECT
    uniq(transaction_id)    AS card_transaction_id,
    uniq(account_id)        AS card_account_id,
    uniq(merchant_category) AS card_merchant_category,
    uniq(payment_method)    AS card_payment_method,
    uniq(country)           AS card_country,
    uniq(currency)          AS card_currency,
    uniq(transaction_date)  AS card_transaction_date,
    uniq(status)            AS card_status
FROM financial_transactions;
```

### 1.7 Практика: два запроса для сравнения

```sql
-- ============================================================
-- 3. ПРИМЕРЫ ЗАПРОСОВ К БАЗОВОЙ ТАБЛИЦЕ
-- ============================================================

-- Запрос 1: фильтр по merchant_category + country — первые два ключа PRIMARY KEY,
-- ClickHouse эффективно пропускает ненужные гранулы
SELECT
    transaction_date,
    count()              AS tx_count,
    sum(amount)          AS total_amount,
    avg(fee)             AS avg_fee,
    avg(risk_score)      AS avg_risk,
    sum(cashback_amount) AS total_cashback
FROM financial_transactions
WHERE merchant_category = 'Travel' -- проверьте наличие комбинации в синтетических данных
  AND country = 'FR'               -- отдельным запросом
GROUP BY transaction_date
ORDER BY transaction_date;
-- Ориентировочные параметры выполнения в clickhouse-client
-- 73 rows in set. Elapsed: 0.066 sec. Processed 2.75 million rows, 79.07 MB (41.33 million rows/s., 1.19 GB/s.)
-- Peak memory usage: 19.03 MiB.

-- Запрос 2: фильтр только по payment_method — не входит в PRIMARY KEY/ORDER BY,
-- поэтому полный скан партиций (демонстрация неэффективного случая)
SELECT
    merchant_category,
    count()        AS tx_count,
    sum(amount)    AS total_amount
FROM financial_transactions
WHERE payment_method = 'card'
GROUP BY merchant_category
ORDER BY total_amount DESC;

-- 5 rows in set. Elapsed: 0.143 sec. Processed 50.00 million rows, 306.51 MB (350.59 million rows/s., 2.15 GB/s.)
-- Peak memory usage: 824.40 KiB.
```

Сравнивать по строке итога в clickhouse-client (время, прочитано строк) либо во
вкладке метаданных WebSQL.

---

## 2. Настройка таблиц - проекции

Проекция — дополнительный скрытый набор данных внутри той же таблицы,
поддерживаемый автоматически при вставке. Фактически это скрытая таблица со
своим порядком строк, своим первичным индексом и, при необходимости,
инкрементально пересчитываемыми агрегатами.

Проекции особенно удобны в связке с BI-инструментами: запрос всегда адресован
одной таблице, а ClickHouse сам выбирает оптимальный layout. В большинстве
BI-инструментов смена целевой таблицы нетривиальна — даже там, где это
технически возможно (например, через параметры датасета в DataLens), это влечёт
ограничения: таблицы должны поддерживаться в полном соответствии по составу
полей, что создаёт операционную нагрузку.

Обратная сторона: отдельные таблицы-агрегаты (через материализованные
представления, раздел 6) дают гарантированное использование нужного layout — запрос явно
адресован конкретной таблице, и оптимизатор не принимает решение за вас.
Проекции же используются по усмотрению оптимизатора, и выбор базовой таблицы
вместо проекции — ситуация не такая уж редкая. Поэтому проверка обязательна
(2.6).

### 2.1 Что бывает

- **Альтернативная сортировка** (normal). Копия выбранных колонок в другом
  порядке сортировки со своим первичным индексом. Внутренний движок MergeTree
  (нет `GROUP BY`, только `ORDER BY`).
- **Предагрегация** (aggregate). Результаты `GROUP BY`, подготовленные заранее.
  Внутренний движок AggregatingMergeTree — назначается автоматически, если в
  проекции есть `GROUP BY`.
- **Индексоподобная** (`_part_offset`, с 25.5). Хранится только ключ сортировки
  плюс указатель на позицию строки в базовом парте; проекция работает как
  вторичный индекс, данные читаются из базовой таблицы. Режимы можно смешивать.
  С 25.6 несколько таких проекций применяются к одному запросу с несколькими
  фильтрами. С 26.1 доступен компактный синтаксис
  `PROJECTION by_town INDEX town TYPE basic`.

### 2.2 Ограничение: проекция живёт внутри парта

Проекция хранится внутри тех же партов основной таблицы. Отсюда следствие: её
сортировка и её индекс локальны для парта, и в лучшем случае читается по одному
диапазону на каждый парт, где есть подходящие строки.

- Работает, когда значения ключа проекции сгруппированы: конкретное значение
  встречается не во всех партах, а в немногих. Такая группировка возникает из
  корреляции с порядком вставки (типично для дат и монотонных id) либо с
  основной сортировкой.
- Если ключ проекции распределён по партам равномерно и независимо от всего
  остального, высокоселективный фильтр найдёт совпадения в каждом парте, и
  накладные расходы съедят выигрыш. Особенно для `_part_offset`-варианта, где
  после поиска нужно ещё сходить в базовую таблицу.
- Мелкие партиции и много мелких партов бьют по проекциям сильнее, чем по
  обычным запросам: стоимость умножается на число партов.

### 2.3 DISTINCT и селекторы DataLens

С 25.11 проекция с предагрегацией используется и для запросов с `DISTINCT`: если
получение всех уникальных значений требует чтения меньшего числа строк из
проекции, ClickHouse возьмёт её автоматически.

Почему это важно именно в паре с DataLens: каждый селектор на дашборде — это
отдельный запрос с `DISTINCT` по измерению. Селекторов обычно несколько, и они
выполняются при каждом открытии дашборда, до любых действий пользователя. На
таблице в сотни миллионов строк это несколько сканирований на каждое открытие,
и в глазах пользователя выглядит как «дашборд грузится», хотя сами графики могут
быть быстрыми. Особенно заметно на дашбордах с обилием селекторов, построенных
на представлениях.

Проекция с `GROUP BY` по набору измерений, используемых в селекторах, — один из
самых дешёвых способов улучшить восприятие скорости дашборда. Проверка — в 5.4.

### 2.4 Нюансы движков

MergeTree — семейство, и в движках с заменой и агрегацией использование
проекций имеет особенности:

- С 24.8 настройка `deduplicate_merge_projection_mode` определяет поведение
  проекций при мержах на ReplacingMergeTree, CollapsingMergeTree,
  VersionedCollapsingMergeTree и при `OPTIMIZE DEDUPLICATE`. Значения:
  `throw` (по умолчанию), `drop`, `rebuild`, `ignore`.
- При режиме по умолчанию добавление проекции на таком движке отклоняется с
  требованием выбрать `drop` или `rebuild` явно.
- Причина осторожности — расхождение результатов: агрегат в проекции может быть
  посчитан по строкам до дедупликации, и тогда число по проекции и по таблице
  отличаются.
- По умолчанию лёгкие удаления на таблицах с проекциями завершаются ошибкой;
  поведение настраивается через `lightweight_mutation_projection_mode` (с 24.7):
  `throw`, `drop`, `rebuild`.

Практический вывод: на движках с заменой и агрегацией проекции применимы, но
режим нужно выбрать осознанно и проверить совпадение чисел с базовой таблицей
до того, как результат попадёт на дашборд. `rebuild` при потоке обновлений может
оказаться дорогим.

### 2.5 Практика: четыре проекции

Материализация не ретроактивна: добавление проекции действует только на новые
вставки, для существующих данных нужен `MATERIALIZE PROJECTION` (тяжёлая
операция, прогресс — в `system.mutations`).

```sql
-- ------------------------------------------------------------
-- Проекция 1: агрегаты по категории и стране
-- Ускоряет: COUNT, SUM, AVG по измерениям merchant_category/country
-- Внутренний движок: AggregatingMergeTree (автоматически, т.к. есть GROUP BY)
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_agg_by_category_country
(
    SELECT
        merchant_category,
        country,
        currency,
        status,
        dateTrunc('month', transaction_date) AS month,
        count()                              AS tx_count,
        sum(amount)                          AS total_amount,
        avg(amount)                          AS avg_amount,
        sum(fee)                             AS total_fee,
        sum(cashback_amount)                 AS total_cashback,
        avg(risk_score)                      AS avg_risk_score
    GROUP BY
        merchant_category,
        country,
        currency,
        status,
        month
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_agg_by_category_country;

-- Пример запроса использующего проекцию
SELECT
    merchant_category,
    country,
    count()            AS total_transactions,
    sum(amount)        AS revenue,
    avg(risk_score)    AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC;

-- 30 rows in set. Elapsed: 0.017 sec. Processed 54.10 thousand rows, 3.61 MB (3.12 million rows/s., 208.32 MB/s.)
-- Peak memory usage: 937.27 KiB.

-- Проверка
EXPLAIN indexes = 1
SELECT
    merchant_category,
    country,
    count()        AS total_transactions,
    sum(amount)    AS revenue,
    avg(risk_score) AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC;
```

```sql
-- ------------------------------------------------------------
-- Проекция 2: агрегаты по аккаунту
-- Ускоряет: профиль клиента, история по account_id
-- Внутренний движок: AggregatingMergeTree (автоматически, т.к. есть GROUP BY)
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_agg_by_account
(
    SELECT
        account_id,
        merchant_category,
        payment_method,
        dateTrunc('month', transaction_date) AS month,
        count()                              AS tx_count,
        sum(amount)                          AS total_amount,
        max(amount)                          AS max_amount,
        sum(cashback_amount)                 AS total_cashback,
        avg(risk_score)                      AS avg_risk_score
    GROUP BY
        account_id,
        merchant_category,
        payment_method,
        month
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_agg_by_account;

-- Пример запроса использующего проекцию
-- dateTrunc в GROUP BY — ClickHouse сопоставит с алиасом month в проекции
SELECT
    dateTrunc('month', transaction_date) AS month,
    merchant_category,
    count()                              AS transactions,
    sum(amount)                          AS spent
FROM financial_transactions
WHERE account_id = 42345
GROUP BY month, merchant_category
ORDER BY month;

-- 27 rows in set. Elapsed: 0.031 sec. Processed 937.86 thousand rows, 44.09 MB (30.25 million rows/s., 1.42 GB/s.)
-- Peak memory usage: 11.70 MiB.
```

```sql
-- ------------------------------------------------------------
-- Проекция 3: иная сортировка для фрод-аналитики
-- Ускоряет: ORDER BY risk_score, amount — альтернативный порядок чтения
-- Внутренний движок: MergeTree (нет GROUP BY, только ORDER BY)
-- ВНИМАНИЕ: SELECT * дублирует все данные таблицы на диске!
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
ADD PROJECTION prj_order_by_risk
(
    SELECT *
    ORDER BY (risk_score, amount, transaction_date)
);

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION prj_order_by_risk;

-- Пример запроса использующего проекцию
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 500
ORDER BY risk_score DESC, amount DESC
LIMIT 100;

-- 100 rows in set. Elapsed: 0.053 sec. Processed 5.63 million rows, 94.68 MB (106.19 million rows/s., 1.79 GB/s.)
-- Peak memory usage: 14.27 MiB.
```

Начиная с версии 25.5 доступны проекции-индексы (projection index): они хранят
только ключ сортировки и виртуальную колонку `_part_offset`, по которой строки
читаются из базовой таблицы. Компактный синтаксис `INDEX ... TYPE basic` из
примера ниже доступен с версии 26.1.

```sql
-- ------------------------------------------------------------
-- Проекция 4: lookup по transaction_id (точечный поиск)
-- Ускоряет: WHERE transaction_id = ?
-- Внутренний движок: projection index — хранит только _part_offset
-- ПРЕИМУЩЕСТВО: минимальное потребление дискового пространства
-- может быть использован при ORDER BY (toDate(created_at), transaction_id)
-- при наличии корреляции между created_at и transaction_id
-- ------------------------------------------------------------
ALTER TABLE financial_transactions
    ADD PROJECTION idx_by_transaction_id INDEX transaction_id TYPE basic;

ALTER TABLE financial_transactions
    MATERIALIZE PROJECTION idx_by_transaction_id;

-- Пример запроса использующего проекцию
-- подставьте существующее значение transaction_id (диапазон 1..50 000 000)
SELECT
    transaction_id,
    account_id,
    amount,
    status,
    transaction_date
FROM financial_transactions
WHERE transaction_id = 12345678;

-- 1 rows in set. Elapsed: 0.107 sec. Processed 50.00 million rows, 400.00 MB (467.29 million rows/s., 3.74 GB/s.)
-- Peak memory usage: 5.89 MiB.
-- ВАЖНО: полный скан не исчезает даже на существующем значении. EXPLAIN indexes = 1
-- показывает, что запрос читается из проекции prj_order_by_risk (Condition: true,
-- Parts: 115/115) — planner выбрал более раннюю по порядку добавления normal-проекцию
-- вместо idx_by_transaction_id, и настройка max_projection_rows_to_use_projection_index
-- на выбор не влияет (тестировалось со значением 100000000). Это ровно тот случай,
-- ради которого нужна проверка из 2.6: наличие проекции не гарантирует её использование.
```

### 2.6 Практика: проверка использования

Проекции занимают место на диске и расходуют ресурсы при мержах. Не держите те,
что не используются.

```sql
-- ============================================================
-- 5. ПРОВЕРКА РАБОТЫ ПРОЕКЦИЙ
-- ============================================================

-- Метод 1: EXPLAIN — смотрим ReadFromMergeTree (имя проекции или имя таблицы)
EXPLAIN indexes = 1
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 5000
ORDER BY risk_score DESC, amount DESC
LIMIT 100;
```

```sql
-- Метод 2: FORMAT Null + force_optimize_projection_name
-- Если проекция НЕ используется — запрос упадёт с ошибкой
-- Проверяем prj_order_by_risk
SELECT
    transaction_id,
    account_id,
    country,
    merchant_category,
    amount,
    risk_score
FROM financial_transactions
WHERE risk_score > 90
  AND amount > 5000
ORDER BY risk_score DESC, amount DESC
LIMIT 100
FORMAT Null
SETTINGS force_optimize_projection_name = 'prj_order_by_risk';

-- Ошибки нет — force_optimize_projection_name подтверждает, что
-- prj_order_by_risk реально используется:
-- 100 rows in set. Elapsed: 0.057 sec. Processed 5.63 million rows, 94.71 MB (98.77 million rows/s., 1.66 GB/s.)
-- Peak memory usage: 15.29 MiB.

-- Проверяем prj_agg_by_category_country
-- Обращаемся к базовым колонкам (как в 2.5) — ClickHouse сам сопоставит
-- агрегаты с проекцией
SELECT
    merchant_category,
    country,
    count()         AS total_transactions,
    sum(amount)     AS revenue,
    avg(risk_score) AS risk
FROM financial_transactions
GROUP BY merchant_category, country
ORDER BY revenue DESC
FORMAT Null
SETTINGS force_optimize_projection_name = 'prj_agg_by_category_country';
-- Если проекция не используется — получим ошибку вида:
-- Exception: Projection prj_agg_by_category_country is specified in setting
-- force_optimize_projection_name, but it is not used.

-- 30 rows in set. Elapsed: 0.016 sec. Processed 47.38 thousand rows, 3.17 MB (2.96 million rows/s., 197.85 MB/s.)
-- Peak memory usage: 4.84 MiB.
```

```sql
-- Метод 3: system.query_log — реальная статистика после выполнения
SELECT
    query,
    projections,
    formatReadableQuantity(read_rows) AS read_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
ORDER BY initial_query_start_time DESC
LIMIT 10;

-- 10 rows in set. Elapsed: 0.023 sec. Processed 271.41 thousand rows, 3.78 MB (11.80 million rows/s., 164.38 MB/s.)
-- Peak memory usage: 10.14 MiB.
-- Колонка projections в выводе явно называет использованную проекцию,
-- например test.financial_transactions.prj_agg_by_category_country —
-- это самый прямой способ убедиться в выборе оптимизатора постфактум.
```

---

## 3. Настройка таблиц - скип-индексы

### 3.1 Когда работают

Скип-индексы — дополнение к проекциям и сортировке, а не замена. Основная
область применения: колонки невысокой кардинальности, с хорошей корреляцией с
`ORDER BY` таблицы и неравномерным распределением значений. Тогда индекс быстро
отсекает гранулы, где нужного значения заведомо нет.

Если бы индексируемая колонка была случайной (без корреляции с `ORDER BY`),
значения оказались бы размазаны по всем гранулам, и bloom filter почти не давал
бы выигрыша.

Частный, но интересный случай — фильтр `OR` по двум колонкам: найти транзакции,
где аккаунт участвует в любой роли (отправитель ИЛИ получатель), аналог
`credit_account OR debit_account`. Ключом такой запрос не закрывается.

### 3.2 Практика: minmax индексы для числовых колонок
Хорошие кандидаты:
`transaction_id` — монотонно возрастающий UInt64. Minmax будет очень эффективен, так как значения естественно упорядочены внутри частей.

Сомнительные и плохие кандидаты:
`account_id` — высокая кардинальность (~1M), но значения внутри гранул будут перемешаны (последний элемент `ORDER BY`). Minmax даст слабую селективность — диапазоны min/max в каждой грануле будут широкими и перекрывающимися.

`fee`, `cashback_amount`, `exchange_rate` — зависят от `amount` и `merchant_category`, корреляция с `ORDER BY` слабая. `Minmax` может помочь, но эффект непредсказуем без реального тестирования.
`payment_method`, `status` — всего 4 значения. `Minmax` будет хранить min=1, max=4 почти в каждой грануле — индекс никогда ничего не отсечёт.
https://clickhouse.com/docs/clickstack/managing/performance-tuning#when-to-add-skip-indexes

```sql
-- transaction_id — монотонно возрастает, minmax очень эффективен
-- Особенно полезен для запросов вида WHERE transaction_id BETWEEN X AND Y
ALTER TABLE financial_transactions
    ADD INDEX idx_minmax_transaction_id transaction_id TYPE minmax GRANULARITY 1;

-- Материализация индексов (обязательно после ADD INDEX!)
ALTER TABLE financial_transactions MATERIALIZE INDEX idx_minmax_transaction_id;

-- Проверяем что индекс реально отсекает гранулы
EXPLAIN indexes = 1
SELECT count(), sum(amount)
FROM financial_transactions
WHERE transaction_id = 97234;

-- В выводе ищем секцию Skip:
-- Parts: 36/109   <- чем меньше левое число, тем лучше работает индекс
-- Granules: 2955/6146
```
Если индекс не используется, то, вероятно risk_score и amount не коррелируют с ключом сортировки. 

### 3.3 Практика: bloom filter и OR по двум колонкам

```sql
-- ============================================================
-- 6. BLOOM FILTER ИНДЕКСЫ ДЛЯ OR-ФИЛЬТРАЦИИ ПО ДВУМ КОЛОНКАМ
-- ============================================================
-- Паттерн: найти транзакции где аккаунт участвует в любой роли
-- (отправитель ИЛИ получатель) — аналог credit_account OR debit_account

-- Добавляем колонки
ALTER TABLE financial_transactions
    ADD COLUMN sender_account_id   UInt32 DEFAULT account_id,
    ADD COLUMN receiver_account_id UInt32 DEFAULT toUInt32(rand() % 1000000);

-- Материализуем значения в существующих строках
-- (DEFAULT заполняет только новые вставки, не существующие строки)
ALTER TABLE financial_transactions
    MATERIALIZE COLUMN sender_account_id;

ALTER TABLE financial_transactions
    MATERIALIZE COLUMN receiver_account_id;

-- Добавляем bloom filter индексы
-- false_positive_rate = 0.01: 1% ложных срабатываний
-- GRANULARITY 1: индекс на каждую гранулу (8192 строки)
ALTER TABLE financial_transactions
    ADD INDEX sender_account_bf sender_account_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE financial_transactions
    ADD INDEX receiver_account_bf receiver_account_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

-- Материализуем для существующих данных
ALTER TABLE financial_transactions
    MATERIALIZE INDEX sender_account_bf;

ALTER TABLE financial_transactions
    MATERIALIZE INDEX receiver_account_bf;

-- Проверяем что индексы материализованы
SELECT
    name,
    formatReadableSize(data_compressed_bytes)   AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    marks_bytes
FROM system.data_skipping_indices
WHERE table = 'financial_transactions'
  AND name IN ('sender_account_bf', 'receiver_account_bf');

-- 2 rows in set. Elapsed: 0.078 sec. Processed 15.00 rows, 1.12 KB (192.31 rows/s., 14.40 KB/s.)
-- Peak memory usage: 4.02 MiB.
```

```sql
-- Основной запрос: OR по двум колонкам
-- Найти все транзакции где аккаунт 42345 участвует в любой роли
-- Автоматическое использование обоих bloom filter для OR появилось в версии
-- 25.12.
--
-- На более старых версиях OR между разными колонками не позволяет
-- использовать skip indexes — в этом случае переписываем через UNION ALL:
--
--   SELECT ... FROM financial_transactions
--   WHERE sender_account_id = 42345
--     AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
--   UNION ALL
--   SELECT ... FROM financial_transactions
--   WHERE receiver_account_id = 42345
--     AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
--
-- Каждая ветка UNION ALL выполняется независимо и использует свой
-- bloom filter — sender_account_bf и receiver_account_bf соответственно.
--
-- UNION ALL и параллелизм: ветки выполняются одновременно, но общий пул
-- потоков делится между ними — при UNION из двух запросов каждая ветка
-- получает примерно половину. При большом числе подзапросов в UNION ALL
-- все они открывают буферы чтения одновременно, что может приводить к
-- пиковому потреблению памяти пропорционально числу веток.
-- Настройка max_streams_for_union_step, ограничивающая число одновременно
-- активных потоков в UNION-шаге (снижает пиковую память без потери
-- параллелизма в целом), появилась в версии 26.5 — на более ранних версиях
-- число веток UNION ALL приходится ограничивать самостоятельно.

-- * Выполните запрос, чтобы найти реально существующие сгенеренные значения
-- в sender_account_id и receiver_account_id

SELECT
    transaction_id,
    sender_account_id,
    receiver_account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    status
FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today();

-- 60 rows in set. Elapsed: 0.046 sec. Processed 1.10 million rows, 16.49 MB (23.80 million rows/s., 358.48 MB/s.)
-- Peak memory usage: 14.85 MiB.
```

```sql
-- Проверка через EXPLAIN: оба Skip индекса должны быть в выводе
EXPLAIN indexes = 1
SELECT
    transaction_id,
    sender_account_id,
    receiver_account_id,
    amount,
    status
FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today();

--В выводе должна быть секция вида
--Skip
--  Name: <Combined skip indexes>
--  Description: Final set of granules after AND/OR processing
--  Parts: 16/16
--  Granules: 29/853
```

```sql
-- Замер эффекта: с индексами vs без индексов
SELECT * FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today()
FORMAT Null
SETTINGS use_query_condition_cache = 0;

-- 60 rows in set. Elapsed: 0.052 sec. Processed 1.10 million rows, 23.91 MB (21.06 million rows/s., 459.89 MB/s.)
-- Peak memory usage: 24.75 MiB.

SELECT * FROM financial_transactions
WHERE (sender_account_id   = 42345 -- подставьте существующее значение
    OR receiver_account_id = 42345)
  AND transaction_date BETWEEN today() - INTERVAL 2 YEAR AND today()
FORMAT Null
SETTINGS use_query_condition_cache = 0, use_skip_indexes = 0;

-- 64 rows in set. Elapsed: 0.248 sec. Processed 33.52 million rows, 212.40 MB (135.15 million rows/s., 856.44 MB/s.)
-- Peak memory usage: 29.67 MiB.
-- Сравнение: без skip-индексов читается почти вся таблица (33.52 млн строк
-- вместо 1.10 млн с индексами) — почти в 31 раз больше данных, запрос
-- примерно в 4.8 раза медленнее.
```

---

## 4. Использование таблиц - не страшный JOIN и словари

ClickHouse постоянно развивается в направлении повышения производительности JOIN,
и бытовавшее ранее мнение, что эта система не для них, давно неактуально. Чаще
всего критичное падение производительности происходит из-за грубых ошибок в
структуре соединений и из-за непропорциональных ресурсам объёмов объединяемых
данных.

### 4.1 Справочник и «внешний ключ»

```sql
-- Небольшая таблица: category_id → семантика
CREATE TABLE merchant_category_dict_source
(
    category_id   UInt32,
    category_name LowCardinality(String), -- совпадает с merchant_category в основной таблице
    risk_tier     Enum8(
                      'low'    = 1,
                      'medium' = 2,
                      'high'   = 3
                  ),
    is_regulated  UInt8                   -- 1 = регулируемая отрасль
)
ENGINE = MergeTree
ORDER BY category_id;

INSERT INTO merchant_category_dict_source VALUES
    (1,  'Retail',             'low',    0),
    (2,  'Food & Beverage',    'low',    0),
    (3,  'Travel',             'medium', 0),
    (4,  'Healthcare',         'high',   1),
    (5,  'Entertainment',      'low',    0),
    (6,  'Utilities',          'medium', 1),
    (7,  'E-commerce',         'medium', 0),
    (8,  'Finance',            'high',   1),
    (9,  'Education',          'low',    1),
    (10, 'Automotive',         'medium', 0),
    (11, 'Insurance',          'high',   1),
    (12, 'Real Estate',        'high',   1),
    (13, 'Telecommunications', 'medium', 1),
    (14, 'Gaming',             'medium', 0),
    (15, 'Gambling',           'high',   1),
    (16, 'Pharmaceuticals',    'high',   1),
    (17, 'Logistics',          'medium', 0),
    (18, 'Agriculture',        'low',    0),
    (19, 'Media',              'low',    0),
    (20, 'Government',         'medium', 1);
```

```sql
-- Добавляем "внешний ключ" в основную таблицу
-- Колонка с маппингом merchant_category → category_id
ALTER TABLE financial_transactions
    ADD COLUMN category_id UInt32 DEFAULT
        toUInt32(indexOf(
            ['Retail', 'Food & Beverage', 'Travel', 'Healthcare',
             'Entertainment', 'Utilities', 'E-commerce', 'Finance',
             'Education', 'Automotive', 'Insurance', 'Real Estate',
             'Telecommunications', 'Gaming', 'Gambling',
             'Pharmaceuticals', 'Logistics', 'Agriculture', 'Media', 'Government'],
            merchant_category
        ));

-- Материализуем для существующих строк
ALTER TABLE financial_transactions
    MATERIALIZE COLUMN category_id;

-- Проверяем что маппинг корректный
SELECT
    merchant_category,
    category_id,
    count() AS cnt
FROM financial_transactions
GROUP BY merchant_category, category_id
ORDER BY category_id;

-- 10 rows in set. Elapsed: 1.367 sec. Processed 50.00 million rows, 50.01 MB (36.58 million rows/s., 36.59 MB/s.)
-- Peak memory usage: 34.72 MiB.
```

```sql
-- Bloom filter на category_id
--
-- ВАЖНО: эффективность bloom filter напрямую зависит от корреляции
-- индексируемой колонки с ORDER BY таблицы.
--
-- merchant_category — первый ключ ORDER BY, а category_id является
-- её прямым производным (маппинг 1:1). Данные физически сгруппированы
-- на диске по category_id — все строки с category_id = 8 (Finance)
-- лежат рядом в небольшом числе гранул.
--
-- Bloom filter в такой ситуации максимально эффективен: он быстро
-- отсекает гранулы где нужного category_id заведомо нет.
--
-- Если бы category_id был случайным (без корреляции с ORDER BY) —
-- значения были бы размазаны по всем гранулам и bloom filter
-- почти не давал бы выигрыша.
ALTER TABLE financial_transactions
    ADD INDEX category_id_bf category_id
    TYPE bloom_filter(0.01) GRANULARITY 1;

ALTER TABLE financial_transactions
    MATERIALIZE INDEX category_id_bf;

-- Проверяем что индекс материализован
SELECT
    name,
    formatReadableSize(data_compressed_bytes)   AS compressed,
    formatReadableSize(data_uncompressed_bytes) AS uncompressed,
    marks_bytes
FROM system.data_skipping_indices
WHERE table = 'financial_transactions'
  AND name = 'category_id_bf';

-- 1 rows in set. Elapsed: 0.004 sec. Processed 16.00 rows, 1.20 KB (4.00 thousand rows/s., 299.75 KB/s.)
-- Peak memory usage: 4.02 MiB.
```

### 4.2 Словарь — более эффективный метод присоединения семантики

```sql
CREATE DICTIONARY merchant_category_dict
(
    category_id   UInt32,
    category_name String,
    risk_tier     String,
    is_regulated  UInt8
)
PRIMARY KEY category_id
SOURCE(CLICKHOUSE(
    TABLE 'merchant_category_dict_source'
    DB currentDatabase()
))
LAYOUT(FLAT())          -- FLAT — самый быстрый layout, ключи UInt64, таблица маленькая
LIFETIME(MIN 300 MAX 900);
```

### 4.3 JOIN против dictGet

```sql
-- ============================================================
-- Вариант A: обычный JOIN
-- ClickHouse строит хэш-таблицу из merchant_category_dict_source,
-- затем проверяет каждую строку financial_transactions.
-- Фильтр по risk_tier применяется ПОСЛЕ джойна.
--
-- Начиная с версии 25.10 ClickHouse автоматически строит runtime bloom filter
-- из правой части JOIN (merchant_category_dict_source) и передаёт его
-- в сканирование левой части (financial_transactions) ещё до джойна.
-- Это позволяет отсечь строки financial_transactions у которых category_id
-- заведомо не встречается в правой части — то есть частично имитирует
-- поведение которое раньше было доступно только через dictGet.
-- Тем не менее dictGet остаётся быстрее: словарь уже в памяти,
-- хэш-таблица не строится заново при каждом запросе, и статический
-- bloom filter на category_id работает на уровне гранул до чтения данных.
-- ============================================================
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount,
    avg(ft.risk_score)   AS avg_risk
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
ORDER BY total_amount DESC;

-- 6 rows in set. Elapsed: 1.256 sec. Processed 50.00 million rows, 650.02 MB (39.81 million rows/s., 517.53 MB/s.)
-- Peak memory usage: 55.57 MiB.
```

```sql
-- ============================================================
-- Вариант B: dictGet — Direct Join
-- ClickHouse делает lookup прямо в памяти для каждой строки
-- пережившей фильтр по PRIMARY KEY.
-- Фильтр по risk_tier вычисляется inline через dictGet,
-- bloom filter на category_id отсекает ненужные гранулы.
-- Сравните количество обработанных строк
-- ============================================================
SELECT
    merchant_category,
    country,
    count()            AS tx_count,
    sum(amount)        AS total_amount,
    avg(risk_score)    AS avg_risk
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
ORDER BY total_amount DESC;

-- 6 rows in set. Elapsed: 0.247 sec. Processed 11.69 million rows, 198.68 MB (47.31 million rows/s., 804.36 MB/s.)
-- Peak memory usage: 51.05 MiB.
-- Сравнение с вариантом A (JOIN): 11.69 млн строк против 50.00 млн — bloom filter
-- на category_id отсекает лишние гранулы ещё до lookup, запрос быстрее
-- примерно в 5 раз (0.247 сек против 1.256 сек).
```

Почему `dictGet` быстрее JOIN в этом сценарии:

| | JOIN | dictGet |
|---|---|---|
| Построение хэш-таблицы | да, каждый раз | не нужно — словарь уже в RAM |
| Алгоритм | Hash Join | Direct Join |
| Фильтр по `risk_tier` | после джойна | inline, строка за строкой |
| Bloom filter на `category_id` | не используется | отсекает гранулы до lookup |
| Память | хэш-таблица правой части | только словарь (уже загружен) |

Документация:
<https://clickhouse.com/docs/concepts/features/dictionaries#speeding-up-joins-using-a-dictionary>

### 4.4 Практика: замеры

При сравнении производительности для более точных результатов можно сбрасывать
кеши перед очередной итерацией:

```sql
-- Сбросить mark cache (индексные метки MergeTree)
SYSTEM DROP MARK CACHE;

-- Сбросить uncompressed cache (несжатые данные)
SYSTEM DROP UNCOMPRESSED CACHE;

-- Сбросить query cache (результаты запросов)
SYSTEM CLEAR QUERY CACHE;

-- Сбросить filesystem cache (кеш над S3/Azure/локальными дисками)
SYSTEM DROP FILESYSTEM CACHE;

-- Сбросить userspace page cache
SYSTEM DROP PAGE CACHE;
```

```sql
-- План для JOIN
EXPLAIN indexes = 1
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
ORDER BY total_amount DESC;

-- План для dictGet
EXPLAIN indexes = 1
SELECT
    merchant_category,
    country,
    count()          AS tx_count,
    sum(amount)      AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
ORDER BY total_amount DESC;
```

```sql
-- Замер времени
-- Baseline: JOIN без индексов
SELECT
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    count()              AS tx_count,
    sum(ft.amount)       AS total_amount
FROM financial_transactions AS ft
INNER JOIN merchant_category_dict_source AS mcd ON ft.category_id = mcd.category_id
WHERE mcd.risk_tier = 'high'
  AND ft.transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY ft.merchant_category, ft.country
FORMAT Null
SETTINGS use_query_condition_cache = 0;

-- 6 rows in set. Elapsed: 1.246 sec. Processed 50.00 million rows, 600.02 MB (40.13 million rows/s., 481.55 MB/s.)
-- Peak memory usage: 41.28 MiB.
-- (кэши не сбрасывались перед замером — SYSTEM DROP CACHE недоступен для
-- пользователя mcp_ro и является инстанс-wide операцией)

-- dictGet с bloom filter
SELECT
    merchant_category AS merchant_category,
    country           AS country,
    count()           AS tx_count,
    sum(amount)       AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
FORMAT Null
SETTINGS use_query_condition_cache = 0;

-- 6 rows in set. Elapsed: 0.219 sec. Processed 11.69 million rows, 186.99 MB (53.36 million rows/s., 853.85 MB/s.)
-- Peak memory usage: 44.40 MiB.

-- dictGet без bloom filter — для чистоты сравнения
SELECT
    merchant_category AS merchant_category,
    country           AS country,
    count()           AS tx_count,
    sum(amount)       AS total_amount
FROM financial_transactions
WHERE dictGet('merchant_category_dict', 'risk_tier', category_id) = 'high'
  AND transaction_date BETWEEN today() - INTERVAL 3 YEAR AND today()
GROUP BY merchant_category, country
FORMAT Null
SETTINGS use_query_condition_cache = 0, use_skip_indexes = 0;

-- 6 rows in set. Elapsed: 1.153 sec. Processed 50.00 million rows, 600.02 MB (43.37 million rows/s., 520.40 MB/s.)
-- Peak memory usage: 4.86 MiB.
-- Без skip-индексов dictGet читает все 50 млн строк (как и JOIN), а не 11.69 млн —
-- разница во времени против предыдущего замера (0.219 сек) подтверждает вклад
-- именно bloom filter на category_id, а не самого dictGet.
```

```sql
-- Проверка через query_log
SELECT
    query,
    projections,
    formatReadableQuantity(read_rows)  AS read_rows,
    formatReadableQuantity(read_bytes) AS read_bytes,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
  AND query_duration_ms > 0
ORDER BY initial_query_start_time DESC
LIMIT 6;

-- 6 rows in set. Elapsed: 0.015 sec. Processed 272.69 thousand rows, 4.25 MB (18.18 million rows/s., 283.08 MB/s.)
-- Peak memory usage: 4.94 MiB.
-- В колонке projections видно: JOIN-вариант читался через prj_order_by_risk,
-- а оба dictGet-варианта — напрямую из базовой таблицы ([] в projections).
-- Наличие подходящей проекции не гарантирует её использование именно там,
-- где ожидаешь, — ещё один повод для обязательной проверки из 2.6.
```

---

## 5. Использование таблиц - представления и оконные функции

Оконные функции выносятся на сторону БД, чтобы не передавать сырые данные в BI и
не считать их там: в DataLens встроенные оконные функции реализуются как агрегации
над данными, уже полученными из источника.

Переносить вычисления, в том числе оконные, в источник нужно и в том случае, когда
фильтрация в BI должна выполняться после расчётов. Если результаты оконных
вычислений должны использоваться как измерения, их также стоит переносить в
источник.

### 5.1 Представление

Представление позволяет реализовать запрос-витрину, который для внешних систем
вроде BI выглядит как обычная таблица, и перенести часть логики на сторону БД —
в том числе вычисления и маскирование полей.

Правило, которым нельзя пренебрегать: в представлениях всем полям задаются явные
алиасы, а таблицам — короткие псевдонимы; в запросах с несколькими таблицами это
обязательно. Без этого имена колонок на выходе легко превращаются в
`alias.field` или `table.field`, и весь датасет в BI ломается на ровном месте.
Полагаться на автоматический вывод имён нельзя: нет никаких гарантий, что завтра
в одной из таблиц не появится поле с таким же названием или что к представлению
не добавят ещё одну таблицу.

```sql
-- Представление с оконными функциями
CREATE VIEW financial_transactions_analytics AS
SELECT
    ft.transaction_id    AS transaction_id,
    ft.account_id        AS account_id,
    ft.merchant_category AS merchant_category,
    ft.country           AS country,
    ft.transaction_date  AS transaction_date,
    ft.amount            AS amount,
    ft.status            AS status,
    ft.risk_score        AS risk_score,

    -- Ранг транзакции по сумме внутри категории и страны за месяц
    -- Партиция: ~200 комбинаций (merchant_category × country × month)
    rank() OVER (
        PARTITION BY ft.merchant_category, ft.country,
                     dateTrunc('month', ft.transaction_date)
        ORDER BY ft.amount DESC
    ) AS amount_rank_in_segment,

    -- Доля суммы транзакции от общей суммы по категории и стране
    round(
        ft.amount / sum(ft.amount) OVER (
            PARTITION BY ft.merchant_category, ft.country
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ) * 100,
        2
    ) AS pct_of_segment_total,

    -- Скользящая сумма по аккаунту (нарастающий итог по дате)
    -- Партиция: ~1 000 000 аккаунтов — самая дорогая!
    sum(ft.amount) OVER (
        PARTITION BY ft.account_id
        ORDER BY ft.transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total,

    -- Предыдущая транзакция аккаунта (lag)
    -- 0, а не 0. — у amount тип Decimal(18, 2), с Float64-литералом нет общего супертипа
    lagInFrame(ft.amount, 1, 0) OVER (
        PARTITION BY ft.account_id
        ORDER BY ft.transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS prev_tx_amount,

    -- Отклонение от средней суммы по категории
    -- Партиция: ~20 категорий
    round(
        ft.amount - avg(ft.amount) OVER (
            PARTITION BY ft.merchant_category
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        ),
        2
    ) AS diff_from_category_avg,

    -- Перцентильный ранг по risk_score внутри страны
    -- Партиция: ~20 стран
    percent_rank() OVER (
        PARTITION BY ft.country
        ORDER BY ft.risk_score
    ) AS risk_percentile_in_country

FROM financial_transactions AS ft
WHERE ft.status = 'completed';
```

### 5.2 Стоимость окон

```sql
-- Запрос к представлению
-- Топ транзакций с высоким риском, которые одновременно
-- входят в топ-3 по сумме в своём сегменте
-- ВАЖНО: этот запрос выполняется медленнее предыдущих, потому что в нём
-- запрашиваются результаты сразу нескольких оконных функций.
--
-- Причина: оконные функции вычисляются для ВСЕХ строк удовлетворяющих
-- WHERE status = 'completed', и только после этого применяются фильтры
-- amount_rank_in_segment <= 3 и risk_percentile_in_country > 0.9.
--
-- Фильтр по transaction_date частично помогает — PRIMARY KEY таблицы
-- позволяет ClickHouse отсечь ненужные гранулы до вычисления окон,
-- но внутри отобранного диапазона все строки всё равно обрабатываются.
--
-- Это фундаментальное свойство оконных функций: они требуют полного
-- прохода по каждой партиции (PARTITION BY) прежде чем вернуть результат.
SELECT
    transaction_id,
    account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    risk_score,
    amount_rank_in_segment,
    pct_of_segment_total,
    risk_percentile_in_country,
    diff_from_category_avg
FROM financial_transactions_analytics
WHERE amount_rank_in_segment <= 3
  AND risk_percentile_in_country > 0.9
  AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
ORDER BY risk_score DESC, amount DESC
LIMIT 50;

-- 50 rows in set. Elapsed: 5.473 sec. Processed 50.00 million rows, 1.30 GB (9.14 million rows/s., 237.54 MB/s.)
-- Peak memory usage: 2.09 GiB.
-- Подтверждает предупреждение из текста выше: запрос на порядки дороже
-- single-window запросов из 5.3 — оконные функции считаются по всем строкам
-- status = 'completed' до применения фильтров amount_rank_in_segment
-- и risk_percentile_in_country.
```

```sql
-- Проверка плана выполнения
EXPLAIN
SELECT
    transaction_id,
    amount_rank_in_segment,
    risk_percentile_in_country
FROM financial_transactions_analytics
WHERE amount_rank_in_segment <= 3
  AND risk_percentile_in_country > 0.9
  AND transaction_date BETWEEN today() - INTERVAL 1 YEAR AND today()
ORDER BY risk_score DESC
LIMIT 50;
```

### 5.3 Запрос с одной оконной функцией

В BI целесообразно реализовывать не огромные, сложные для анализа таблицы, а
наборы графиков, в которых участвует лишь несколько измерений и показателей.
Если запрос включает лишь одно-два значения из оконной функции — ситуация с
производительностью значительно лучше, чем при обращении к представлению, где
вычисляются все оконные функции сразу.

```sql
-- Нарастающий итог по аккаунту
SELECT
    transaction_date,
    amount,
    sum(amount) OVER (
        PARTITION BY account_id
        ORDER BY transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total
FROM financial_transactions
WHERE account_id = 282208 -- сделайте выборку для получения реальных сгенеренных значений поля для фильтра
  AND status = 'completed'
ORDER BY transaction_date;

-- 57 rows in set. Elapsed: 0.064 sec. Processed 50.00 million rows, 252.42 MB (781.25 million rows/s., 3.94 GB/s.)
-- Peak memory usage: 6.13 MiB.
```

```sql
-- EXPLAIN показывает что PRIMARY KEY не отсекает гранулы по account_id —
-- он стоит четвёртым в ORDER BY после merchant_category, country и month.
-- Данные физически не сгруппированы по account_id, поэтому ClickHouse
-- вынужден читать большую часть таблицы.
-- Оконная функция при этом всё равно вычисляется только для строк
-- одного аккаунта — PARTITION BY account_id гарантирует это логически,
-- но не снижает объём физически прочитанных данных.
--
-- Если запросы по конкретному account_id критичны — стоит рассмотреть
-- вынесение account_id на первую позицию в ORDER BY, но это противоречит
-- текущей модели доступа ориентированной на аналитику по категориям и странам.
EXPLAIN indexes = 1
SELECT
    transaction_date,
    amount,
    sum(amount) OVER (
        PARTITION BY account_id
        ORDER BY transaction_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS account_running_total
FROM financial_transactions
WHERE account_id = 282208
  AND status = 'completed'
ORDER BY transaction_date;
```

### 5.4 Селекторы поверх представления

```sql
-- Проверяем использование проекции для запроса под селектор поверх представления —
-- список уникальных значений одного измерения через DISTINCT.
-- Это появилось совсем недавно и значительно ускоряет работу дашбордов
-- с обилием селекторов, построенных на представлениях.
--
-- ВАЖНО: колонку для DISTINCT нужно брать НЕ из префикса ORDER BY базовой
-- таблицы. merchant_category — первая колонка ORDER BY, поэтому чтение
-- напрямую из financial_transactions и так дёшево (это LowCardinality-
-- дедупликация без реального скана), и планировщику незачем переключаться
-- на проекцию — эффект будет незаметен. currency в ORDER BY не входит, но
-- есть в GROUP BY проекции prj_agg_by_category_country — вот на ней разница
-- видна.
EXPLAIN indexes = 1
SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency;

-- Сравниваем с отключённой проекцией чтобы увидеть разницу
SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency
SETTINGS optimize_use_projections = 0;

-- 7 rows in set. Elapsed: 0.040 sec. Processed 50.00 million rows, 50.01 MB (1.25 billion rows/s., 1.25 GB/s.)
-- Peak memory usage: 4.60 MiB.

SELECT DISTINCT currency
FROM financial_transactions
ORDER BY currency;

-- 7 rows in set. Elapsed: 0.013 sec. Processed 47.38 thousand rows, 57.19 KB (3.64 million rows/s., 4.40 MB/s.)
-- Peak memory usage: 4.61 MiB.
-- EXPLAIN indexes = 1 подтверждает: ReadFromMergeTree (prj_agg_by_category_country),
-- Granules: 116/6149 — читается только проекция, а не 50 млн строк базовой
-- таблицы. В 1000+ раз меньше прочитанных строк и в 3 раза быстрее по
-- времени, хотя запрос и так короткий.
```

```sql
-- Проверяем через query_log какая проекция была использована
SYSTEM FLUSH LOGS;

SELECT
    query,
    projections,
    formatReadableQuantity(read_rows) AS read_rows,
    query_duration_ms
FROM system.query_log
WHERE type = 'QueryFinish'
  AND arrayExists(t -> t LIKE '%financial_transactions%', tables)
  AND query LIKE '%DISTINCT currency%'
ORDER BY initial_query_start_time DESC
LIMIT 4;

-- 3 rows in set. Elapsed: 0.033 sec. Processed 275.27 thousand rows, 34.28 MB (8.34 million rows/s., 1.04 GB/s.)
-- Peak memory usage: 33.33 MiB.
-- В строке без SETTINGS optimize_use_projections = 0 поле projections
-- заполнено — test.financial_transactions.prj_agg_by_category_country —
-- то же самое, что видно и в EXPLAIN.
```

---

## 6. Производные таблицы - материализованные представления

Для реализации классического подхода с таблицами-агрегатами в Clickhouse можно 
использовать материализованные представлены и обновляемые материализованные представления.

### 6.1 Инкрементальный MV — триггер на вставку

Обычный материализованный view в ClickHouse — *это триггер на вставку*. Он
срабатывает при каждом `INSERT` в исходную таблицу и вычисляет агрегаты в момент
вставки.

Ключевой момент: агрегаты вычисляются над вставляемым блоком, а не над всей
таблицей. Доагрегация выполняется механизмами SummingMergeTree и
AggregatingMergeTree на фоновых мержах.

Отдельное правило, нарушение которого ловится не сразу: колонки MV сопоставляются
с колонками целевой таблицы по именам, а не по позициям.

Осторожно: не рекомендуется использовать в качестве источника
ReplacingMergeTree и CollapsingMergeTree — MV видит строки до фоновой
дедупликации и схлопывания, что может привести к некорректным агрегатам.
Аналогично, при удалении или изменении данных в обычной MergeTree перенос
изменений в целевые таблицы не произойдёт.

Внимание: если в запросе представления используется несколько таблиц, то триггер
будет срабатывать при вставке в первую из них и в общем случае не рекомендуется
строить представление на нескольких таблицах, а при необходимости обогащения данных 
следует использовать словари `DICTIONARY`.

```sql
-- ------------------------------------------------------------
-- MV 1: Агрегация по merchant_category + currency за день
-- Ускоряет: аналитику оборота по категориям мерчантов
-- Движок: SummingMergeTree — суммирует числовые колонки
-- при совпадении ключа сортировки
-- ------------------------------------------------------------
-- Целевая таблица
CREATE TABLE financial_transactions_daily_summary
(
    transaction_date  Date,
    merchant_category LowCardinality(String),
    currency          LowCardinality(String),
    total_amount      Decimal(18, 2),
    total_fee         Decimal(10, 4),
    tx_count          UInt64
)
ENGINE = SummingMergeTree
ORDER BY (merchant_category, currency, transaction_date);

-- Материализованное представление
CREATE MATERIALIZED VIEW financial_transactions_daily_summary_mv
TO financial_transactions_daily_summary
AS
SELECT
    transaction_date,
    merchant_category,
    currency,
    sum(amount)  AS total_amount,
    sum(fee)     AS total_fee,
    count()      AS tx_count
FROM financial_transactions
GROUP BY transaction_date, merchant_category, currency;
```

```sql
-- Тестовая вставка — проверяем что MV срабатывает
-- и данные попадают в целевую таблицу
-- ПРИМЕЧАНИЕ: transaction_id взяты 50000001..50000005, а не 1..5 — значения
-- 1..5 уже заняты сквозной генерацией из 1.6 (numbers(50000000)).
INSERT INTO financial_transactions
    (transaction_id, account_id, merchant_category, payment_method,
     country, currency, transaction_date, status,
     amount, fee, exchange_rate, cashback_amount, risk_score)
VALUES
    (50000001, 1001, 'Retail',          'card',     'US', 'USD', today(), 'completed', 150.00, 1.5000, 1.0, 0.75, 10),
    (50000002, 1002, 'Retail',          'transfer', 'DE', 'EUR', today(), 'completed', 200.00, 2.0000, 1.1, 1.00, 20),
    (50000003, 1003, 'Food & Beverage', 'card',     'US', 'USD', today(), 'pending',   75.50,  0.7500, 1.0, 0.38, 5),
    (50000004, 1004, 'Food & Beverage', 'cash',     'FR', 'EUR', today(), 'completed', 50.00,  0.5000, 1.1, 0.25, 15),
    (50000005, 1005, 'Retail',          'crypto',   'US', 'USD', today(), 'failed',    500.00, 5.0000, 1.0, 0.00, 95);

-- Проверяем что данные попали в целевую таблицу
-- FINAL нужен для SummingMergeTree — схлопывает несмёрженные части
SELECT
    merchant_category,
    currency,
    sum(total_amount) AS total_amount,
    sum(tx_count)     AS tx_count
FROM financial_transactions_daily_summary
FINAL
WHERE transaction_date = today()
GROUP BY merchant_category, currency
ORDER BY total_amount DESC;

-- 4 rows in set. Elapsed: 0.004 sec. Processed 4.00 rows, 155.00 B (1.00 thousand rows/s., 38.75 KB/s.)
-- Peak memory usage: 4.23 MiB.
```

```sql
-- Backfill исторических данных
-- (если таблица уже содержала данные до создания MV)
INSERT INTO financial_transactions_daily_summary
SELECT
    transaction_date,
    merchant_category,
    currency,
    sum(amount)  AS total_amount,
    sum(fee)     AS total_fee,
    count()      AS tx_count
FROM financial_transactions
GROUP BY transaction_date, merchant_category, currency;
```

### 6.2 AggregatingMergeTree и состояния агрегатов

Там, где нужны не суммы, а средние, уникальные и прочие агрегаты, целевая
таблица хранит не значения, а частичные состояния: колонки типа
`AggregateFunction(...)`, заполняемые функциями с суффиксом `-State` и читаемые
функциями с суффиксом `-Merge`.

```sql
-- ------------------------------------------------------------
-- MV 2: Фрод-мониторинг — агрегация по risk_score bucket + country
-- Ускоряет: анализ распределения рисков по странам
-- Движок: AggregatingMergeTree — хранит partial aggregate states
-- ------------------------------------------------------------
CREATE TABLE financial_transactions_risk_by_country
(
    country       LowCardinality(String),
    risk_bucket   UInt8,          -- 0=low(0-33), 1=medium(34-66), 2=high(67-100)
    tx_count      AggregateFunction(count),
    avg_amount    AggregateFunction(avg, Decimal(18, 2)),
    avg_risk      AggregateFunction(avg, UInt8)
)
ENGINE = AggregatingMergeTree
ORDER BY (country, risk_bucket);

CREATE MATERIALIZED VIEW financial_transactions_risk_by_country_mv
TO financial_transactions_risk_by_country
AS
SELECT
    country,
    toUInt8(intDiv(risk_score, 34)) AS risk_bucket,
    countState()                    AS tx_count,
    avgState(amount)                AS avg_amount,
    avgState(risk_score)            AS avg_risk
FROM financial_transactions
GROUP BY country, risk_bucket;

-- Backfill существующих данных
INSERT INTO financial_transactions_risk_by_country
SELECT
    country,
    toUInt8(intDiv(risk_score, 34)) AS risk_bucket,
    countState()                    AS tx_count,
    avgState(amount)                AS avg_amount,
    avgState(risk_score)            AS avg_risk
FROM financial_transactions
GROUP BY country, risk_bucket;

-- Пример запроса
SELECT
    country,
    risk_bucket,
    countMerge(tx_count) AS tx_count,
    avgMerge(avg_amount) AS avg_amount,
    avgMerge(avg_risk)   AS avg_risk
FROM financial_transactions_risk_by_country
GROUP BY country, risk_bucket
ORDER BY country, risk_bucket;

-- 45 rows in set. Elapsed: 0.004 sec. Processed 45.00 rows, 3.49 KB (11.25 thousand rows/s., 872.00 KB/s.)
-- Peak memory usage: 237.45 KiB.
```

### 6.3 Refreshable MV: REPLACE и APPEND

Refreshable MV пересчитывается по расписанию, а не по вставке. Подходит для
отчётов, где допустима небольшая задержка. На `INSERT` он не реагирует вообще.

Поддерживаются два режима записи:

- **REPLACE** (по умолчанию) — каждый refresh атомарно перезаписывает всё
  содержимое целевой таблицы, старые данные полностью заменяются новыми.
  Подходит для текущих состояний, топов, lookup-таблиц.
- **APPEND** — каждый refresh добавляет строки в конец таблицы, не удаляя
  старые. Вставка не атомарна, это аналог обычного `INSERT INTO ... SELECT`.
  Подходит для накопления снапшотов, исторических срезов, периодических отчётов.

| | REPLACE (по умолчанию) | APPEND |
|---|---|---|
| Старые данные | удаляются | сохраняются |
| Атомарность | атомарна | не атомарна |
| Подходит для | текущее состояние, топы | снапшоты, история, динамика |

Запрос всегда адресуется целевой таблице, а не самому MV.

```sql
-- ------------------------------------------------------------
-- Refreshable MV 1: Ежедневный отчёт по статусам транзакций
-- Режим: REPLACE (по умолчанию) — атомарная полная перезапись
-- Обновляется каждый день в 03:00 UTC
-- Пересчитывает ПОЛНОСТЬЮ — подходит для актуального состояния
-- ВНИМАНИЕ: не реагирует на INSERT, только на расписание!
-- ------------------------------------------------------------
-- Целевая таблица (явная, рекомендуется всегда)
CREATE TABLE financial_transactions_status_report
(
    transaction_date  Date,
    status            Enum8('completed'=1,'pending'=2,'failed'=3,'refunded'=4),
    merchant_category LowCardinality(String),
    country           LowCardinality(String),
    tx_count          UInt64,
    total_amount      Decimal(18, 2),
    avg_amount        Float64,
    total_fee         Decimal(10, 4)
)
ENGINE = MergeTree
ORDER BY (transaction_date, status, merchant_category);

-- Refreshable MV через TO — только определение запроса и расписание
CREATE MATERIALIZED VIEW financial_transactions_status_report_mv
REFRESH EVERY 1 DAY OFFSET 3 HOUR
TO financial_transactions_status_report
AS
SELECT
    transaction_date,
    status,
    merchant_category,
    country,
    count()      AS tx_count,
    sum(amount)  AS total_amount,
    avg(amount)  AS avg_amount,
    sum(fee)     AS total_fee
FROM financial_transactions
WHERE transaction_date >= today() - INTERVAL 90 DAY
GROUP BY transaction_date, status, merchant_category, country;

-- Принудительный refresh для проверки (не ждём расписания)
SYSTEM REFRESH VIEW financial_transactions_status_report_mv;
-- ПРИМЕЧАНИЕ: SYSTEM REFRESH VIEW требует отдельный грант SYSTEM VIEWS,
-- которого у read-only пользователя может не быть. Проверять это не
-- обязательно: Refreshable MV без EMPTY выполняет первый refresh сразу
-- при создании, поэтому данные в целевой таблице уже есть.

-- Запрос идёт к ЦЕЛЕВОЙ таблице, не к MV!
SELECT
    status,
    merchant_category,
    sum(tx_count)     AS tx_count,
    sum(total_amount) AS total_amount
FROM financial_transactions_status_report  -- <- целевая таблица
WHERE transaction_date = today()
GROUP BY status, merchant_category
ORDER BY total_amount DESC;

-- 4 rows in set. Elapsed: 0.004 sec. Processed 361.00 rows, 7.40 KB (90.25 thousand rows/s., 1.85 MB/s.)
-- Peak memory usage: 4.23 MiB.
```

```sql
-- ------------------------------------------------------------
-- Refreshable MV 2: Исторические снапшоты риска по странам
-- Режим: APPEND — каждый refresh ДОБАВЛЯЕТ новый срез, не удаляя старые
-- Обновляется каждый час
-- Позволяет отслеживать динамику показателей во времени
-- ВНИМАНИЕ: вставка НЕ атомарна — аналог обычного INSERT INTO ... SELECT
-- ВНИМАНИЕ: таблица растёт неограниченно — нужен TTL или партиционирование!
-- ------------------------------------------------------------
-- Целевая таблица (явная, рекомендуется всегда)
CREATE TABLE financial_transactions_risk_snapshots
(
    snapshot_ts   DateTime,
    country       LowCardinality(String),
    tx_count      UInt64,
    total_amount  Decimal(18, 2),
    avg_risk      Float64,
    high_risk_cnt UInt64
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(snapshot_ts)
ORDER BY (snapshot_ts, country)
TTL snapshot_ts + INTERVAL 1 YEAR;

-- Refreshable MV с APPEND через TO
CREATE MATERIALIZED VIEW financial_transactions_risk_snapshots_mv
REFRESH EVERY 1 HOUR
APPEND TO financial_transactions_risk_snapshots  -- <- APPEND + явная целевая таблица
AS
SELECT
    now()                    AS snapshot_ts,
    country,
    count()                  AS tx_count,
    sum(amount)              AS total_amount,
    avg(risk_score)          AS avg_risk,
    countIf(risk_score > 66) AS high_risk_cnt
FROM financial_transactions
WHERE transaction_date >= today() - INTERVAL 7 DAY
GROUP BY country;

-- Принудительный refresh для проверки
SYSTEM REFRESH VIEW financial_transactions_risk_snapshots_mv;
-- ПРИМЕЧАНИЕ: как и в 6.3 REPLACE-примере, первый refresh уже произошёл
-- автоматически при создании MV, отдельный грант SYSTEM VIEWS не понадобился.

-- Запрос идёт к ЦЕЛЕВОЙ таблице, не к MV!
SELECT
    snapshot_ts,
    country,
    avg_risk,
    high_risk_cnt
FROM financial_transactions_risk_snapshots  -- <- целевая таблица
WHERE snapshot_ts >= now() - INTERVAL 24 HOUR
ORDER BY snapshot_ts ASC, country;

-- 9 rows in set. Elapsed: 0.005 sec. Processed 9.00 rows, 287.00 B (1.80 thousand rows/s., 57.40 KB/s.)
-- Peak memory usage: 4.20 MiB.
-- 9, а не 15 стран — потому что в срез transaction_date >= today() - INTERVAL 7 DAY
-- попадают только страны, для которых в сквозной генерации 1.6
-- (today() - 1095 + rand() % 1095, максимум — «вчера») или в тестовой
-- вставке из 6.1 (today(), страны US/DE/FR) нашлись строки за последнюю неделю.
```

### 6.4 Что выбирать

| | Инкрементальный MV | Refreshable MV |
|---|---|---|
| Обновление | при каждом INSERT | по расписанию |
| Задержка данных | реальное время | до следующего refresh |
| JOIN между таблицами | ограниченно | полная поддержка |
| LIMIT и сложные подзапросы | нет | да |
| Петабайтные таблицы | да, инкрементально | полный пересчёт |
| Подходит для | агрегации, трансформации | отчёты, топы, сложная аналитика |

---

## 7. Мониторинг BI-нагрузки

Ключевой момент для BI-нагрузки: смотрите на `total_cpu_sec` (сумма времени всех
выполнений одного типа запроса), а не на `max_ms` отдельного запроса — именно
суммарная нагрузка показывает, что реально нагружает систему, когда BI присылает
пачки запросов.

```sql
-- Текущие активные запросы прямо сейчас
-- Что выполняется в данный момент, сгруппировано по типу нагрузки
SELECT
    user,
    count()                                AS query_count,
    sum(elapsed)                           AS total_elapsed_sec,
    max(elapsed)                           AS max_elapsed_sec,
    formatReadableSize(sum(memory_usage))  AS total_memory,
    formatReadableSize(max(memory_usage))  AS max_memory,
    formatReadableQuantity(sum(read_rows)) AS total_read_rows,
    groupArray(left(query, 80))            AS sample_queries
FROM system.processes
GROUP BY user
ORDER BY total_elapsed_sec DESC;

-- 1 rows in set. Elapsed: 0.004 sec. Processed 1.00 rows, 3.38 KB (250.00 rows/s., 843.75 KB/s.)
-- Peak memory usage: 4.02 MiB.
-- На момент прогона в system.processes виден только сам этот запрос —
-- других активных сессий на инстансе не было.
```

```sql
-- Куда упирается система: CPU vs IO vs Memory
-- Агрегированная статистика за последние N минут
-- Показывает узкие места по типам запросов
SELECT
    toStartOfMinute(query_start_time)              AS minute,
    count()                                        AS total_queries,
    countIf(query_duration_ms > 1000)              AS slow_queries,
    countIf(hasAny(tables, ['test.financial_transactions'])) AS ft_queries,

    -- CPU
    formatReadableQuantity(sum(read_rows))         AS rows_read,
    round(avg(query_duration_ms))                  AS avg_duration_ms,
    max(query_duration_ms)                         AS max_duration_ms,

    -- IO
    formatReadableSize(sum(read_bytes))            AS bytes_read,
    formatReadableSize(sum(written_bytes))         AS bytes_written,

    -- Memory
    formatReadableSize(max(memory_usage))          AS peak_memory,
    formatReadableSize(sum(memory_usage))          AS total_memory,

    -- Параллелизм: сколько запросов шло одновременно
    max(ProfileEvents['ConcurrentQueriesCount'])   AS max_concurrent
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 30 MINUTE
  AND query_start_time < now()
GROUP BY minute
ORDER BY minute DESC;

-- 31 rows in set. Elapsed: 0.016 sec. Processed 272.38 thousand rows, 1.48 MB (17.02 million rows/s., 92.20 MB/s.)
-- Peak memory usage: 4.49 MiB.
-- hasAny() ищет точное совпадение строки, а в system.query_log таблица
-- записана с префиксом БД — используйте полное имя 'test.financial_transactions'
-- (или свою схему), иначе ft_queries всегда будет 0.
```

```sql
-- Топ тяжёлых запросов за период
-- Находим запросы которые суммарно съедают больше всего ресурсов
-- normalized_query_hash группирует похожие запросы (с разными параметрами)
SELECT
    normalized_query_hash,
    count()                                AS executions,
    any(left(query, 120))                  AS sample_query,
    round(avg(query_duration_ms))          AS avg_ms,
    max(query_duration_ms)                 AS max_ms,
    round(sum(query_duration_ms) / 1000)   AS total_cpu_sec,
    formatReadableSize(avg(memory_usage))  AS avg_memory,
    formatReadableSize(max(memory_usage))  AS max_memory,
    formatReadableQuantity(avg(read_rows)) AS avg_rows_read,
    formatReadableSize(avg(read_bytes))    AS avg_bytes_read,
    countIf(exception != '')               AS error_count
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 1 HOUR
  AND user != 'system'
GROUP BY normalized_query_hash
ORDER BY total_cpu_sec DESC
LIMIT 20;

-- 20 rows in set. Elapsed: 0.020 sec. Processed 271.90 thousand rows, 3.20 MB (13.60 million rows/s., 160.11 MB/s.)
-- Peak memory usage: 5.31 MiB.
-- На реальном шэренном инстансе Managed Service в топ-20 по total_cpu_sec
-- попадают и запросы других пользователей кластера — сама эта метрика
-- (сумма времени всех выполнений) для того и нужна, чтобы такие запросы
-- не терялись за max_ms одного запроса. Чужой SQL из вывода в кукбук
-- не копирую, из относящихся к этому кукбуку строк, например:
--   avg_ms=5473, max_memory=2.09 GiB, avg_rows_read=50.00 million — запрос из 5.2
--     (view + несколько оконных функций)
--   avg_ms=1153, avg_rows_read=50.00 million — dictGet без skip-индексов из 4.4
--   avg_ms=804,  avg_rows_read=50.00 million — backfill INSERT в 6.1
```

```sql
-- Конкурентность: пиковая нагрузка по временным слотам
-- Видим пики когда BI присылает пачки запросов одновременно
SELECT
    toStartOfMinute(query_start_time)                   AS minute,
    count()                                             AS queries_started,
    countIf(hasAny(tables, ['test.financial_transactions'])) AS on_main_table,
    -- Запросы с подзапросами обычно длиннее и тяжелее
    countIf(query_duration_ms > 2000)                   AS heavy_queries,
    countIf(query_duration_ms <= 500)                   AS fast_queries,
    formatReadableSize(sum(read_bytes))                 AS total_io
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_start_time >= now() - INTERVAL 2 HOUR
  AND user != 'system'
GROUP BY minute
ORDER BY minute DESC;

-- 121 rows in set. Elapsed: 0.013 sec. Processed 272.42 thousand rows, 1.84 MB (20.96 million rows/s., 141.25 MB/s.)
-- Peak memory usage: 4.41 MiB.
-- Полные 120+ строк в файл не переношу — по минутам видна фоновая активность
-- на инстансе за пределами этого кукбука (устойчивые 7-60 запросов/мин ещё
-- до начала прогона примеров, т.к. это общий шэренный managed-инстанс).
```

```sql
-- Ожидания на мержах и вставках (конкуренция за ресурсы)
-- Если BI-запросы конкурируют с фоновыми мержами
SELECT
    event_time,
    metric,
    value
FROM system.metric_log
ARRAY JOIN
    ['BackgroundMergesAndMutationsPoolTask',
     'BackgroundMergesAndMutationsPoolSize',
     'MemoryTracking',
     'Query',
     'MergeTreeDataSelectExecutorThreads'] AS metric,
    [CurrentMetric_BackgroundMergesAndMutationsPoolTask,
     CurrentMetric_BackgroundMergesAndMutationsPoolSize,
     CurrentMetric_MemoryTracking,
     CurrentMetric_Query,
     CurrentMetric_MergeTreeDataSelectExecutorThreads] AS value
WHERE event_time >= now() - INTERVAL 10 MINUTE
ORDER BY event_time DESC, metric ASC
LIMIT 10;
```

---

## 8. Использование таблиц и витрин в Datalens

> Раздел написан по документации DataLens (<https://yandex.cloud/ru/docs/datalens/>).
> В отличие от остальных разделов кукбука шаги в интерфейсе DataLens (создание
> подключения, датасета, вычисляемого поля, чарта, включение параметризации)
> руками не прогонялись — терминология и порядок экранов взяты из документации
> и их стоит перепроверить в актуальном интерфейсе. SQL-часть 8.2
> (представление и его вызов с параметрами) реально выполнена на базе `test`
> — результат см. ниже.

В качестве сквозного примера возьмём не саму таблицу `financial_transactions`,
а представление `financial_transactions_analytics` из 5.1 — то, в котором уже
посчитаны оконные функции (`amount_rank_in_segment`, `risk_percentile_in_country`
и т.д.). Идея та же, что и в разделе 5: BI получает уже готовые колонки и не
должен ничего досчитывать сам.

### 8.1 Практика: датасет и чарт на представлении с оконными функциями

**1. Подключение.** Если подключение к этому ClickHouse в DataLens ещё не
создано — **Подключения → Создать подключение → ClickHouse®**. Для
Managed Service указываются хост, порт HTTP-интерфейса, пользователь и
пароль; для BI имеет смысл заводить отдельного пользователя с доступом
только на чтение (см. 0.1). Уровень доступа SQL-запросов — «SQL на чтение».
После проверки соединения — **Создать подключение**.

**2. Создание датасета.** На странице подключения — **Создать датасет**.
В списке таблиц источника выбираем не `financial_transactions`, а
`financial_transactions_analytics` — представления ClickHouse видны в этом
списке наравне с обычными таблицами, отдельно выбирать «режим SQL» не
требуется. Перетаскиваем таблицу на рабочую область, переходим на вкладку
**Поля**, сохраняем датасет с названием.

**3. Вычисляемое поле.** На вкладке **Поля** — **Добавить поле**. Пример,
завязанный на уже посчитанные в представлении оконные функции — метка
риск-сегмента по перцентилю `risk_percentile_in_country`:

```
IF([risk_percentile_in_country] > 0.9, 'Высокий риск (топ 10%)',
   IF([risk_percentile_in_country] > 0.5, 'Средний риск', 'Низкий риск'))
```

Называем поле, например, `Риск-сегмент`, и создаём.

**4. Чарт.** На странице датасета — **Создать чарт**. Раскладка для
примера: `merchant_category` — в **Измерения** (по оси X), `Риск-сегмент` —
в **Измерения** как разбивку (легенда/цвет), количество транзакций
(`count()` по `transaction_id`) — в **Показатели** (ось Y). DataLens при
этом просто группирует и считает `count()` — сам перцентиль он не считает
ни разу, вся тяжёлая часть уже отработала в ClickHouse при обращении к
представлению.

### 8.2 Задел: параметры датасета и параметризированные представления в БД

DataLens умеет передавать значения параметров датасета в SQL-подзапрос
источника через плейсхолдер `{{имя_параметра}}` — но, по документации, это
работает только когда датасет описан произвольным SQL-запросом, а не
выбором таблицы из списка (как в 8.1). Порядок в общих чертах:

1. В датасете включить параметризацию, на вкладке **Параметры** добавить
   параметр (название, тип, значение по умолчанию), включить для него
   **Разрешить использовать в настройке источника**.
2. В SQL-запросе источника подставить `{{имя_параметра}}` — например,
   в `WHERE` или, как в этом кукбуке, в качестве значения параметра
   параметризированного представления ClickHouse (в самом ClickHouse это
   отдельная сущность: `CREATE VIEW ... AS SELECT ... WHERE col = {p:Type}`,
   вызывается как `view_name(p = значение)`).

Вариант такого представления над `financial_transactions_analytics` — вывод
топа сегмента по стране с фильтром по минимальному риск-перцентилю,
задаваемому параметром:

```sql
CREATE VIEW financial_transactions_top_by_country AS
SELECT
    transaction_id,
    account_id,
    merchant_category,
    country,
    transaction_date,
    amount,
    risk_score,
    amount_rank_in_segment,
    risk_percentile_in_country
FROM financial_transactions_analytics
WHERE country = {country:String}
  AND risk_percentile_in_country >= {min_risk_percentile:Float64}
ORDER BY amount_rank_in_segment;
```

Проверка из `clickhouse-client`:

```sql
SELECT count()
FROM financial_transactions_top_by_country(country = 'US', min_risk_percentile = 0.9);

-- 1 rows in set. Elapsed: 3.662 sec. Processed 50.00 million rows, 700.02 MB (13.65 million rows/s., 191.16 MB/s.)
-- Peak memory usage: 545.29 MiB.
```

ВАЖНО: представление реально проверено на базе `test`, и даже с фильтром по
`country` и `min_risk_percentile` ClickHouse читает и считает оконные функции
по ВСЕЙ `financial_transactions_analytics` (50 млн строк, 3.7 сек) — ровно то
предупреждение из 5.2: `WHERE` поверх представления с окнами применяется уже
после того, как окна посчитаны по всей партиции. Для параметров, задающих
узкий срез, разумнее не накручивать параметризацию поверх такого
представления как есть, а выносить фильтр по параметру внутрь, до
`PARTITION BY` — например, отдельным CTE/подзапросом по `financial_transactions`
с `WHERE country = {country:String}`, и уже над ним считать оконные функции.

В датасете DataLens источник тогда выглядит так (кавычки для `country`
нужны явно — `{{...}}` в DataLens это текстовая подстановка, а не
параметр с типом):

```sql
SELECT *
FROM financial_transactions_top_by_country(
    country = '{{country_param}}',
    min_risk_percentile = {{min_risk_percentile_param}}
);
```

Это именно задел: сама схема рабочая, но конкретные шаги мастера датасета
для SQL-источника, валидацию параметров (регулярные выражения, о которых
пишет документация DataLens) и поведение при пустом/некорректном значении
параметра нужно проверять в интерфейсе отдельно, прежде чем выносить в
продакшн-дашборд.
