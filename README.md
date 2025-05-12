# tryBI
небольшое сравнение DataLens и Apache superset

[Создание таблиц и наполнение их данными при помощи airflow](#first)
2) Создание диаграмм в DataLens
3) Создание диаграмм в Apache superset

## <a id="first">Создание таблиц и наполнение их данными при помощи airflow</a>
Данные, на основании которых будут построены диаграммы имеют примерно следующий вид (пример протокола любительского старта https://5verst.ru/tyumenparkgagarina/results/12.04.2025/)
|Город |Локация           |id Участника|Дата забега|Группв     |Результат (секунд)|
|------|------------------|------------|-----------|-----------|------------------|
|Тюмень|zatyumensky       |790145311   | 16.09.2023|Ж30-34     | 1681             |
|Тюмень|tyumenparkgagarina|790145311   | 12.04.2025|Ж30-34     | 1958             |

Парсить ФИО участников не планируется - так как это противоречит закону о защите персональных данных.
Можно было бы превести данные к 3НФ, выделив сущности:
* Город (id_города, наименование)
* Локация (id_локации, наименование, id_города)
* Протокол (id_протокола, id_локации, дата_забега, группа, id_участника, результат)
Причечание: Группа относится к сущности протокол, а не к сущности участник, так как возраст участника меняется от забега к забегу

Но в рамках данной задачи я не вижу в этом смысла (просто сохраню все в одну таблицу, тем более ошибок из-за ручного ввода не случится)
Поэтому я создам только одну таблицу и на основании нее буду строить диаграммы
|Что такое   |Название поля в БД|Тип данных|Дополнительно|
|------------|------------------|----------|-------------------|
|Город       |city              | String   |Поле партицирования
|Локация     |loc               | String   |Создать индекс
|id_участника|id_runner         | String   |Создать индекс
|Дата забега |date_r            | Date     |Создать индекс
|Группа      |agroup            | String   |Создать индекс
|Пол         |Bool              | Bool     |True женский/False мужской
|Результат   |result            | Int16    |

Запрос на создание  БД и таблицы:
```sql
create database fivev;

CREATE TABLE fivev.protocols
(
   
    city String NOT NULL,
    loc String NOT NULL,
    id_runner String NOT NULL,
    date_r Date NOT NULL,
    agroup String NOT NULL,
    sex Bool NOT NULL,
    result_ Int8 NOT NULL,
    INDEX loc_idx loc TYPE set(0) GRANULARITY 3,
    INDEX runner_idx id_runner TYPE set(0) GRANULARITY 3,
    INDEX agroup_idx agroup TYPE   set(0) GRANULARITY 3,
    INDEX date_idx date_r TYPE minmax GRANULARITY 3
)
ENGINE = MergeTree
ORDER BY (city)
PARTITION BY city
SETTINGS index_granularity = 8192;
```
Для подключениия из airflow создадим отдельного пользователя и роль:
```sql
CREATE ROLE etl;
CREATE USER airflow DEFAULT ROLE etl IDENTIFIED WITH sha256_password BY 'airflow';
```
Дадим права на БД fivev
```sql
GRANT SELECT ON fivev.* TO etl; 
GRANT INSERT ON fivev.* TO etl; 
```


## Создание диаграмм в DataLens 
## Создание диаграмм в  Apache superset