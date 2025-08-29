# InterviewPreparation
Data Analysis

### Section A: SQL

# Q1. Write a SQL query to find the total amount of successful transactions per country for the month of July 2025.

```sql
SELECT u.country, SUM(t.amount) AS total_amount
FROM transactions t
JOIN users u ON t.user_id = u.user_id
WHERE t.status = 'success'
  AND DATE_TRUNC('month', t.transaction_date) = '2025-07-01'
GROUP BY u.country;
```

# Q2. Find the top 3 users who spent the most in July 2025.

```sql
SELECT u.user_id, u.name, SUM(t.amount) AS total_spent
FROM transactions t
JOIN users u ON t.user_id = u.user_id
WHERE t.status = 'success'
  AND DATE_TRUNC('month', t.transaction_date) = '2025-07-01'
GROUP BY u.user_id, u.name
ORDER BY total_spent DESC
LIMIT 3;
```

# Q3. Write a query to calculate the retention rate of users who signed up in June 2025 and made at least one transaction in July 2025.

```sql
WITH june_users AS (
    SELECT user_id 
    FROM users 
    WHERE DATE_TRUNC('month', signup_date) = '2025-06-01'
),
july_active AS (
    SELECT DISTINCT user_id 
    FROM transactions 
    WHERE DATE_TRUNC('month', transaction_date) = '2025-07-01'
)
SELECT 
    COUNT(DISTINCT j.user_id)::DECIMAL / COUNT(DISTINCT u.user_id) * 100 AS retention_rate
FROM june_users u
LEFT JOIN july_active j ON u.user_id = j.user_id;
```



### Section B: Statistics & Probability

# Q1. Define p-value and explain its importance in hypothesis testing.

```text
The p-value is the probability of observing test results at least as extreme as the actual results, assuming the null hypothesis is true.
If p < 0.05, results are statistically significant, meaning we reject the null hypothesis.
It helps avoid concluding effects exist when they don’t (Type I error).
```

# Q2. A/B test: Variant A has 2000 users with 240 conversions. Variant B has 2100 users with 315 conversions. Which variant performs better?

-- Conversion Rate (A) = 240 / 2000 = 12%
-- Conversion Rate (B) = 315 / 2100 = 15%

Variant B performs better.
(If tested with a Z-test, the difference would likely be statistically significant.)

# Q3. Explain the difference between correlation and causation with an example.

Correlation: Two variables move together (e.g., ice cream sales and drowning rates rise in summer).
Causation: One variable directly influences another (e.g., smoking causes lung cancer).
Correlation does not imply causation.


