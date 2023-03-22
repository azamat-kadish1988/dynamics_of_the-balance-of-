# Анализ и визуализация динамики баланса учеников онлайн-школы
## Задачи по проекту:
+ смоделировать изменение балансов студентов онлайн-школы в 2016 году
+ создать таблицу, где будут балансы каждого студента за каждый день
+ изучить изменения балансов студентов, найти аномалии в данных, составить список вопросов дата-инженерам и владельцам таблиц
+ визуализировать, сколько всего уроков было на балансе всех студентов за каждый календарный день, как это количество менялось под влиянием транзакций (оплат, начислений, корректирующих списаний) и уроков (списаний с баланса по мере их прохождения)
+ сделать выводы на основе визуализации
___

## Выводы по проекту:
+ В диаграмме виден рост количества транзакций с апреля месяца и сильный рост с конца сентября,
по пикам можно сказать, что оплаты проходили в начале и в конце месяца, так же можно выдвинуть гипотезу, что пики связаны с маркетинговыми акциями, либо предложением купить сразу пакет уроков, по более выгодной стоимости. Возможно, это связано с тем, что студенты в эти дни получают зарплату (аванс) и тратят на покупку уроков. 
+ Количество пройденных уроков, так же увеличивается к концу года, что говорит, об увеличении продаж, желанием студентов учиться и узнавать новое, и что привлекательность онлайн курсов растет.
+ По линии баланса видно, что студенты прошли не все оплаченные уроки и тут уже надо делать выводы.
  + А) Возможно уроки стали неинтересными, скучными или сложными. Они не соответствуют ожиданиям студентов. Если подтвердится эта гипотеза, необходимо принять меры по улучшению качества, загруженности уроков. А также необходимо будет оценить качество преподавания учителей. 

  + Б) Возможно, нет ничего критичного в такой разности баланса. Может быть такое, что студенты на конец года покупают больше уроков, чем сами проходят. Либо обычная загруженность перед концом года, либо часть уроков студенты будут проходить в следующем 2017 календарном году.
 ___

## 01. Структура базы данных skyeng_db
  1. Узнаем, когда была первая транзакция для каждого студента, начиная с этой даты, будем собирать его баланс уроков
  2. Соберем таблицу с датами за каждый календарный день 2016 года
  3. Узнаем, за какие даты имеет смысл собирать баланс для каждого студента
  4. Найдем изменения балансов студентов, связанные с успешными транзакциями
  5. Найдем баланс студентов, который сформирован только транзакциями
  6. Найдем изменения балансов студентов, связанные с прохождением уроков
  7. Найдем баланс студентов, который сформирован только прохождением уроков
  8. Найдем общий баланс студентов, сформированный транзакциями и прохождением уроков
  9. Посмотрим, как менялось общее количество уроков на балансе всех студентов
____

