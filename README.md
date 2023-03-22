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
```with first_payments as
        (select     user_id  
                , date_trunc('day',min(transaction_datetime)) first_payment_date
            from SKYENG_DB.payments
            where status_name = 'success'     
            group by user_id
            order by user_id)
, 
    all_dates as
        (select distinct date_trunc('day', class_end_datetime) dt
            from  SKYENG_DB.classes
            where class_end_datetime >= '2016-01-01' and  class_end_datetime < '2017-01-01' )
,
    payments_by_dates as
        (select    user_id
                , date_trunc('day',transaction_datetime) payment_date
                , sum(classes) transaction_balance_change
            from  SKYENG_DB.payments
            where status_name = 'success'
            group by  user_id
                    , payment_date)
,
    all_dates_by_user as
        (select    user_id
                , dt 
            from all_dates q
        join first_payments w
            on q.dt >= w.first_payment_date)
,
    classes_by_dates as
        (select    user_id
                , date_trunc('day', class_end_datetime) class_date     
                , count(id_class) * (-1) classes 
            from  SKYENG_DB.classes
            where class_type != 'trial' and (class_status = 'success' or class_status = 'failed_by_student')
            group by  user_id
                , class_date
            order by classes)
, 
    payments_by_dates_cumsum as
        (select   q.user_id user_id
                , dt
                , coalesce(transaction_balance_change,0) transaction_balance_change 
                , sum(transaction_balance_change) over(partition by q.user_id  order by  dt) transaction_balance_change_cs
            from all_dates_by_user q
        left join payments_by_dates w
            on  q.dt = w.payment_date 
                and
                q.user_id = w.user_id)
,
    classes_by_dates_dates_cumsum as
        (select   q.user_id user_id
                , dt
                , coalesce(classes,0) classes 
                , sum(classes) over(partition by q.user_id  order by  dt) classes_cs
            from all_dates_by_user q
        left join classes_by_dates w
            on  q.dt = w.class_date
                and
                q.user_id = w.user_id)
,
    balances as
        (select   q.user_id user_id
                , q.dt dt
                , transaction_balance_change
                , transaction_balance_change_cs
                , classes
                , classes_cs
                , (classes_cs + transaction_balance_change_cs) balance 
            from payments_by_dates_cumsum q
        left join classes_by_dates_dates_cumsum w
            on  q.dt  = w.dt
                and
                q.user_id  = w.user_id)
               
    -- select *
    --     from balances
    --     order by 
    --             --   user_id
    --             -- , dt
    --              balance asc
    --     limit 1000
    
  select     dt
            , sum(transaction_balance_change) transaction_balance_change
            , sum(transaction_balance_change_cs) transaction_balance_change_cs
            , sum(classes) classes
            , sum(classes_cs) classes_cs
            , sum(balance) balance
        from balances
        group by dt
        order by dt
        ```        
    




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



