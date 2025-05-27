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
    result_ Int16 NOT NULL,
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
Следующим шагом надо заполнить таблицу. 
Маловероятно, что получится спарсить все данные за один запуск скрипта, поэтому используем airflow-даг, чтобы сделать это пошагово.
Заполять таблицу планируется данными за 2023-2024 гг c локаций:
```python
locations = {
    'http://5verst.ru/zatyumensky/': 'Тюмень',
    'http://5verst.ru/tyumenparkgagarina/': 'Тюмень',
    'http://5verst.ru/plotinka/': 'Екатеринбург',
    'http://5verst.ru/parkmayakovskogo/': 'Екатеринбург',
    'http://5verst.ru/tobolsk/': 'Тобольск',
    'http://5verst.ru/solnechnyostrov/': 'Краснодар',
    'http://5verst.ru/chistyakovskayaroshcha/': 'Краснодар',
    'http://5verst.ru/krasnoyarsknaberezhnaya/': 'Красноярск',
    'http://5verst.ru/sirius/': 'Сочи'
}
```
Здесь ключами словаря locations является часть url-адреса страницы, откуда планируется парсить данные, а значение город к которому эти данные относятся

За 1 запуск дага планируется спарсить:

## Подготовка
После того как я собрала скриптом данные, хорошо бы проверить их правильность
Для этого я убедилась что в таблице protocols присутствуют записи по всем локациям
А также что статистика по участнику с id_runner = 790145311  (это я) верная 
Согласно https://5verst.ru/userstats/790145311/ 


Так как две из моих пробежек на момент запуска скрипта были в 2025, по условию runner_id = 790145311
должно вернуться 30 строк (2 с тобольском, одна с Красноярском, одна с Краснодаром и 28 с Тюменью)
общее 

Какую информацию получить по каждому участнику
1) Общее количество пробежек 
2) Домашняя локация - ей буду считать локацию где бегун совершил максимальное число пробежек (при условии хотя бы две), в остальных случаях неизвестно
3) Лучшее время общее (с указанием локации в скобочках)
4) Лучшее место пол (с указанием локации и результата)
5) Лучшее место категория (с указанием локации, результата, категории - т.к возраст бегуна меняется)
6) Лучшее место общее (с указанием локации и результата)



По пункту 1: все просто - количество записей в таблице fivev.protocols где id_runner=xxx 
Пункт 2: нужно собрать статистику сколько пробежек совершил бегун на каждой из 8 локаций 
Для начала просто посчитаем сколько раз и на какой локации бегали бегуны
```SQL
   SELECT id_runner, loc, count() AS cnt
    FROM fivev.protocols
    GROUP BY id_runner, loc
```
В результате нам нужно будет оставить только те записи, где количество пробежек для данного бегуна максимальное
Для этого можно использовать argMax (в других СУБД пришлось бы использовать подзапрос)
```SQL
SELECT 
    id_runner,
    argMax(loc, cnt) AS most_frequent_loc,
    max(cnt) AS run_count
FROM (
    SELECT id_runner, loc, count() AS cnt
    FROM fivev.protocols
    GROUP BY id_runner, loc
)
GROUP BY id_runner
```
Дополним запрос лучшим временем и локацией где оно было получшено
```sql

SELECT 
    id_runner,
    argMax(loc, cnt) AS most_frequent_loc,
    max(cnt) AS run_count,
    argMin(loc, best_result) AS best_result_loc,
    min(best_result) AS best_result_time,
    argMin(loc,best_result) as best_time_loc
FROM (
    SELECT 
        id_runner, 
        loc, 
        count() AS cnt, 
        min(result_) AS best_result
    FROM fivev.protocols
    GROUP BY id_runner, loc
)
GROUP BY id_runner
```

И было бы интересно посмотреть сколько пробежек бегун совершил на каждой локации
Казалось что нужно использовать PIVOT - но в clickhouse такого нет 
Зато есть  CountIf 

```sql
SELECT 
    id_runner,
    count(distinct date_r) "Пробежек",
    COUNTIf(loc = 'https://5verst.ru/zatyumensky/') AS "Затюменский",
    COUNTIf(loc = 'https://5verst.ru/tyumenparkgagarina/') AS "Парк Гарина Тюмень",
    COUNTIf(loc = 'https://5verst.ru/tobolsk/') AS "Тобольск",
    COUNTIf(loc = 'https://5verst.ru/plotinka/') AS "Плотинка Екатегринбург",
    COUNTIf(loc ='https://5verst.ru/parkmayakovskogo/') as  "Парк Маяковского Екатеринбург",
    COUNTIf(loc ='https://5verst.ru/solnechnyostrov/') as "Солнечный Остров Краснодар",
    COUNTIf(loc ='https://5verst.ru/chistyakovskayaroshcha/') AS "Чистяковская роща Краснодар",
    COUNTIf(loc ='https://5verst.ru/krasnoyarsknaberezhnaya/') AS "Красноярск Набережная",
    COUNTIf(loc ='https://5verst.ru/sirius/') AS "Сириус Сочи"
FROM 
    fivev.protocols
GROUP BY 
    id_runner
having count(DISTINCT loc) >3
order by count(DISTINCT loc) desc
```

Было бы интересно построить диаграмму по дате и числу участников



## Создание диаграмм в DataLens 
### Вкладка статистика локаций
Первый чарт это динамика количества участников в течении времени
Для него его я создала таблицу
```
	
	CREATE table fivev.locations_stats  engine=MergeTree order by date_r as 
	
	select date_r, loc, count(id_runner) c,
	COUNTIf(sex) female_c,
		COUNTIf(not sex) male_c,
	from fivev.protocols
	group by date_r, loc
```
В график я добавила необязательный фильтр по полю LOC 
Так как из выбранных мной локаций zatyumensky 
![mage_alt](https://github.com/marinkal/tryBI/blob/main/images/1.png)
## Создание диаграмм в  Apache superset