## 02. SQL - запрос с описанием логики его выполнения.sql
```-- 1 Шаг. Узнаем, когда была первая транзакция для каждого студента. Начиная с этой
даты, мы будем собирать его баланс уроков.

with first_payments
as (
    select p.user_id, 
        min(date_trunc ('day', p.transaction_datetime)) as first_payment_date
    from skyeng_db.payments p
    where p.status_name='success'
    group by 1
    order by 1 asc 
    ),

-- 2 Шаг. Соберем таблицу с датами за каждый календарный день 2016 года.

all_dates as
    (select date_trunc ('day', class_start_datetime) as dt
    from skyeng_db.classes
    where class_start_datetime < '2017-01-01'
    group by 1
    order by 1 asc
    ),

-- 3 Шаг. Узнаем, за какие даты имеет смысл собирать баланс для каждого студента.

all_dates_by_user as
    (select
        user_id,
        all_dates.dt as dt
    from first_payments
    join all_dates
         on all_dates.dt>=first_payments.first_payment_date
    ),
-- 4 Шаг. Найдем все изменения балансов, связанные с успешными транзакциями.

payments_by_dates as  
    (select
        c.user_id,
        date_trunc ('day', c.transaction_datetime) as payment_date,
        sum(c.classes) as transaction_balance_change
    from skyeng_db.payments c
    where c.status_name='success'
    group by 1,2
    order by 1 asc, 2,3
    ),

-- 5 Шаг. Найдем баланс студентов, который сформирован только транзакциями.

payments_by_dates_cumsum as 
    (select
        all_dates_by_user.user_id as user_id,
        all_dates_by_user.dt as dt,
        (coalesce(payments_by_dates.transaction_balance_change, 0)) as transaction_balance_change,
        sum (coalesce(transaction_balance_change,0)) over(partition by all_dates_by_user.user_id order by all_dates_by_user.dt) as transaction_balance_change_cs
    from all_dates_by_user
    left join payments_by_dates
         on payments_by_dates.user_id = all_dates_by_user.user_id
         and payments_by_dates.payment_date = all_dates_by_user.dt
    order by 1 asc, 2
    ),

-- 6 Шаг.  Найдем изменения балансов из-за прохождения уроков.

classes_by_dates as 
    (select
        user_id,
        date_trunc ('day', class_start_datetime) as class_date,
        count (id_class)*-1 as classes
    from skyeng_db.classes
    where 
        class_status in ('success', 'failed_by_student')
        and class_type != 'trial'
     group by 1,2    
    ),

-- 7 Шаг.  Создадим баланс студентов, который сформирован только прохождением уроков. 

classes_by_dates_dates_cumsum as 
    (select
        all_dates_by_user.user_id,
        all_dates_by_user.dt as dt,
        (coalesce(classes,0)) as classes,
        -- case when classes is not null then sum(coalesce(classes,0)) over (partition by all_dates_by_user.user_id order by all_dates_by_user.dt) else 0 end as classes_cs
        sum(coalesce(classes,0)) over (partition by all_dates_by_user.user_id order by all_dates_by_user.dt) as classes_cs
    from all_dates_by_user
    left join classes_by_dates
        on all_dates_by_user.dt = classes_by_dates.class_date
        and all_dates_by_user.user_id = classes_by_dates.user_id
    order by 1
    ),

-- 8 Шаг. Найдем общий баланс студентов, сформированный транзакциями и прохождением уроков.

balances as 
    (select
    classes_by_dates_dates_cumsum.user_id,
    classes_by_dates_dates_cumsum.dt,
    payments_by_dates_cumsum.transaction_balance_change,
    payments_by_dates_cumsum.transaction_balance_change_cs,
    classes_by_dates_dates_cumsum.classes,
    classes_by_dates_dates_cumsum.classes_cs,
    (payments_by_dates_cumsum.transaction_balance_change_cs + classes_by_dates_dates_cumsum.classes_cs) as balance
    from  classes_by_dates_dates_cumsum 
    join payments_by_dates_cumsum 
        on classes_by_dates_dates_cumsum.user_id = payments_by_dates_cumsum.user_id
        and classes_by_dates_dates_cumsum.dt = payments_by_dates_cumsum.dt
    order by 2
    )

/* Задание 1. Выберите топ-1000 строк из CTE balances с сортировкой по 'user_id' и 'dt'.
Посмотрите на изменения балансов студентов. Какие вопросы стоит задавать дата-инженерам и владельцам таблицы?
payments? */

--select *
--from balances
--order by 1
--limit 1000

/* Ответ.
 Почему проводятся уроки, которые не оплачены студентами?
Как отслеживается баланс студента по оплаченным и пройденным урокам? */


/* 9 Шаг. Посмотрим, как менялось общее количество уроков на балансах студентов.*/

select
    dt,
    sum (transaction_balance_change) as sum_transaction_balance_change,
    sum (transaction_balance_change_cs) as sum_transaction_balance_change_cs,
    sum (classes) as sum_classes,
    sum (classes_cs) as sum_classes_cs,
    sum (balance) as sum_balance
from balances
group by 1
order by 1

/* Задание 2
Создайте визуализацию (линейную диаграмму) итогового результата.
Какие выводы можно сделать из получившейся визуализации? */

/* Ответ.
Для решения задачи использовались таблицы: skyeng_db.payments-данные об оплатах и skyeng_db.classes- данные об уроках
Обратимся к линейной диаграмме, которая отражаются изменение данных во времени.
По оси абсцисс - временная шкала 2016 года
По оси ординат с левой стороны, мы видим количество транзакций из таблицы payments и количество пройденных уроков из таблицы classes
Правая сторона Оси ординат показывает кумулятивные суммы количества транзакций и пройденных уроков, а также мы можем наблюдать, как меняется баланс
В диаграмме виден рост количества транзакций (желтая кривая) с апреля месяца и сильный рост с конца сентября,  по пикам можно сказать, что оплаты проходили в начале и в конце месяца,
так же можно выдвинуть гипотезу, что пики связаны с маркетинговыми акциями, либо предложением купить сразу пакет уроков, по более выгодной стоимости.
Количество пройденных уроков (фиолетовая кривая), так же увеличивается к концу года, что говорит, об увеличении продаж, желанием студентов учиться и узнавать новое, и что привлекательность онлайн курсов растет.
По синей линии Баланса видно, что студенты прошли не все оплаченные уроки и тут уже надо делать выводы, 
либо часть уроков студенты будут проходить в следующем 2017 календарном году, либо студенты покупают курсы и не проходят их. */
        
    




## 03. Визуализация динамики общего баланса всех учеников онлайн-школы
+ **sum_transaction_balance_change** - изменение общего баланса под влиянием транзакций (оплат, начислений, корректирующих списаний)
+ **sum_transaction_balance_change_cs** - кумулятивная сумма изменений общего баланса под влиянием транзакций
+ **sum_classes** - изменение общего баланса под влиянием уроков (списаний с баланса по мере их прохождения)
+ **sum_classes_cs** - кумулятивная сумма изменений общего баланса под влиянием уроков
+ **sum_balance** - общее количество уроков на балансе всех студентов
____

## 04. Таблицы с итоговыми данными, выводы и вопросы по консистентности
  ### Вопросы к дата-инженерам и владельцам таблиц:
  #### В базе за 2016 год присутствует минусовой баланс по оплаченным и пройдённым урокам.
 + Если мы выберем топ-1000 строк из CTE `balances` с сортировкой по "user_id" и "dt" и изучим  изменение баланса студентов, можем увидеть, что у некоторых студентов имеется отрицательный баланс. То есть, студенты прошли уроков больше, чем купили на самом деле. Чтобы убедиться, что это проблема действительно имеет место быть, отсортируем не по "user_id" и "dt" , а по "balance asc " и увидим, что у значительного количества студентов имеется отрицательный баланс, и он достаточно  внушителен. Это говорит о том, что дата-инженерами и владельцами таблицы `payments` совершена ошибка при заполнении данной таблицы. 
 + Дата-инженерам можно указать также и на то, что в другой таблице  «classes» имеются неточности. Например, время окончания урока раньше времени начала урока.  


1. Почему проводятся уроки, которые не оплачены студентами?
2. Как отслеживается баланс студента по оплаченным и пройденным урокам?
3. При удалении уроков не всегда проставляется информация о статусе?
4. Как так произошло, что  время окончания урока, раньше его начала?



