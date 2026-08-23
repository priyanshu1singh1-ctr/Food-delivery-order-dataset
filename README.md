<img width="720" height="480" alt="image" src="https://github.com/user-attachments/assets/f616545b-ca93-4fee-a4dc-71ce3d1957cd" /># Food-delivery-order-dataset- management 

├── data/
│   ├── raw/Sales_analysis
│   └── cleaned/orders_cleaned
├── scripts/
│   ├── 01_clean_data.py
│   └── 02_run_analysis.py
├── sql/
│   └── analysis_queries.sql
└── images/
    ├── revenue_by_city.png
    ├── late_rate_traffic_weather.png
    ├── orders_by_hour.png
    ├── revenue_by_segment.png
    └── weekly_revenue.png

End-to-end analysis of one month (October 2025) of food-delivery orders — cleaning a raw export, analyzing it with SQL, and turning it into business recommendations.

Skills demonstrated: data cleaning (Python) · SQL analysis · data visualization · business storytelling.


READ ME:

#. Business context
The dataset is an order-level export from a food delivery platform (Uber-Eats / DoorDash style) operating across 4 cities. Each row is one order, with details on the customer, restaurant, driver, pricing, weather/traffic conditions, and delivery timing.

**Questions this analysis answers:**

1. Where is revenue coming from, and how is it trending?
2. Which cities have the best/worst cancellation rates?
3. What's driving late deliveries — and can we predict them?
4. Which customer segment is most valuable?
5. Do experienced delivery partners actually perform better?
6. Does discounting actually grow basket size?

#. Data 
Source file - 	data/raw/Sales_analysis.xlsx
Rows -	649 orders
Period -	October 2025

Data quality issues found and fixed (see scripts/01_clean_data.py):

**tip_amount**, **actual_delivery_time_minutes**, and **late_delivery** were **NULL** for all 14 cancelled orders — expected, since cancelled orders are never delivered. Kept the rows and added an explicit **is_cancelled** flag instead of dropping them, so cancellations can still be analyzed.

Redundant date fields **(order_timestamp** vs **order_date**) collapsed into one source of truth, with clean **order_month** / **order_week** fields derived from it.

**discount_percent** stored as a whole number (10 = 10%) — added **discount_fraction** for rate math.

Text columns stripped of stray whitespace/casing for safe grouping.

Added business fields used throughout the analysis: **revenue** (0 for cancelled orders), d**elivery_delay_minutes** (actual − estimated), **is_late**.

Cleaned output: **data/cleaned/orders_cleaned.csv**


#. Key Findings 
Revenue is concentrated in two cities, and not proportional to order count.
City A has the most orders (240) but City B generates more revenue ($9,159 vs $7,462) on far fewer orders — its average order value ($43.82) is 41% higher than City A's ($31.09).

**Recommendation**: investigate what's different about City B's menu mix or customer base — whatever is driving higher basket size there may be replicable in City A, which has the volume but not the value.

Bad weather + heavy traffic is a near-guarantee of a late delivery
Late-delivery rate climbs from 17–22% in clear weather with low/medium traffic to 75–100% when it's storming, regardless of traffic level.

**Delivery partner experience doesn't reduce lateness**

Late rate is roughly flat across experience buckets (34.8% for under a year, 36.1% for 1–3 years, 30.2% for 3+ years) — a much smaller effect than weather. This suggests late deliveries are driven by external conditions and possibly restaurant prep time, not driver skill, so partner training programs aimed at speed are unlikely to move the needle much on their own.

**Regular customers are the backbone; Premium customers order less often**

"Regular" customers drive the most total revenue ($12,781, 54% of total) simply through volume (331 orders). Interestingly, "Premium"-tier customers have the lowest average order value ($28.28) of the three segments, despite the name — worth checking whether "Premium" reflects a pricing tier or a labeling artifact in the source system.

**Discounts aren't growing basket size**

Discounted orders had essentially the same average subtotal ($29.79) and items per order (2.82) as non-discounted orders ($29.86 / 2.88). At face value, October's discounts subsidized existing behavior rather than driving larger baskets.
Recommendation: test discounts that require a minimum basket size (e.g. "20% off orders over $30") instead of blanket percentage-off codes, to see if that changes the pattern.

**Ordering has two daily peaks — lunch and dinner**

Orders climb steadily from 11am, peak at 1pm (61 orders), dip mid-afternoon, then climb again to a second peak between 6–9pm.


#. Next steps
1. Bring in more months of data to check whether the weekend/weekday and hourly patterns hold, or are specific to October.
2. Join in restaurant-level cost data to move from revenue to margin analysis by city/category.
3. Build a simple logistic model predicting **is_late** from weather, traffic, and hour — the SQL breakdown above suggests weather alone is a strong signal.







ANALSYIS QUERIES:


