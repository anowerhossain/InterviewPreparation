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



