# SQL Analysis - Retention (IT Resume)

В этом разделе представлены SQL-запросы, использованные для анализа удержания пользователей платформы IT Resume.

Цель анализа - понять:

* как часто пользователи возвращаются
* где происходит основной отток
* насколько продукт становится привычкой

---

## Data Model

В анализе используются следующие таблицы:

* `users` - информация о пользователях (дата регистрации)
* `userentry` - события входа пользователя в систему
* `coderun` - запуск кода
* `codesubmit` - отправка решения

Действия из `coderun` и `codesubmit` используются как **первое осмысленное действие пользователя**

---

# 1. DAU (Daily Active Users)

## Цель

Определить среднее количество активных пользователей в день.

## Логика

* считаем уникальных пользователей по дням
* затем берём среднее значение

## SQL

```sql
with daus as (

select
    to_char(entry_at, 'YYYY-MM-DD') as dt,
    count(distinct user_id) as cnt

from
    userentry u

where
    to_char(entry_at, 'YYYY') >= '2022'

group by
    to_char(entry_at, 'YYYY-MM-DD')

)

select round(avg(cnt)) as dau
from daus;
```

## Интерпретация

DAU показывает ежедневную активность пользователей и помогает оценить масштаб продукта.

---

# 2. MAU (Monthly Active Users)

## Цель

Определить среднее количество активных пользователей в месяц.

## Логика

* считаем уникальных пользователей по месяцам
* исключаем “неполные” месяцы

## SQL

```sql
with maus as (

select
    to_char(entry_at, 'YYYY-MM') as dt, count(distinct user_id) as cnt

from userentry u

where to_char(entry_at, 'YYYY') >= '2022'

group by to_char(entry_at, 'YYYY-MM')

having count(distinct entry_at::date) >= 25

)

select round(avg(cnt)) as mau
from maus;
```

## Интерпретация

MAU отражает общий размер активной аудитории продукта.

---

# 3. Stickiness (DAU / MAU)

## Цель

Оценить, насколько продукт становится привычкой для пользователя.

## Формула

DAU / MAU

## Интерпретация

* низкое значение -> пользователи заходят редко
* высокое значение -> продукт встроен в повседневную жизнь

---

# 4. N-day Retention

## Цель

Определить, какой процент пользователей возвращается в конкретные дни после регистрации.

## Логика

* считаем разницу между датой входа и датой регистрации
* группируем пользователей по когортам (месяц регистрации)
* считаем долю вернувшихся пользователей

## SQL

```sql
with diffs as (

    select 

        ue.user_id, 

        ue.entry_at::date as entry_at,

        u.date_joined::date as date_joined, 

        extract(day from ue.entry_at - u.date_joined) as diff,

        to_char(u.date_joined, 'YYYY-MM') as cohort

    from userentry ue join users u 

        on ue.user_id = u.id 

    where to_char(u.date_joined, 'YYYY')  = '2022'

    )

select

    cohort,

    round(count(distinct case when diff = 0 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "0 (%)",

    round(count(distinct case when diff = 1 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "1 (%)",

    round(count(distinct case when diff = 3 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "3 (%)",

    round(count(distinct case when diff = 7 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "7 (%)",

    round(count(distinct case when diff = 14 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "14 (%)",

    round(count(distinct case when diff = 30 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "30 (%)",

    round(count(distinct case when diff = 60 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "60 (%)",

    round(count(distinct case when diff = 90 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "90 (%)"

from diffs

group by cohort;
```

## Интерпретация

Позволяет увидеть, в какие дни происходит максимальный отток пользователей.

---

# Rolling Retention

## Цель

Оценить вероятность того, что пользователь вернётся хотя бы один раз после определённого дня.

## Логика

* считаем пользователей, вернувшихся в этот день или позже
* делим на всех пользователей

## SQL

```sql
with diffs as (

select ue.user_id, 

    ue.entry_at::date as entry_at,

    u.date_joined::date as date_joined, 

    extract(day from ue.entry_at - u.date_joined) as diff,

    to_char(u.date_joined, 'YYYY-MM') as cohort

from userentry ue join users u 

on ue.user_id = u.id 

where to_char(u.date_joined, 'YYYY')  = '2022'

    )

select

    cohort,

    round(count(distinct case when diff >= 0 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "0 (%)",

    round(count(distinct case when diff >= 1 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "1 (%)",

    round(count(distinct case when diff >= 3 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "3 (%)",

    round(count(distinct case when diff >= 7 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "7 (%)",

    round(count(distinct case when diff >= 14 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "14 (%)",

    round(count(distinct case when diff >= 30 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "30 (%)",

    round(count(distinct case when diff >= 60 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "60 (%)",

    round(count(distinct case when diff >= 90 then user_id end) * 100.0 / count(distinct case when diff >= 0 then user_id end), 2) as "90 (%)"

from diffs

group by cohort;
```

## Интерпретация

Показывает долгосрочную вовлечённость и нерегулярные возвраты пользователей.

---

# TTFV (Time To First Value)

## Цель

Определить, сколько времени проходит до первого осмысленного действия пользователя.

## Логика

* объединяем действия пользователя
* находим первое действие
* считаем разницу с регистрацией

## SQL

```sql
with actions as (

select user_id, created_at as dt

from coderun

union

select user_id, created_at as dt

from codesubmit

),

diffs as (

select u.id, u.date_joined, MIN(dt) as first_action, extract(day from MIN(dt) - u.date_joined::date) as diff

from users u

join actions a

on u.id = a.user_id

where to_char(date_joined, 'YYYY') >= '2022'

group by u.id

)

select avg(diff) as ttfv

from diffs;
```

## Интерпретация

Одна из ключевых метрик:

* высокий TTFV -> пользователь не видит ценности
* низкий TTFV -> быстрее вовлекается

---

# Churn Rate

## Цель

Определить долю пользователей, которые перестают использовать продукт.

## Логика

* считаем пользователей, не вернувшихся в конкретный день
* рассчитываем процент от общего числа

## SQL

```sql
with diffs as (

select

ue.user_id,

ue.entry_at::date as entry_at,

u.date_joined::date as date_joined,

extract(day from ue.entry_at - u.date_joined) as diff,

to_char(u.date_joined, 'YYYY-MM') as cohort

from userentry ue

join users u

on ue.user_id = u.id

where to_char(u.date_joined, 'YYYY') = '2022'

)

select

cohort,

round(100 - count(distinct case when diff = 0 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "0 (%)",

round(100 - count(distinct case when diff = 1 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "1 (%)",

round(100 - count(distinct case when diff = 3 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "3 (%)",

round(100 - count(distinct case when diff = 7 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "7 (%)",

round(100 - count(distinct case when diff = 14 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "14 (%)",

round(100 - count(distinct case when diff = 30 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "30 (%)",

round(100 - count(distinct case when diff = 60 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "60 (%)",

round(100 - count(distinct case when diff = 90 then user_id end) * 100.0 / count(distinct case when diff = 0 then user_id end), 2) as "90 (%)"

from diffs

group by cohort;
```

## Интерпретация

Позволяет количественно оценить отток пользователей и его динамику.
