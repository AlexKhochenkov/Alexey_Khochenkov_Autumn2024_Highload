# Программирование высоконагруженного сервиса напоминаний и планирования событий

# 1 Тема и целевая аудитория

## Тема

Google Calendar - сервис для планирования мероприятий, задач и встреч, позволяющий указывать подробную информацию и устанавливать напоминания, предоставляющий совместный доступ к планированию.

## MVP

1. Авторизация с помощью Google
2. Отображение календаря
3. Добавление новых мероприятий/задач
4. Редактирование информации о мероприятии (время, место и т. д.)
5. Установка напоминаний о событии
6. Прикрепление других пользователей к событию
7. Совместное отображение и редактирование календарей
8. Поиск по событиям

## ЦА

Размер целевой аудитории: около 500 млн. пользователей в месяц. [[1]](https://earthweb.com/blog/google-calendar-users/) [[3]](https://zipdo.co/google-calendar-statistics/)

Ниже приведена статистика по возрасту целевой аудитории, начиная с 18 лет [[2]](https://pro.similarweb.com/#/digitalsuite/websiteanalysis/audience-demographics/*/999/1m?webSource=Total&key=calendar.google.com)
Возрастная группа | Процентное соотношение
------------ | -------------
18-24 года| 14,05% 
25-34 года| 29,47% 
35-44 года| 20,31%
45-54 года| 16,77%
55-64 года| 11,92%
65+| 7,48%

Гендерное распределение аудитории Google Calendar [[2]](https://pro.similarweb.com/#/digitalsuite/websiteanalysis/audience-demographics/*/999/1m?webSource=Total&key=calendar.google.com):
Пол          | Процентное соотношение
-------------| -------------
Мужской      | 50,26%
Женский      | 49,74% 

Общемировая география пользователей Google Calendar [[2]](https://pro.similarweb.com/#/digitalsuite/websiteanalysis/audience-geography/*/999/1m?key=calendar.google.com&webSource=Total):  
Регион           | Процентное соотношение
---------------- | -------------
США                     | 34,42%
Япония                  | 19,83% 
Бразилия                | 4,12% 
Объединённое королевство| 2,95% 
Канада                  | 2,9% 
Остальные               | 35,76%

# 2 Расчёт нагрузки

## Продуктовые метрики

Информация о DAU сервиса отсутствует в открытых источниках, поэтому примем, что DAU/MAU = 25%, так как сервис используется для планирования дневных задач и мероприятий, что предполагает частое использование. В таком случае DAU = 1/4MAU = 125 млн. пользователей.

Метрика      | Значение
-------------| -------------
Месячная аудитория MAU        | 500 млн. пользователей
Дневная аудитория DAU         | 125 млн. пользователей
Средняя длительность сессии   | 07:23 минут [[4]](https://webstatsdomain.org/d/calendar.google.com)

### Хранилище данных для пользователя

Сначала посчитаем вес одного события в календаре. Поскольку символы в UTF-8 могут кодироваться разным количеством байтов(от 1 до 4), а Google Calendar имеет многоязычную аудиторию, то можно взять средний размер символа равный 3 байтам.

Параметр          | Вес
-------------| -------------
Название                                   | 128 символов UTF-8
Описание                                   | 1024 символа UTF-8
Дата создания                              | 8 Б
Дата последнего обновления                 | 8 Б
Дата начала события                        | 8 Б
Дата окончания события                     | 8 Б
Ссылка на создателя                        | 8 Б
Email гостя                                | 255 Б
Максимальное количество гостей             | 20
Местоположение                             | 8 Б
Календарь                                  | 8 Б
Общий размер                               | 128 * 3 + 1024 * 3 + 8 + 8 + 8 + 8 + 8 + 255 * 20 + 8 + 8 = 8604 Б

В среднем каждый день средний пользователь среднего календаря создает 5 событий [[7]](https://reclaim.ai/blog/productivity-report-one-on-one-meetings).

По версии сайта marketsplash.com в среднем в день создаётся около 1.5 млрд событий. [[8]](https://web.archive.org/web/20240424130647/https://marketsplash.com/google-workspace-statistics/). Тогда, поделив это число на DAU, получим 1.5 млрд/125 млн = 12 событий.

События хранятся в памяти 1 год после завершения [[9]](https://www.calendar.com/blog/how-do-you-look-up-past-appointments-in-your-calendar/), поэтому на одного пользователя в среднем нужно: 

365 * 12 * 8604 Б = 35,94 Мб

Также есть лимит в 10000 событий, после которого старые события удаляются [[6]](https://classroom.synonym.com/google-calendar-limits-17378.html).

Посчитаем вес одного календаря.

Параметр          | Вес
-------------| -------------
Название                                   | 128 символов UTF-8
Описание                                   | 1024 символа UTF-8
Дата создания                              | 8 Б
Дата последнего обновления                 | 8 Б
Ссылка на создателя                        | 8 Б
Часовой пояс                               | 8 Б
Общий размер                               | 8 + 8 + 8 + 8 + 128 * 3 + 1024 * 3 = 3488 Б

На календари у пользователей также есть лимит в 25 штук [[6]](https://classroom.synonym.com/google-calendar-limits-17378.html).

25 * 3488 Б = 85,156 Кб

Посчитаем средний размер хранилища пользователя.

Параметр          | Вес
-------------| -------------
Добавление событий                         | 20 запросов/день
Получение напоминаний                      | 40 запросов/день
Размер событий                             | 35,94 Мб
Размер календарей                          | 85,156 Кб
Информация о пользователе                  | 1 Кб
Общий размер                               | 82,054 Мб + 85,156 Кб + 1 Кб = 82,138 Мб

## Технические метрики

### Размер хранения в разбивке по типам данных

Поскольку было найдено примерное количество пользователей, а MAU = 500 млн, то можно рассчитать примерный размер хранения по этим типам данных:

По событиям: 35,94 Мб * 500 млн = 17 137,3904 Тб = 16,73 Пб

По календарям: 25 * 3488 Б * 500 млн = 39,65 Тб

### Динамический рост

Сейчас Gmail имеет базу поьзователей в 4.3 млрд (на момент 18 апреля 2024). Прогнозируют рост этого показателя до 4.6 млрд к 2025 [[11]](https://www.broadbandsearch.net/blog/google-statistics-facts). Поскольку Gmail непосредственно связан с Google Calendar, можно предположить, что рост его базы пользователей будет пропорциональным. Таким образом, можем посчитать на сколько процентов повышается база пользователей Gmail и применить этот показатель к MAU Google Calendar.

500 * ((4 600 000 000 / 4 300 000 000 - 1) / 8) = 4,36 млн

Получаем, что каждый месяц число пользователей увеличивается на 4,36 млн. Теперь можем посчитать ежемесячный прирост хранилища данных:

4,36 млн * 82,138 Мб = 341,5 Тб

### Сетевой трафик

Следует учесть, что при добавлении гостя к событию, гостю приходит приглашение, а также то, что гость должен ответить на приглашение. Возьмём, что в среднем к событию прикрепляется один гость.

1. Просмотр календаря:

* 25 запросов/день. 25 * 3488 Б = 85,16 Кб

2. Добавление событий:

* 12 запросов/день. 12 * 8604 Б = 100,8 Кб

3. Получение напоминаний. Поскольку на одно событие приходится одно напомининие:

* 12 запросов/день. Оценим напоминание в 1 Кб. 12 * 1Кб = 12 Кб.

4. Получение приглашений. Поскольку на одно событие приходится одно приглашение:

* 12 запросов/день. Оценим приглашение в 1 Кб. 12 * 1Кб = 12 Кб.

5. Получение ответов на приглашение. Поскольку на одно событие приходится одно приглашение:

* 12 запросов/день. Оценим ответ на приглашение в 1 Кб. 12 * 1Кб = 12 Кб.

### Сетевой трафик по видам активности в виде таблицы

Пиковое значение пользователей можно оценить примерно в 2 раза. Возьмём запас в 2.5.

DAU = 125 млн.

Тип                           | Отправка  | Отправка Гб/сек | Пиковое значение Гб/сек |
-------------                 | ------------------------------------  |--------------- |----------------- |
Просмотр календаря            | 125 млн * 85,16 Кб = 10 151,8 Гб      | 0.117          | 0.294            |
Добавление событий            | 125 млн * 100,8 Кб = 12 016,3 Гб      | 0.139          | 0.348            | 
Получение напоминаний         | 125 млн * 12 Кб = 1430,5 Гб           | 0.017          | 0.041            | 
Получение приглашений         | 125 млн * 12 Кб = 1430,5 Гб           | 0.017          | 0.041            |
Получение ответов на приглашения         | 125 млн * 12 Кб = 1430,5 Гб           | 0.017          | 0.041            |
Итого                         | 26 460 Гб	                            | 0.306          | 0.766            |

### RPS

Тип                                 | RPS  | Пиковое значение |
------------------------------------| ---  |----------------- |
Просмотр календаря                  | 125 млн * 25 / 86400 сек = 36 169   | 90 423            |
Добавление событий                  | 125 млн * 12 / 86400 сек = 17 361   | 43 403            |
Получение напоминаний               | 125 млн * 12 / 86400 сек = 17 361   | 43 403            |
Получение приглашений               | 125 млн * 12 / 86400 сек = 17 361   | 43 403            |
Получение ответов на приглашения    | 125 млн * 12 / 86400 сек = 17 361   | 43 403            |
Итого                               | 125 млн * 73 / 86400 сек = 105 613  | 264 034           |

# 3 Глобальная балансировка нагрузки

По версии сайта hypestat.com распределение пользователей по местонахождению имеет следующий характер [[10]](https://hypestat.com/info/calendar.google.com):

Регион           | Процентное соотношение
---------------- | -------------
США                     | 18,8%
Индия                   | 10,7% 
Япония                  | 5,2% 
Россия                  | 2,9% 
Бразилия                | 2,8% 

Исходя из этих данных, а также из данных, представленных в 1 пункте, можно сделать вывод, что наибольшее количество пользователей находится в США. Также крупными регионами являются Индия, Япония, Бразилия и Европа.

Из этого можно сделать вывод, что следующее расположение ДЦ будет наиболее предпочтительным:

* Лос-Анджелес, США

* Нью-Йорк, США

* Майами, США

* Сан-Паулу, Бразилия

* Франкфурт, Германия

* Мумбаи, Индия

* Токио, Япония

![Карта физического расположения ДЦ](./images/Stage3.jpg)

Полную карту можно посмотреть здесь (https://yandex.ru/maps/?um=constructor%3Ae21060ef020adb7da3aa4cf59ba341c5599006ff95ce84aca2b34eb51499d540&source=constructorLink)

## RPS по регионам

Тип запроса                  |	RPS (Лос-Анджелес) |	RPS (Нью-Йорк) |	RPS (Майами) |	RPS (Сан-Паулу)	| RPS (Франкфурт) |	RPS (Мумбаи) |	RPS (Токио)
-----------------------------| -------------------| -------------------| --------------------| ---------------| ----------------| -------------| -------------
Просмотр календаря           |	12659.22             |	12659.22             |	7233.84              |	9042.3          |	21701.52          |	13563.45      |	13563.45
Добавление событий           |	6076.42            |	6076.42             |	3472.24	             | 4340.3          |	10416.72	          | 6510.45      |	6510.45
Получение напоминаний        |	6076.42            |	6076.42             |	3472.24	             | 4340.3          |	10416.72	          | 6510.45      |	6510.45
Получение приглашений        |	6076.42            |	6076.42             |	3472.24	             | 4340.3          |	10416.72	          | 6510.45      |	6510.45
Получение ответов на приглашения            |	6076.42            |	6076.42             |	3472.24	             | 4340.3          |	10416.72	          | 6510.45      |	6510.45
Итого                        |	36964.76           |	36964.76           |	21122.72             |	26403.4         |	63368.16          |	39605.1      |	39605.1

## DNS-балансировка

В данном случае эффективно использовать Geo-based DNS, так как у нас есть ряд ярко выраженных регионов, которые расположены далеко друг от друга и в каждом из которых есть ДЦ.

## BGP Anycast

Если рассматривать регион США, то видно, что в нём находится 3 ДЦ, а также большинство пользователей сервиса. Исходя из этого, применим технологию BGP Anycast, которая в рамках США будет справляться с выбором необходимого ДЦ для конкретного пользователя и учитывать задержку.

# 4 Локальная балансировка нагрузки

Для балансировки был выбран L7 балансировщик, так как он обеспечивает равномерное распределение нагрузки и поможет решить проблему медленных клиентов.

Будем использовать балансировщик NGINX, так как он обладает высокой производительностью, гибко настраивается и поддерживает большое количество протоколов. В качестве стратегии балансировки можно выбрать round robin. Для кэширования SSL в целях оптимизации выберем Session Cache.

Для оркестрации сервисов будем использовать Kubernetes по следующим причинам:

* Масштабирование через auto-scaling, позволяющее эффективно использовать ресурсы

* Распределение и перераспределение экземпляров по узлам

Для обеспечения отказоустойчивости будем применять протокол VRRP(Virtual Router Redundancy Protocol), который позволит нам при отказе одного из балансировщиков перенести весь трафик на другую машину, поднятую на том же IP адресе. Так же можно использовать keepalive соединение, которое позволит отслеживать и оповещать балансировщик о доступности узлов.

# 5 Логическая схема БД

![Схема БД](./images/Stage5.jpg)

Полную схему можно посмотреть здесь (https://dbdiagram.io/d/Copy-of-Stage_5-670cc75897a66db9a3de522e)

## Размеры данных в таблице

| User                 | Calendar             | Event                 | Session          | User_to_calendar  | User_to_event  | Reminder       |
| -------------------- | -------------------- | --------------------- | ---------------- | ----------------- | -------------- | -------------- |
| id (8 B)             | id (8 B)             | id (8 B)              | id (8 B)         | id (8 B)          | id (8 B)       | id (8 B)       |
| email (255 B)        | name (128 B)         | calendar_id (8 B)     | user_id (8 B)    | user_id (8 B)     | user_id (8 B)  | user_id (8 B)  |
| password_hash (128 B)        | user_id (8 B)       | user_id (8 B)         | create_date (8 B) | calendar_id (8 B) | event_id (8 B) | event_id (8 b)  |
| username (128 B)     | timezone (72 B)      | create_date (8 B)     | expire_date (8 B) |                  |                | remind_time (8 B)
|                      | description (1024 B) | update_date (8 B)     |                  |                   |                |
|                      |                      | start_date (8 B)      |                  |                   |                |
|                      |                      | end_date (8 B)        |                  |                   |                |
|                      |                      | name (128 B)          |                  |                   |                |
|                      |                      | description (1024 B)  |                  |                   |                |
|                      |                      | location (4 B)        |                  |                   |                |

Итого:

|                    | User                 | Calendar             | Event                 | Session          | User_to_calendar  | User_to_event  | Reminder     |
| ------------------ | -------------------- | -------------------- | --------------------- | ---------------- | ----------------- | -------------- | ------------ |
| Вес записи         | 517 B                | 1240 B               | 1212 B                | 32 B             | 24 B              | 24 B           | 32 B         |
| Количество записей | 500 млн (MAU)        | 25 * 500 млн         | 365 * 12 * 500 млн    | 125 млн (DAU)    | 25 * 500 млн * 10 | 365 * 12 * 500 млн * 2 | 365 * 12 * 500 млн         |
| Общий размер       | 240 Gb               | 14 435 Gb            | 2 414 Tb              | 3,72 Gb          | 2 793 Gb          | 97 900 Gb      | 65 267 Gb    |

Запросов на чтение в БД поступает значительно больше, чем запросов на запись во всех таблицах.

# 6 Физическая схема БД

## Индексы

В качестве индексов выбраны:

* Для таблицы Calendar индекс на user_id, для быстрой выдачи календарей пользователя

* Для таблицы Event индекс на user_id, для быстрого отображения событий пользователя, индекс на start_date для отображения событий в нужном временном диапозоне

* Для таблицы Reminder индекс на remind_date, так как необходимо быстро отображать напоминания при достижении временной отметки

## Денормализация

Чтобы избавиться от таблиц User_to_calendar и User_to_event, так как они будут требовать JOIN запросов, что крайне не эффективно, добавим в таблицы Event и Calendar поля-массивы users, которые будут содержать всех участников события и календаря соответсвенно.

![Схема БД](./images/Stage6.jpg)

Полную схему можно посмотреть здесь (https://dbdiagram.io/d/Copy-of-Stage_6-670cc75897a66db9a3de522e)

## Выбор СУБД

Для всех таблиц, кроме Session, рационально использовать PostgreSQL, так как количество запросов на чтение значительно больше запросов на запись, для чего подходит данная СУБД.

Для хранения сессий рационально использовать in-memory СУБД, например Tarantool.

| User                 | Calendar             | Event                 | Session          | Reminder       |
| -------------------- | -------------------- | --------------------- | ---------------- | -------------- |
| PostgreSQL           | PostgreSQL           | PostgreSQL            | Tarantool        | PostgreSQL     |

## Шардирование

| User                 | Calendar             | Event                 | Session          | Reminder       |
| -------------------- | -------------------- | --------------------- | ---------------- | -------------- |
| id                   | id                   | id                    | id               | id             |

Шардировать рационально по полю id, чтобы обеспечить равномерное распределение по шардам.

## Клиентские библиотеки

Для языка Go, на котором будет написан backend, можно использовать следующие библиотеки:

PostgreSQL: pgx

Tarantool: go-tarantool

## Резервное копирование

Будем проводить полное резервное коипрование каждые 2 недели, а так же дифференциальное резервное коипрование каждые два дня.

# Список источников

1. https://earthweb.com/blog/google-calendar-users/
2. https://pro.similarweb.com/#/digitalsuite/websiteanalysis/audience-demographics/*/999/1m?webSource=Total&key=calendar.google.com
3. https://zipdo.co/google-calendar-statistics/
4. https://webstatsdomain.org/d/calendar.google.com
5. https://www.template.net/google/how-long-does-google-calendar-keep-past-events/
6. https://classroom.synonym.com/google-calendar-limits-17378.html
7. https://reclaim.ai/blog/productivity-report-one-on-one-meetings
8. https://web.archive.org/web/20240424130647/https://marketsplash.com/google-workspace-statistics/
9. https://www.calendar.com/blog/how-do-you-look-up-past-appointments-in-your-calendar/
10. https://hypestat.com/info/calendar.google.com
