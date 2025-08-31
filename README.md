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

### Q7. User Assets & Pareto Analysis

-- users ( user_id, name ) || asset ( asset_id, user_id, asset_name, asset_value )

1. **Calculate the total asset value per user.**  
2. **List each user’s assets in descending order of `asset_value`.**  
3. **For each asset, calculate:**
   - Running total of asset values per user.
   - Running percentage of total assets per user.  
4. **Filter the results:**
   - For each user, include **only the assets that cumulatively account for ~80% of the user’s total asset value**.  
   - Include the asset that first **crosses the 80% threshold**.  
5. **Order the final output** by `user_id` and descending `asset_value`.

```sql
WITH asset_running AS (
    SELECT 
        ur.user_id,
        ur.name,
        ss.asset_name,
        ss.asset_value,
        SUM(ss.asset_value) OVER (PARTITION BY ur.user_id) AS total_value,
        SUM(ss.asset_value) OVER (
            PARTITION BY ur.user_id
            ORDER BY ss.asset_value DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_total,
        (SUM(ss.asset_value) OVER (
            PARTITION BY ur.user_id
            ORDER BY ss.asset_value DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) * 100.0 
         / SUM(ss.asset_value) OVER (PARTITION BY ur.user_id)) AS running_pct,
        ROW_NUMBER() OVER (
            PARTITION BY ur.user_id 
            ORDER BY ss.asset_value DESC
        ) AS rn
    FROM test.asset AS ss
    INNER JOIN test.users AS ur ON ss.user_id = ur.user_id
),
cutoff AS (
    SELECT user_id, MIN(rn) AS cutoff_rn
    FROM asset_running
    WHERE running_pct >= 80
    GROUP BY user_id
)
SELECT ar.*
FROM asset_running ar
JOIN cutoff c ON ar.user_id = c.user_id
WHERE ar.rn <= c.cutoff_rn  
ORDER BY ar.user_id, ar.running_total;
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



# Section F: Python for programming 

### Q1. Problem: Reverse a given string without using built-in reverse().

```python
def reverse_string(s):
    # Using slicing to reverse the string
    return s[::-1]

# Test
print(reverse_string("Python"))  # Output: "nohtyP"
```
- s[::-1] slices the string starting from the end to the beginning, effectively reversing it.

### Q2. Determine if a string reads the same backward. (Palindome word)

```python
def is_palindrome(s):
    s = s.lower().replace(" ", "")  # Normalize string
    return s == s[::-1]

# Test
print(is_palindrome("Race car"))  # Output: True
```

### Q3. Find the Largest Element in a List

```python
def largest_element(arr):
    max_val = arr[0]  # Assume first element is largest
    for num in arr:
        if num > max_val:
            max_val = num
    return max_val

# Test
print(largest_element([3, 5, 7, 2, 8]))  # Output: 8
```

### Q4. Fibonacci Sequence (Iterative)

```python
def fibonacci(n):
    if n <= 0:
        return []
    fib_seq = [0, 1]
    while len(fib_seq) < n:
        fib_seq.append(fib_seq[-1] + fib_seq[-2])
    return fib_seq[:n]

# Test
print(fibonacci(7))  # Output: [0, 1, 1, 2, 3, 5, 8]
```

- Start with [0, 1].
- Append the sum of the last two elements until reaching n terms.


### Q5. Remove Duplicates from a List

```python
def remove_duplicates(lst):
    return list(set(lst))  # Convert to set and back

# Test
print(remove_duplicates([1, 2, 2, 3, 4, 4]))  # Output: [1, 2, 3, 4]
```

###  Q6. Find the Missing Number in an Array

```python
def find_missing(arr):
    n = len(arr) + 1
    total_sum = n * (n + 1) // 2
    return total_sum - sum(arr)

# Test
print(find_missing([1, 2, 4, 5, 6]))  # Output: 3
```

- Use formula for sum of 1 to n: n*(n+1)/2.
- Subtract the actual sum to find missing number.

### Q7.Check if Two Strings are Anagrams.

```python
def are_anagrams(s1, s2):
    return sorted(s1.replace(" ", "").lower()) == sorted(s2.replace(" ", "").lower())

# Test
print(are_anagrams("listen", "silent"))  # Output: True
```

- Sort both strings and compare.
- Ignoring spaces and case ensures proper comparison.

### Q8. Count Frequency of Elements in a List

```python
from collections import Counter

def count_frequency(lst):
    return dict(Counter(lst))

# Test
print(count_frequency([1, 2, 2, 3, 3, 3]))  # Output: {1: 1, 2: 2, 3: 3}
```

### Q9. What is a KeyError?
- A KeyError occurs in Python when you try to access a dictionary key that does not exist.
```python
student_scores = {"Alice": 85, "Bob": 92}

# Accessing a non-existing key
print(student_scores["Charlie"])  # KeyError: 'Charlie'

# The .get() method safely returns None (or a default value you provide) if the key doesn’t exist.

score = student_scores.get("Charlie")  # Returns None instead of error
print(score)  # Output: None

# You can provide a default value
score = student_scores.get("Charlie", 0)
print(score)  # Output: 0

```

### Q10. Shallow Copy vs Deep Copy in Python

- Shallow Copy Changes to mutable objects inside the copy will affect the original.

```python
import copy

original = [[1, 2], [3, 4]]

# Shallow copy
shallow = copy.copy(original)

shallow[0][0] = 99  # Modify inner list
print(original)  # Output: [[99, 2], [3, 4]]
print(shallow)   # Output: [[99, 2], [3, 4]]
```

- Deep copy creates a completely independent copy do not affect the original.

```python
import copy

original = [[1, 2], [3, 4]]

# Deep copy
deep = copy.deepcopy(original)

deep[0][0] = 99  # Modify inner list
print(original)  # Output: [[1, 2], [3, 4]]
print(deep)      # Output: [[99, 2], [3, 4]]
```


### Q11. What are decorators in Python? 

- Decorators in Python are a design pattern that allows you to add new functionality to an existing object without modifying its structure. They are commonly used to extend the behavior of functions or methods.

```python
def my_decorator(func):
    def wrapper():
        print("Something is happening before the function is called.")
        func()
        print("Something is happening after the function is called.")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# Output:
# Something is happening before the function is called.
# Hello!
# Something is happening after the function is called.
```



