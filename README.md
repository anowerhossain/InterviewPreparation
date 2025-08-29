# InterviewPreparation
Data Analysis

# Section A: SQL

### Q1. Write a SQL query to find the total amount of successful transactions per country for the month of July 2025.

```sql
SELECT u.country, SUM(t.amount) AS total_amount
FROM transactions t
JOIN users u ON t.user_id = u.user_id
WHERE t.status = 'success'
  AND DATE_TRUNC('month', t.transaction_date) = '2025-07-01'
GROUP BY u.country;
```

### Q2. Find the top 3 users who spent the most in July 2025.

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

### Q3. Write a query to calculate the retention rate of users who signed up in June 2025 and made at least one transaction in July 2025.

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


### Q4. SQL to calculate retention: users who made transactions in January AND February.

```sql
WITH jan AS (
  SELECT DISTINCT user_id FROM transactions WHERE DATE_TRUNC('month', date) = '2025-01-01'
),
feb AS (
  SELECT DISTINCT user_id FROM transactions WHERE DATE_TRUNC('month', date) = '2025-02-01'
)
SELECT COUNT(*) AS retained_users
FROM jan
JOIN feb USING (user_id);
```

### Q5. Customers Who Made Transactions Every Month. (Find users who made at least 1 transaction in every month of 2024.)

```sql
SELECT user_id
FROM (
    SELECT user_id, COUNT(DISTINCT DATE_TRUNC('month', txn_date)) AS active_months
    FROM transactions
    WHERE DATE_PART('year', txn_date) = 2024
    GROUP BY user_id
) t
WHERE active_months = 12;
```

### Q6. Write a query to calculate the running total of amount per user ordered by date.

```sql
SELECT 
    user_id,
    txn_date,
    amount,
    SUM(amount) OVER (PARTITION BY user_id ORDER BY txn_date) AS running_total
FROM transactions;
```


# Section B: Statistics & Probability

### Q1. Define p-value and explain its importance in hypothesis testing.


The p-value is the probability of observing test results at least as extreme as the actual results, assuming the null hypothesis is true.
If p < 0.05, results are statistically significant, meaning we reject the null hypothesis.
It helps avoid concluding effects exist when they don’t (Type I error).


### Q2. A/B test: Variant A has 2000 users with 240 conversions. Variant B has 2100 users with 315 conversions. Which variant performs better?

Conversion Rate (A) = 240 / 2000 = 12%
Conversion Rate (B) = 315 / 2100 = 15%

Variant B performs better.
(If tested with a Z-test, the difference would likely be statistically significant.)

### Q3. Explain the difference between correlation and causation with an example.

Correlation: Two variables move together (e.g., ice cream sales and drowning rates rise in summer).
Causation: One variable directly influences another (e.g., smoking causes lung cancer).
Correlation does not imply causation.



# Section C: Python for Data

### Q1. Write a Python script to calculate daily active users (DAU) from a logins.csv file with columns (user_id, login_time).

```python
import pandas as pd  # Import pandas for data manipulation

# Read CSV file, parse 'login_time' as datetime column
df = pd.read_csv("logins.csv", parse_dates=["login_time"])

# Extract only the date (without time) from login_time
df["date"] = df["login_time"].dt.date

# Group by each date and count unique user_ids to get DAU
dau = df.groupby("date")["user_id"].nunique().reset_index(name="DAU")

# Print the resulting Daily Active Users
print(dau)

```


### Q2. Given a Pandas DataFrame df with user_id and signup_date, write code to calculate the number of new users per week.

```python
# Convert signup_date column to datetime
df["signup_date"] = pd.to_datetime(df["signup_date"])

# Convert signup_date into week period, then get the start of each week
df["week"] = df["signup_date"].dt.to_period("W").apply(lambda r: r.start_time)

# Group by week and count unique user_id values = number of new users per week
weekly_new_users = df.groupby("week")["user_id"].nunique().reset_index(name="new_users")

# Print the resulting weekly new users DataFrame
print(weekly_new_users)

```


### Q3. Write a Python function to calculate churn rate given two sets: users at the beginning of the month and users at the end.

```python
# Function to calculate churn rate
def churn_rate(start_users, end_users):
    # Users who were in start but not in end = lost users
    lost_users = len(start_users - end_users)
    
    # Churn rate = lost users divided by total starting users
    return lost_users / len(start_users)

# Example sets of users at start and end of month
start = {"u1","u2","u3","u4","u5"}   # users at beginning
end = {"u2","u3","u5"}               # users at end

# Print the churn rate result
print("Churn Rate:", churn_rate(start, end))
```

### Q4. Python function to detect fraudulent transactions where amount > 3x user’s average.

```python

# Calculate avg transaction per user
user_avg = df.groupby("user_id")["amount"].mean().to_dict()

# Flag fraud if transaction > 3x avg
df["is_fraud"] = df.apply(lambda x: x["amount"] > 3 * user_avg[x["user_id"]], axis=1)

print(df[df["is_fraud"]])

```

### Q5. Calculate Monthly Active Users (MAU) given a logins dataset (user_id, login_time).

```python
df = pd.read_csv("logins.csv", parse_dates=["login_time"])

# Extract year-month
df["month"] = df["login_time"].dt.to_period("M")

# Count unique active users per month
mau = df.groupby("month")["user_id"].nunique().reset_index(name="MAU")

print(mau)

```


### Q6. Write a Python query to calculate the average transaction amount per user from a transactions.csv file with columns (user_id, amount, date).

```pythpn

# Load CSV
df = pd.read_csv("transactions.csv")

# Group by user_id and take average transaction amount
avg_txn = df.groupby("user_id")["amount"].mean().reset_index(name="avg_transaction")

print(avg_txn.head())

```

# Section D: Data Interpretation

### Q1. You are given a chart showing average response time (ART) per agent. Some agents have much higher ART than others. What could be possible reasons?

Agents handling more complex queries take longer.
Some may be less trained or inefficient.
System issues (e.g., slow tools, downtime).
Time zone / shift overlap differences.

### Q2. If customer wait time is much higher during non-business hours, what recommendation would you make?

Introduce a chatbot to handle FAQs off-hours.
Enable agent scheduling / shifts for peak hours.
Set clear SLA expectations with automated responses.


# Section E: Case Study

### Q1. Your company wants to analyze churn. You are given tables users(user_id, signup_date, country) and transactions(transaction_id, user_id, transaction_date). How would you approach churn analysis?

- Define churn: e.g., user inactive for 30/60 days.
- Build active user cohorts (monthly).
- Track last transaction date per user.
- Mark churned users = no activity after threshold.
- Calculate churn rate per cohort, country, signup month.
- Use visualization (cohort retention charts).
- Recommend interventions (e.g., reactivation campaigns).
