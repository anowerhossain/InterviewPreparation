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
         ARRAY_AGG(ss.asset_name) OVER (
            PARTITION BY ur.user_id
            ORDER BY ss.asset_value DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_asset,
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


### Understanding `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` in SQL

`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is a **frame specification** used in window functions like `SUM()`, `AVG()`, etc.

- **`UNBOUNDED PRECEDING`** → start from the very first row of the partition.  
- **`CURRENT ROW`** → stop at the current row.

### Meaning

Together, this frame specification tells SQL:

> For each row, include **all rows from the start of the partition up to the current row** when calculating the window function.

### Q8. Find drivers whose **fuel efficiency** has improved by comparing their average fuel efficiency in the **first half of the year** with the **second half of the year**.

- drivers(driver_id,driver_name) |  trips(trip_id,driver_id,trip_date,distance_km,fuel_consumed)
- Calculate fuel efficiency as distance_km / fuel_consumed for each trip
- First half: January to June, Second half: July to December
- Only include drivers who have trips in both halves of the year
- Calculate the efficiency improvement as (second_half_avg - first_half_avg)
- Round all results to 2 decimal places

```sql
WITH fuel_efficiency AS (
    SELECT *, distance_km / fuel_consumed AS df
    FROM trips
), 
first_half AS (
    SELECT 
        driver_id, 
        AVG(df) AS first_half_avg_raw, 
        ROUND(AVG(df), 2) AS first_half_avg
    FROM fuel_efficiency
    WHERE trip_date >= '2023-01-01'
      AND trip_date < '2023-07-01'
    GROUP BY 1
), 
second_half AS (
    SELECT 
        driver_id, 
        AVG(df) AS second_half_avg_raw, 
        ROUND(AVG(df), 2) AS second_half_avg
    FROM fuel_efficiency
    WHERE trip_date >= '2023-07-01'
      AND trip_date < '2024-01-01'
    GROUP BY 1
)
SELECT 
    fh.driver_id,
    ds.driver_name,
    fh.first_half_avg,
    sh.second_half_avg,
    ROUND(sh.second_half_avg_raw - fh.first_half_avg_raw, 2) AS efficiency_improvement
FROM first_half AS fh
INNER JOIN second_half AS sh 
    ON fh.driver_id = sh.driver_id
INNER JOIN drivers AS ds 
    ON fh.driver_id = ds.driver_id
WHERE ROUND(sh.second_half_avg_raw - fh.first_half_avg_raw, 2) > 0
ORDER BY 
    ROUND(sh.second_half_avg_raw - fh.first_half_avg_raw, 2) DESC,
    ds.driver_name ASC;
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



# Section C: Data Interpretation

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

### Q2. How can you define HVC in your application.
- Total Transaction Value.
- Total Transaction Volume.
- Consistency.
- Diversity of transaction ( P2P, Pay Bill ( Shopping, Subscriptions), Bank Cashin, Card cashin ).
- Fee Generator in terms of revenue generation.
- High Balance Holder.
- Premium Feature Users.
- Early Adopter of New Feature.

### Q3. Factors Causing Churn
- Complex onboarding or KYC → too many steps or document rejections.
- App reliability → downtime, failed transactions, or technical glitches.
- Poor user experience → slow interface, confusing navigation.
- Low rewards or incentives (competitors offering cashback, discounts).
- Delayed refunds or failed dispute handling.
- Slow response times to issues.
- Lack of proactive support when transactions fail.
- Better competitor offers (cashbacks, lower fees, more merchants supported).
- One-time use (e.g., only for signup bonus, then abandoned).
- Not integrated into lifestyle (no recurring bill payments, travel, P2P).

### Q4. Definition of churn users of a system.
- No transaction for X days/weeks (e.g., 60 or 90 days).
- No logins/engagement within a certain period.
- Closed/deleted account.
- Balance withdrawn and never reloaded.

### Q5. What is Cohort analysis
*Retention Over Time*

If you only look at overall active users, it hides patterns. Cohort analysis groups users by their signup or first-transaction date and tracks how long they stay active.
Example:
- Users who joined in January → 60% still active after 3 months.
- Users who joined in June → only 30% active after 3 months.
This shows whether retention is improving or declining with new signups.

*Spot Early Churn Patterns*

- Payment apps often lose users right after the first transaction or bonus offer.
- Cohorts help identify when churn happens (Week 1, Month 1, Month 3).
- This tells you if the problem is onboarding, habit building, or long-term loyalty.

*Measure Impact of Product Changes*
- You can compare cohorts before and after new features (e.g., cashback, QR pay, bill payments).
- If retention improves in later cohorts, you know the feature helped.
- If not, you know changes didn’t address churn.

*Compare Customer Segments*
Cohorts don’t have to be only by signup date — they can also be:
- By acquisition channel (ads, referral, organic).
- By geography (urban vs rural users).
- By transaction type (bill pay first vs P2P transfer first).

### Q6. What RFM Stands and Why It’s Useful in a Payment App
- *R = Recency* → How recently a customer made a transaction.
- *F = Frequency* → How often the customer makes transactions.
- *M = Monetary* → How much money the customer spends (transaction value).

How It Works
Assign scores for each customer- Example (scale 1–5, where 5 = best):
- Recency → Customers who transacted last week get 5, last month get 3, last 6 months get 1.
- Frequency → Customers making 50+ payments/month get 5, 10–20 get 3, <5 get 1.
- Monetary → Customers spending $10,000+/month get 5, $1,000 get 3, <$100 get 1.

*Combine the scores into an RFM score (e.g., R=5, F=4, M=5).*

*Segment users based on RFM scores*
- Champions (555) → Active, frequent, high spenders.
- Loyal Customers (x5x) → Frequent, even if not high spenders.
- Big Spenders (xx5) → High-value but maybe less frequent.
- At Risk (1xx) → Haven’t transacted recently, could churn.
- Dormant (111) → Low recency, frequency, and spend → almost lost.

*Why It’s Useful in a Payment App*
- Identify High-Value Customers (HVCs) → Champions and big spenders.
- Detect Churn Early → Customers with dropping recency/frequency scores.
Targeted Marketing →
- Cashback campaigns for at-risk customers.
- Premium offers for high spenders.
- Referral rewards for loyal customers.
- Prioritize Support → Give faster support to high RFM score users.


### Q7. If payment success rate drops suddenly, how would you investigate?

*Confirm the Drop*
- Compare against historical baseline (hourly/daily success rate).
- Check if the drop is global or specific (bank, merchant, region, channel).

*Segment the Problem*
- By payment channel: card, mobile banking, wallet balance.
- By partner bank/gateway: maybe one partner is failing.
- By customer journey step: login → OTP → payment → confirmation.
- Create monitoring alerts for sudden success rate drops.

*User-related (insufficient balance, wrong PIN)*

### Q8. How would you detect fraudulent transactions in Pathao Pay?

Fraud in payments can be:
- Multiple failed login/payment attempts
- Abnormally high transaction volume in short time
- Transactions from unusual devices/IP locations
- Using stolen cards/accounts
- Bypassing KYC/AML limits

Detection Approaches
- Rule-based checks: e.g., flag if > 5 failed OTP attempts in 10 min.
- Statistical anomaly detection: Compare with user’s historical behavior.


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

##  Interview Questions at pathao
### Analytical Round (q1) How to identify high valued customers ?

### Analytical Round (q2) Product team is concern about high churn rate. What is the formula of churn? And what are the possible reasons of become churn?

### Analytical Round (q3) How you ensure the data quality and integrity of a transaction table (SQL)

### Analytical Round (q4) How to write a complex scanrio SQL? (MCQ) 

### Analytical Round (q5) To findout early churn what appoach you will initiate?

### SQL Round (q1) Find out the userwise first transaction amount avg.

### SQL Round (q2) Find out the user wise assets values that cover their 80% of total asset value.

### SQL Round (q3) app_open -> KYC verified -> bank_cashin -> first_payment. Find out the dropoff % in each stage.

### Head of Growth Interview (q1) What do you mean by transaction in fintech app.

### Head of Growth Interview (q2) A users have both bank and MFS integrate in app he made cashin but from the source is missing from where the cash come in wallet. How do you identify the missing source?

### Head of Growth Interview (q3) Churn rate spike last month compare to previous 3 month. What would be the reason? How do you analysis to findout the root cause?

### Head of Growth Interview (q4) Tell me one and only one metric that will indicate is the company sustrain or bankrupt?

### Head of Growth Interview (q5) If we want to lauch a promo campaign. How do you identify which promo is fit in market? How do you validate that your decision is correct in C-level.

### My questions to Head of Growth (q1) What are the challeges I will face in my applied role?

### My questions to Head of Growth (q1) What are current active user in your fintech app?

### Head of Fintech (q1) Tell me about yourself.

### Head of Fintech (q2) Why you want to move just after 3 month after promoted to Product Manager (Data & Growth).

### Head of Fintech (q3) You have been assigned for 4 task that need 4 man day but for urgency you have to submit them within 2 days, how do you deal that situation.

### Head of Fintech (q4) What the challeges you faced in your current role.

### My Question to Head of Fintech (q1) I see one of your interview in youtube. You worked with young founder for drive success in their startup. Lets say if i want to open a new startup where some competitors are already in the dominating position in the market. How can i establish my startup as a Brand.

### My suggestion to Head of Fintech (q2) I use pathao pay but i see their is no customer support solution in your app and in your website. If customers need any support for any failed transaction or KYC issue or system bug that will increase the number disappointed customer which will increase churn rate. What do you think about that? If your answer is YES. Then I give you information that bKash already used REVE Chat for customer support in their mobile app and website which is white labeled for bkash but developed by REVE Chat. What do you think about that? After that he appriciate for good notice and he amazed to know bkash used REVE Chat with white labeled and ask my visiting card for discuss about this futher. 





