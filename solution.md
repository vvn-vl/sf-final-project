#### Задание 1

```sql
WITH cohort_users AS (
    SELECT 
        id,
        DATE_TRUNC('month', date_joined) AS cohort_month,
        date_joined::DATE AS reg_date
    FROM users
),
user_activity AS (
    SELECT DISTINCT
        u.id,
        u.cohort_month,
        (ue.date - u.reg_date) AS day_diff
    FROM userentry ue
    JOIN cohort_users u ON ue.user_id = u.id
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(*) AS total_users
    FROM cohort_users
    GROUP BY cohort_month
),
retention_days AS (
    SELECT 
        cohort_month,
        day_n
    FROM (SELECT DISTINCT cohort_month FROM cohort_users) AS months,
    (VALUES (0),(1),(3),(7),(14),(30),(60),(90)) AS days(day_n)
),
user_max_day AS (
    SELECT 
        id,
        cohort_month,
        MAX(day_diff) AS max_day
    FROM user_activity
    GROUP BY id, cohort_month
)
SELECT 
    r.cohort_month::DATE AS cohort_month,
    r.day_n,
    ROUND(100.0 * COUNT(DISTINCT CASE WHEN u.max_day >= r.day_n THEN u.id END) / c.total_users, 2) AS retention
FROM retention_days r
JOIN cohort_sizes c ON r.cohort_month = c.cohort_month
LEFT JOIN user_max_day u ON r.cohort_month = u.cohort_month
GROUP BY r.cohort_month, r.day_n, c.total_users
ORDER BY r.cohort_month, r.day_n;
```

Выводы: Retention на 0 день обычно высокий (80–100%), затем резко падает к 1–3 дню. Если на 7–14 день retention >30–40%, пользователи активно возвращаются.К 30 дню retention часто падает ниже 20%, а к 60–90 – ниже 10%.
Рекомендуемые сроки подписки:
Краткосрочная: 1–2 недели (покрывает активных в первую неделю).
Долгосрочная: 1 месяц (основной цикл) или год (для ядра).


#### Задание 2

```sql
WITH user_balance AS (
    SELECT 
        t.user_id,
        SUM(CASE WHEN tt.type IN (1,23,24,25,26,27,28,30) THEN t.amount ELSE 0 END) AS spent,
        SUM(CASE WHEN tt.type NOT IN (1,23,24,25,26,27,28,30) THEN t.amount ELSE 0 END) AS earned,
        SUM(CASE WHEN tt.type NOT IN (1,23,24,25,26,27,28,30) THEN t.amount 
                 ELSE -t.amount END) AS balance
    FROM transaction t
    JOIN transactiontype tt ON t.type_id = tt.id
    GROUP BY t.user_id
)
SELECT 
    ROUND(AVG(spent), 2) AS avg_spent_per_user,
    ROUND(AVG(earned), 2) AS avg_earned_per_user,
    ROUND(AVG(balance), 2) AS avg_balance,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY balance), 2) AS median_balance
FROM user_balance;
```

Выводы:
Средние списания и начисления показывают монетарную активность.
Если медианный баланс ниже среднего – есть небольшое число «богатых» пользователей.
Рекомендация по цене подписки: ориентироваться на средние списания за месяц (например, avg_spent_per_user / 4, если за квартал). При курсе 1 Coin = X руб. получим стоимость.


#### Задание 3
Часть 1: активность
```sql
WITH task_solvers AS (
    SELECT user_id, COUNT(DISTINCT task_id) AS tasks_solved
    FROM codesubmit
    WHERE is_false = 0
    GROUP BY user_id
),
task_attempts AS (
    SELECT user_id, task_id, COUNT(*) AS attempts
    FROM codesubmit
    GROUP BY user_id, task_id
),
test_takers AS (
    SELECT user_id, COUNT(DISTINCT test_id) AS tests_passed
    FROM teststart
    GROUP BY user_id
),
test_attempts AS (
    SELECT user_id, test_id, COUNT(*) AS attempts
    FROM teststart
    GROUP BY user_id, test_id
),
active_users AS (
    SELECT user_id FROM codesubmit
    UNION
    SELECT user_id FROM teststart
),
```
Часть 2: покупки за кодкоины (списания)
```sql
spend_transactions AS (
    SELECT 
        t.user_id,
        tt.name AS trans_name,
        t.amount
    FROM transaction t
    JOIN transactiontype tt ON t.type_id = tt.id
    WHERE tt.type IN (1,23,24,25,26,27,28,30)
)
SELECT 
    (SELECT AVG(tasks_solved) FROM task_solvers) AS avg_tasks_solved,
    (SELECT AVG(tests_passed) FROM test_takers) AS avg_tests_passed,
    (SELECT AVG(attempts) FROM task_attempts) AS avg_attempts_per_task,
    (SELECT AVG(attempts) FROM test_attempts) AS avg_attempts_per_test,
    (SELECT COUNT(*) * 100.0 / (SELECT COUNT(*) FROM users) FROM active_users) AS pct_active_users,
    -- покупки
    (SELECT COUNT(DISTINCT user_id) FROM spend_transactions WHERE trans_name LIKE '%задача%') AS users_opened_tasks,
    (SELECT COUNT(DISTINCT user_id) FROM spend_transactions WHERE trans_name LIKE '%тест%') AS users_opened_tests,
    (SELECT COUNT(DISTINCT user_id) FROM spend_transactions WHERE trans_name LIKE '%подсказк%') AS users_opened_hints,
    (SELECT COUNT(DISTINCT user_id) FROM spend_transactions WHERE trans_name LIKE '%решение%') AS users_opened_solutions,
    (SELECT SUM(amount) FROM spend_transactions WHERE trans_name LIKE '%задача%') AS total_task_opens,
    (SELECT SUM(amount) FROM spend_transactions WHERE trans_name LIKE '%тест%') AS total_test_opens,
    (SELECT SUM(amount) FROM spend_transactions WHERE trans_name LIKE '%подсказк%') AS total_hint_opens,
    (SELECT SUM(amount) FROM spend_transactions WHERE trans_name LIKE '%решение%') AS total_solution_opens,
    (SELECT COUNT(DISTINCT user_id) FROM spend_transactions) AS users_with_any_spend,
    (SELECT COUNT(DISTINCT user_id) FROM transaction) AS users_with_any_transaction;
```

Выводы:
Низкая доля активных пользователей (pct_active_users) говорит о том, что базовый функционал должен оставаться бесплатным.
Если среднее число попыток на задачу >1, то подсказки/решения востребованы – их можно включить в подписку.
Самые популярные покупки за монеты – вероятно, решения и подсказки. Их стоит сделать платными.