-- Food Delivery Analytics — Business Analysis Queries
-- Table: orders  (loaded from data/cleaned/orders_cleaned.csv)
-- Run with: sqlite3, or load the CSV into any SQL engine (Postgres/MySQL/BigQuery)


1. Monthly revenue and order volume trend
SELECT
    order_month,
    COUNT(*)                                   AS total_orders,
    SUM(CASE WHEN is_cancelled = 0 THEN 1 ELSE 0 END) AS completed_orders,
    ROUND(SUM(revenue), 2)                     AS total_revenue,
    ROUND(AVG(revenue), 2)                     AS avg_order_value
FROM orders
GROUP BY order_month
ORDER BY order_month;

2. Revenue and order share by city
SELECT
    city,
    COUNT(*) AS orders,
    ROUND(SUM(revenue), 2) AS revenue,
    ROUND(100.0 * SUM(revenue) / (SELECT SUM(revenue) FROM orders), 1) AS pct_of_revenue,
    ROUND(AVG(revenue), 2) AS avg_order_value
FROM orders
GROUP BY city
ORDER BY revenue DESC;

3. Cancellation rate by city
SELECT
    city,
    COUNT(*) AS total_orders,
    SUM(is_cancelled) AS cancelled_orders,
    ROUND(100.0 * SUM(is_cancelled) / COUNT(*), 2) AS cancellation_rate_pct
FROM orders
GROUP BY city
ORDER BY cancellation_rate_pct DESC;

4. Late-delivery rate and average delay by traffic level & weather
SELECT
    traffic_level,
    weather,
    COUNT(*) AS orders,
    ROUND(100.0 * SUM(is_late) / COUNT(*), 1) AS late_rate_pct,
    ROUND(AVG(delivery_delay_minutes), 1) AS avg_delay_minutes
FROM orders
WHERE is_cancelled = 0
GROUP BY traffic_level, weather
ORDER BY late_rate_pct DESC;

5. Customer segment value: New vs Regular vs Premium
SELECT
    customer_type,
    COUNT(DISTINCT customer_id) AS customers,
    COUNT(*) AS orders,
    ROUND(SUM(revenue), 2) AS revenue,
    ROUND(AVG(revenue), 2) AS avg_order_value,
    ROUND(AVG(tip_amount), 2) AS avg_tip
FROM orders
GROUP BY customer_type
ORDER BY revenue DESC;

6. Top restaurant categories by revenue and rating
SELECT
    restaurant_primary_category,
    COUNT(*) AS orders,
    ROUND(SUM(revenue), 2) AS revenue,
    ROUND(AVG(restaurant_rating), 2) AS avg_restaurant_rating
FROM orders
GROUP BY restaurant_primary_category
ORDER BY revenue DESC;

7. Peak ordering hours (for staffing / driver allocation decisions)
SELECT
    order_hour,
    COUNT(*) AS orders,
    ROUND(AVG(actual_delivery_time_minutes), 1) AS avg_delivery_time_minutes
FROM orders
WHERE is_cancelled = 0
GROUP BY order_hour
ORDER BY order_hour;

8. Does delivery partner experience improve on-time performance?
SELECT
    CASE
        WHEN delivery_partner_experience_months < 12 THEN '0-11 months'
        WHEN delivery_partner_experience_months < 36 THEN '12-35 months'
        ELSE '36+ months'
    END AS experience_bucket,
    COUNT(*) AS deliveries,
    ROUND(100.0 * SUM(is_late) / COUNT(*), 1) AS late_rate_pct,
    ROUND(AVG(delivery_partner_rating), 2) AS avg_partner_rating
FROM orders
WHERE is_cancelled = 0
GROUP BY experience_bucket
ORDER BY late_rate_pct DESC;

9. Discount effectiveness: does discounting correlate with larger baskets?
SELECT
    CASE WHEN discount_percent = 0 THEN 'No discount' ELSE 'Discounted' END AS discount_group,
    COUNT(*) AS orders,
    ROUND(AVG(subtotal), 2) AS avg_subtotal,
    ROUND(AVG(items_count), 2) AS avg_items_per_order
FROM orders
GROUP BY discount_group;

10. Weekend vs weekday performance
SELECT
    CASE WHEN is_weekend = 1 THEN 'Weekend' ELSE 'Weekday' END AS day_type,
    COUNT(*) AS orders,
    ROUND(AVG(revenue), 2) AS avg_order_value,
    ROUND(100.0 * SUM(is_late) / SUM(CASE WHEN is_cancelled = 0 THEN 1 ELSE 0 END), 1) AS late_rate_pct
FROM orders
GROUP BY day_type;




REVENUE BY CITY:
<img width="720" height="480" alt="image" src="https://github.com/user-attachments/assets/7d2cc316-9d08-4d62-8485-a2f0c42c1f9f" />





LATE RATE TRAFFIC WEATHER:
<img width="840" height="480" alt="image" src="https://github.com/user-attachments/assets/da3e6c64-6f07-40fc-93dd-0cff22e07e3a" />










