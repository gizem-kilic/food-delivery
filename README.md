
## 🛒 Food Delivery Customer Segmentation & Insights

This project explores customer behavior based on a food delivery service's user data collected from September 2019 through October 2020. The dataset contains one row per user and includes details like purchase frequency, average order value, and registration metadata. The analysis focuses on creating strategic customer segments and uncovering behavioral patterns to drive better business decisions.


## 📁 Dataset Source

This dataset was anonymously published online to be able to be used for similar projects.


📊 **Dashboard:**  
[View on Tableau Public](https://public.tableau.com/views/CustomerSegmentationandStrategicInsights-FoodDeliveryExample/Dashboard1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

---
![Food Delivery](Online-Food-Delivery.jpg)

- Tools Used: PostgreSQL and Tableau 


## 📌 Project Goals

- Segment users based on spending and frequency
- Identify high-value customers and markets
- Highlight churn risks
- Recommend strategic actions for different segments

---

## 🧪 Exploratory Data Analysis (EDA)

### Missing Values

```sql
SELECT 
  COUNT(*) FILTER (WHERE registration_country IS NULL) AS null_country,
  COUNT(*) FILTER (WHERE preferred_device IS NULL) AS null_device,
  COUNT(*) FILTER (WHERE total_purchases_eur IS NULL) AS null_total_purchases
FROM user_purchases;
```

- 73 users have no recorded preferred device
- 9955 users made no purchases between September 2019 and October 2020

---

### Top 10 Countries by User Registration

```sql
SELECT registration_country, COUNT(*) AS user_count
FROM user_purchases
GROUP BY registration_country
ORDER BY user_count DESC
LIMIT 10;
```

---

### Purchase Behavior Summary

```sql
SELECT 
  AVG(total_purchases_eur) AS avg_spend,
  MIN(total_purchases_eur) AS min_spend,
  MAX(total_purchases_eur) AS max_spend,
  AVG(purchase_count) AS avg_purchase_count
FROM user_purchases;
```

- Avg Spend: €176.21
- Min Spend: €1.01
- Max Spend: €7979.62
- Avg Purchase Count: 3.35

---

## 🔍 Customer Segmentation

### Step 1: Compute Aggregates by Country

```sql
SELECT 
    REGISTRATION_COUNTRY,
    AVG(PURCHASE_COUNT) AS avg_purchase_count,
    AVG(TOTAL_PURCHASES_EUR) AS avg_total_spending,
    AVG(TOTAL_PURCHASES_EUR) / AVG(PURCHASE_COUNT) AS avg_purchase_value
FROM user_purchases
GROUP BY REGISTRATION_COUNTRY
HAVING AVG(PURCHASE_COUNT) >= 1
ORDER BY avg_total_spending DESC;
```

---

### Step 2: Compute Percentiles for Segmentation

```sql
WITH active_users AS (
  SELECT *
  FROM food_delivery_data
  WHERE PURCHASE_COUNT > 0
)

SELECT
  percentile_cont(0.25) WITHIN GROUP (ORDER BY PURCHASE_COUNT) AS p25_purchase_count,
  percentile_cont(0.50) WITHIN GROUP (ORDER BY PURCHASE_COUNT) AS p50_purchase_count,
  percentile_cont(0.75) WITHIN GROUP (ORDER BY PURCHASE_COUNT) AS p75_purchase_count,

  percentile_cont(0.25) WITHIN GROUP (ORDER BY AVG_PURCHASE_VALUE_EUR) AS p25_avg_order_value,
  percentile_cont(0.50) WITHIN GROUP (ORDER BY AVG_PURCHASE_VALUE_EUR) AS p50_avg_order_value,
  percentile_cont(0.75) WITHIN GROUP (ORDER BY AVG_PURCHASE_VALUE_EUR) AS p75_avg_order_value
FROM active_users;
```

Sample output:
- `p25_purchase_count = 1`
- `p75_avg_order_value = 38.46`

---

### Step 3: Assign Customer Segments

```sql
SELECT *,
  CASE
    WHEN PURCHASE_COUNT >= 6 AND AVG_PURCHASE_VALUE_EUR >= 38.46 THEN 'Power User'
    WHEN PURCHASE_COUNT >= 6 AND AVG_PURCHASE_VALUE_EUR < 19.23 THEN 'Frequent Low Spender'
    WHEN PURCHASE_COUNT < 3 AND AVG_PURCHASE_VALUE_EUR >= 38.46 THEN 'Occasional Big Spender'
    WHEN PURCHASE_COUNT < 3 AND AVG_PURCHASE_VALUE_EUR < 19.23 THEN 'Budget User'
    ELSE 'Mid-Tier User'
  END AS customer_segment
FROM user_purchases
WHERE PURCHASE_COUNT > 0;
```

---

## 📈 Strategic Analysis

### 1. Discovery — High-Value Users

#### High-Value Customers per Country

```sql
SELECT registration_country, 
       COUNT(user_id) AS user_count,
       ROUND(SUM(total_purchases_eur), 2) AS total_revenue,
       ROUND(AVG(total_purchases_eur), 2) AS avg_revenue_per_user
FROM customersegment
GROUP BY registration_country
ORDER BY total_revenue DESC
LIMIT 10;
```



#### Top Country x Segment Combinations

```sql
WITH top_countries AS (
  SELECT registration_country
  FROM customersegment
  GROUP BY registration_country
  ORDER BY SUM(total_purchases_eur) DESC
  LIMIT 10
)

SELECT
  cs.registration_country,
  cs.customer_segment,
  COUNT(cs.user_id) AS user_count,
  SUM(cs.total_purchases_eur) AS total_revenue
FROM customersegment cs
JOIN top_countries tc
  ON cs.registration_country = tc.registration_country
GROUP BY cs.registration_country, cs.customer_segment
ORDER BY cs.registration_country, total_revenue DESC;
```

---

### 2. Usage Patterns — How Do Users Behave?

#### Preferred Devices per Segment

```sql
SELECT 
    customer_segment,
    preferred_device,
    COUNT(*) AS user_count
FROM customersegment
GROUP BY customer_segment, preferred_device
ORDER BY customer_segment, user_count DESC;
```

#### Preferred Day of the Week per Segment
(Numbers represent the days in a weekly order, i.e 1=Monday)
```sql
SELECT 
    customer_segment,
    most_common_weekday_to_purchase AS purchase_day,
    COUNT(*) AS user_count
FROM customersegment
GROUP BY customer_segment, most_common_weekday_to_purchase
ORDER BY customer_segment, user_count DESC;
```

#### Preferred Hour of the Day per Segment

```sql
SELECT 
    customer_segment,
    most_common_hour_of_the_day_to_purchase AS purchase_hour,
    COUNT(*) AS user_count
FROM customersegment
GROUP BY customer_segment, most_common_hour_of_the_day_to_purchase
ORDER BY customer_segment, user_count DESC;
```

#### Favorite Meals per Segment

```sql
SELECT
  customer_segment,
  AVG(breakfast_purchases) AS avg_breakfast_orders,
  AVG(lunch_purchases) AS avg_lunch_orders,
  AVG(evening_purchases) AS avg_evening_orders,
  AVG(dinner_purchases) AS avg_dinner_orders,
  AVG(late_night_purchases) AS avg_late_night_orders
FROM customersegment
GROUP BY customer_segment
ORDER BY customer_segment;
```

---

### 3. Retention Risk — Who Might We Lose?

#### Churn Risk per Segment

```sql
SELECT 
    customer_segment,
    COUNT(*) AS user_count,
    ROUND(SUM(total_purchases_eur), 2) AS total_revenue,
    ROUND(AVG(total_purchases_eur), 2) AS avg_revenue_per_user,

    COUNT(
        CASE 
            WHEN avg_days_between_purchases IS NOT NULL 
              AND DATE '2020-10-01' - last_purchase_day > (2 * avg_days_between_purchases)
            THEN 1
        END
    ) AS churn_risk_users,

    ROUND(
        100.0 * COUNT(
            CASE 
                WHEN avg_days_between_purchases IS NOT NULL 
                  AND DATE '2020-10-01' - last_purchase_day > (2 * avg_days_between_purchases)
                THEN 1
            END
        ) / COUNT(*), 
    2) AS churn_risk_rate_percent
FROM customersegment
GROUP BY customer_segment
ORDER BY total_revenue DESC;
```

#### High-Value Customers per Segment

```sql
SELECT 
    customer_segment,
    COUNT(*) AS user_count,
    ROUND(SUM(total_purchases_eur), 2) AS total_revenue,
    ROUND(AVG(total_purchases_eur), 2) AS avg_revenue_per_user
FROM customersegment
GROUP BY customer_segment
ORDER BY total_revenue DESC;
```

---

## ✅ Important Insights

**1. Power Users Drive Majority of Revenue**  
Power Users (≥ 6 orders & high avg. order value) contribute significantly to total revenue, despite being fewer in number. These are key targets for loyalty programs.

**2. Most Registered Users Made No Purchases**  
~9955 users didn’t place any order during the observed period. This highlights a potential gap in activation or onboarding.

**3. Preferred Devices Vary by Segment**  
The most valuable users tend to favor Android, but Mid-Tier and Budget users are more distributed across iOS, Android, and Web.

**4. Weekday Behavior Differs by Segment**  
Power Users typically order more during weekdays like Tuesday and Thursday, while Budget Users have no clear pattern.

**5. Favorite Meals by Segment**  
Dinner and Evening meals dominate for high spenders, while Budget Users show a spread across Lunch and Breakfast too.

**6. Churn Risk Exists Even Among High Revenue Segments**  
Although Power Users generate high revenue, they still have a measurable churn risk. Monitoring their activity is crucial

---

## 🛠️ Future Improvements

- **Filter dashboards by country** to enable localized insights.
- Incorporate **temporal trends** and **seasonal patterns** into churn modeling.

