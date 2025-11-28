# **A/B Testing & Funnel Analysis - SQL + Python**

### **Project Repository**

This project performs end-to-end **A/B Testing**, **conversion funnel analysis**, and **revenue leakage estimation** using **SQL (MySQL)** and **Python (Pandas, SciPy, Matplotlib)**.

# 

# **Problem Statement**

The goal of this project was to evaluate the performance of two UI variants (**A** and **B**) on user engagement and conversion behavior.

Key questions:

* Does Variant B increase CTR significantly compared to Variant A?
* Where do users drop off in the funnel (Page → Click → Cart → Purchase)?
* How much revenue is lost due to conversion leakage at the Add-to-Cart stage?
* How can SQL + Python be used to build a reproducible experiment analysis pipeline?

This project simulates user behavior for **10,000 users**, assigns them experiment variants, tracks multiple event types, and processes results using statistical tests and visualizations.

# 

# **Dataset & Experiment Structure**

* **10,000 simulated users**
* Assigned **50/50** to Variant A and B
* Events captured:

  * `page_view`
  * `click`
  * `add_to_cart`
  * `purchase`
* Revenue is assigned only during purchase events.

# 

# **Steps Followed**

### **Step 1 - Database Setup (MySQL)**

* Dropped existing DB and created a fresh database:

```sql
DROP DATABASE IF EXISTS ab_testing_db;
CREATE DATABASE ab_testing_db;
USE ab_testing_db;
```

* Created tables:
  **users**, **experiments**, **events**.

* Built a **numbers table (0-9999)** using cross joins to auto-generate 10,000 users.

# 

### **Step 2 - User Creation**

* Users inserted with:

  * Cyclic signup dates
  * Country distribution → US, UK, IN, CA, AU

#

### **Step 3 - Variant Assignment (A/B)**

* 50/50 split using:

```sql
CASE WHEN user_id % 2 = 0 THEN 'A' ELSE 'B' END
```

#

### **Step 4 - Event Generation**

Created **four levels of events**:

1. **Page View** - all users

2. **Click Events** - variant-controlled CTR

   * Variant A: **~3.1% CTR**
   * Variant B: **~3.3% CTR**

3. **Add to Cart** - 72% of clickers

4. **Purchase** - 50% of cart users

   * Purchase revenue: random $10-$110

#

### **Step 5 - Data Quality Validation**

Queried:

* Total users = **10,000**
* Variant distribution = 5,000 each
* Event type counts and order

#

### **Step 6 - SQL Analysis**

1. **CTR analysis using user-level deduplication**
   (Ensures every user counted *once*, even with multiple events)

2. **Funnel analysis**

   * Page View → Click → Add to Cart → Purchase
   * Drop-off % at each stage

3. **Revenue analysis**

   * Total revenue by variant
   * AOV (Avg Order Value)
   * Revenue per user

#

### **Step 7 - Python Statistical Testing**  

Using **SciPy**, **Pandas**, and **NumPy**, the following analyses were performed:

* Loaded SQL-exported CSVs (`ctr_analysis.csv`, `funnel_analysis.csv`, `revenue_analysis.csv`)
* Performed **Z-test for two proportions**

  ```python
  z_stat, p_value = z_test_proportions(n_a, ctr_a, n_b, ctr_b)
  ```
* Calculated:

  * **Absolute CTR lift**
  * **Relative lift**
  * **Significance (p-value < 0.05)**

#

### **Step 8 - Revenue Leakage Analysis**

Estimated revenue loss due to drop-offs at Add-to-Cart stage.

* Funnel drop-off computed from means of both variants
* Potential recovered revenue if drop-off reduced by **50%**
* Revenue leakage %:

  ```python
  revenue_leakage_pct = (potential_revenue / current_revenue) * 100
  ```

#

### **Step 9 - Visualization (Python)**

Generated 4-panel visualization:

* CTR comparison
* Funnel comparison
* Drop-off rate chart
* Revenue per user

Saved as: **ab_test_analysis.png**

*(See image below)*
![Visualization](attachment\:ab_test_analysis.png)

#

# **SQL Queries (Complete Workflow)**

All SQL queries used in this project are included below for reproducibility:

<details>
<summary><b>Click to expand SQL code</b></summary>

```sql
-- Drop the entire database and recreate it
DROP DATABASE IF EXISTS ab_testing_db;
CREATE DATABASE ab_testing_db;
USE ab_testing_db;

-- Users table
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    signup_date DATE,
    country VARCHAR(50)
);

-- Experiments table (variant assignment)
CREATE TABLE experiments (
    user_id INT,
    variant VARCHAR(1),
    assignment_date DATE,
    PRIMARY KEY (user_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Events table (user actions)
CREATE TABLE events (
    event_id INT PRIMARY KEY,
    user_id INT,
    event_type VARCHAR(50),
    event_timestamp DATETIME,
    product_id INT,
    revenue DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- Step 1: Create a numbers table with 10,000 rows
-- This is a one-time helper table
DROP TABLE IF EXISTS numbers_temp;
CREATE TABLE numbers_temp (n INT PRIMARY KEY);

-- Insert base 10 numbers
INSERT INTO numbers_temp (n) VALUES 
(0),(1),(2),(3),(4),(5),(6),(7),(8),(9);

-- Build up to 10,000 using cross joins
INSERT INTO numbers_temp (n)
SELECT a.n + b.n*10 + c.n*100 + d.n*1000
FROM numbers_temp a
CROSS JOIN numbers_temp b
CROSS JOIN numbers_temp c
CROSS JOIN numbers_temp d
WHERE a.n + b.n*10 + c.n*100 + d.n*1000 BETWEEN 10 AND 9999;

-- Now insert users (1 to 10000)
INSERT INTO users (user_id, signup_date, country)
SELECT 
    n + 1 as user_id,
    DATE_ADD('2024-01-01', INTERVAL ((n + 1) % 365) DAY),
    CASE ((n + 1) % 5)
        WHEN 0 THEN 'US'
        WHEN 1 THEN 'UK'
        WHEN 2 THEN 'IN'
        WHEN 3 THEN 'CA'
        ELSE 'AU'
    END
FROM numbers_temp
WHERE n < 10000;

-- Step 2: Assign users to variants (50/50 split)
INSERT INTO experiments (user_id, variant, assignment_date)
SELECT 
    user_id,
    CASE WHEN user_id % 2 = 0 THEN 'A' ELSE 'B' END,
    '2024-01-15'
FROM users;

-- Step 3: Generate page view events (all users view a product page)
INSERT INTO events (event_id, user_id, event_type, event_timestamp, product_id, revenue)
SELECT 
    user_id,
    user_id,
    'page_view',
    DATE_ADD('2024-01-16 10:00:00', INTERVAL (user_id % 24) HOUR),
    (user_id % 100) + 1,
    NULL
FROM users;

-- Step 4: Generate click events 
-- Variant A: ~3.1% CTR, Variant B: ~3.3% CTR
INSERT INTO events (event_id, user_id, event_type, event_timestamp, product_id, revenue)
SELECT 
    10000 + u.user_id,
    u.user_id,
    'click',
    DATE_ADD('2024-01-16 10:05:00', INTERVAL (u.user_id % 24) HOUR),
    (u.user_id % 100) + 1,
    NULL
FROM users u
JOIN experiments e ON u.user_id = e.user_id
WHERE (e.variant = 'A' AND (u.user_id * 17) % 1000 < 31)  -- 3.1% for A
   OR (e.variant = 'B' AND (u.user_id * 17) % 1000 < 33); -- 3.3% for B

-- Step 5: Generate add-to-cart events (72% of clicks proceed)
INSERT INTO events (event_id, user_id, event_type, event_timestamp, product_id, revenue)
SELECT 
    20000 + user_id,
    user_id,
    'add_to_cart',
    DATE_ADD('2024-01-16 10:10:00', INTERVAL (user_id % 24) HOUR),
    (user_id % 100) + 1,
    NULL
FROM events
WHERE event_type = 'click' AND (user_id * 23) % 100 < 72;

-- Step 6: Generate purchase events (50% of add-to-cart complete purchase)
INSERT INTO events (event_id, user_id, event_type, event_timestamp, product_id, revenue)
SELECT 
    30000 + user_id,
    user_id,
    'purchase',
    DATE_ADD('2024-01-16 10:15:00', INTERVAL (user_id % 24) HOUR),
    (user_id % 100) + 1,
    ROUND(10 + (RAND() * 100), 2)  -- Revenue between $10-$110
FROM events
WHERE event_type = 'add_to_cart' AND (user_id * 31) % 100 < 50;

-- Clean up helper table
DROP TABLE numbers_temp;

-- Check users count
SELECT COUNT(*) as total_users FROM users;
-- Expected: 10000

-- Check experiments count
SELECT COUNT(*) as total_assignments FROM experiments;
-- Expected: 10000

-- Check variant distribution
SELECT variant, COUNT(*) as count 
FROM experiments 
GROUP BY variant;
-- Expected: A = 5000, B = 5000

-- Check events count
SELECT COUNT(*) as total_events FROM events;
-- Expected: ~11,500

-- Check event types breakdown
SELECT event_type, COUNT(*) as count 
FROM events 
GROUP BY event_type 
ORDER BY 
    CASE event_type
        WHEN 'page_view' THEN 1
        WHEN 'click' THEN 2
        WHEN 'add_to_cart' THEN 3
        WHEN 'purchase' THEN 4
    END;

-- A/B Test CTR Analysis with User-Level Deduplication
WITH user_page_views AS (
    SELECT DISTINCT
        e.user_id,
        ex.variant
    FROM events e
    JOIN experiments ex ON e.user_id = ex.user_id
    WHERE e.event_type = 'page_view'
),
user_clicks AS (
    SELECT DISTINCT
        e.user_id,
        ex.variant
    FROM events e
    JOIN experiments ex ON e.user_id = ex.user_id
    WHERE e.event_type = 'click'
),
variant_stats AS (
    SELECT 
        v.variant,
        COUNT(DISTINCT v.user_id) AS total_users,
        COUNT(DISTINCT c.user_id) AS clicked_users,
        ROUND(100.0 * COUNT(DISTINCT c.user_id) / COUNT(DISTINCT v.user_id), 2) AS ctr_percent
    FROM user_page_views v
    LEFT JOIN user_clicks c ON v.user_id = c.user_id AND v.variant = c.variant
    GROUP BY v.variant
)
SELECT 
    variant,
    total_users,
    clicked_users,
    ctr_percent
FROM variant_stats
ORDER BY variant;


-- Funnel Analysis with Drop-off Rates
WITH funnel_data AS (
    SELECT 
        ex.variant,
        COUNT(DISTINCT CASE WHEN e.event_type = 'page_view' THEN e.user_id END) AS step1_page_view,
        COUNT(DISTINCT CASE WHEN e.event_type = 'click' THEN e.user_id END) AS step2_click,
        COUNT(DISTINCT CASE WHEN e.event_type = 'add_to_cart' THEN e.user_id END) AS step3_add_to_cart,
        COUNT(DISTINCT CASE WHEN e.event_type = 'purchase' THEN e.user_id END) AS step4_purchase
    FROM events e
    JOIN experiments ex ON e.user_id = ex.user_id
    GROUP BY ex.variant
)
SELECT 
    variant,
    step1_page_view,
    step2_click,
    ROUND(100.0 * step2_click / step1_page_view, 2) AS click_rate,
    ROUND(100.0 * (step1_page_view - step2_click) / step1_page_view, 2) AS drop_page_to_click,
    step3_add_to_cart,
    ROUND(100.0 * step3_add_to_cart / step2_click, 2) AS add_to_cart_rate,
    ROUND(100.0 * (step2_click - step3_add_to_cart) / step2_click, 2) AS drop_click_to_cart,
    step4_purchase,
    ROUND(100.0 * step4_purchase / step3_add_to_cart, 2) AS purchase_rate,
    ROUND(100.0 * (step3_add_to_cart - step4_purchase) / step3_add_to_cart, 2) AS drop_cart_to_purchase
FROM funnel_data
ORDER BY variant;


-- Revenue Analysis by Variant
SELECT 
    ex.variant,
    COUNT(DISTINCT e.user_id) AS purchasing_users,
    ROUND(SUM(e.revenue), 2) AS total_revenue,
    ROUND(AVG(e.revenue), 2) AS avg_order_value,
    ROUND(SUM(e.revenue) / 5000, 2) AS revenue_per_user
FROM events e
JOIN experiments ex ON e.user_id = ex.user_id
WHERE e.event_type = 'purchase'
GROUP BY ex.variant;

```

</details>

#

# **Python Code (Complete Notebook)**  

Source file: *AB_Test_Statistical_Analysis.ipynb*
Files to be uploaded: 
<details>
<summary><b>Click to expand Python code</b></summary>

```python
import pandas as pd
import numpy as np
from scipy import stats
import matplotlib.pyplot as plt
import seaborn as sns

# Set display options
pd.set_option('display.max_columns', None)
pd.set_option('display.width', None)
sns.set_style('whitegrid')

# Load data
ctr_df = pd.read_csv('ctr_analysis.csv')
funnel_df = pd.read_csv('funnel_analysis.csv')
revenue_df = pd.read_csv('revenue_analysis.csv')

print("=== CTR Analysis ===")
print(ctr_df)
print("\n=== Funnel Analysis ===")
print(funnel_df)
print("\n=== Revenue Analysis ===")
print(revenue_df)

# Z-test for CTR difference
def z_test_proportions(n1, p1, n2, p2):
    """
    Perform z-test for two proportions
    n1, n2: sample sizes
    p1, p2: proportions (as decimals)
    """
    # Pooled proportion
    p_pool = (n1 * p1 + n2 * p2) / (n1 + n2)
    
    # Standard error
    se = np.sqrt(p_pool * (1 - p_pool) * (1/n1 + 1/n2))
    
    # Z-statistic
    z = (p1 - p2) / se
    
    # Two-tailed p-value
    p_value = 2 * (1 - stats.norm.cdf(abs(z)))
    
    return z, p_value

# Extract data for variants A and B
variant_a = ctr_df[ctr_df['variant'] == 'A'].iloc[0]
variant_b = ctr_df[ctr_df['variant'] == 'B'].iloc[0]

n_a = variant_a['total_users']
clicks_a = variant_a['clicked_users']
ctr_a = clicks_a / n_a

n_b = variant_b['total_users']
clicks_b = variant_b['clicked_users']
ctr_b = clicks_b / n_b

# Perform z-test
z_stat, p_value = z_test_proportions(n_a, ctr_a, n_b, ctr_b)

print("=== Statistical Significance Test ===")
print(f"Variant A CTR: {ctr_a*100:.2f}% (n={n_a})")
print(f"Variant B CTR: {ctr_b*100:.2f}% (n={n_b})")
print(f"\nAbsolute Lift: {(ctr_b - ctr_a)*100:.2f} percentage points")
print(f"Relative Lift: {((ctr_b/ctr_a - 1)*100):.2f}%")
print(f"\nZ-statistic: {z_stat:.4f}")
print(f"P-value: {p_value:.4f}")
print(f"\nResult: {'SIGNIFICANT' if p_value < 0.05 else 'NOT SIGNIFICANT'} at α=0.05")

# Revenue leakage calculation
funnel_combined = funnel_df.mean(numeric_only=True)  # Average across variants

# Drop-off at Add-to-Cart stage
drop_rate_cart = funnel_combined['drop_click_to_cart'] / 100
print(f"=== Revenue Leakage Analysis ===")
print(f"Drop-off rate at Add-to-Cart: {drop_rate_cart*100:.1f}%")

# Estimate revenue if drop-off was reduced by 50%
avg_revenue_per_user = revenue_df['revenue_per_user'].mean()
total_users = ctr_df['total_users'].sum()
current_revenue = revenue_df['total_revenue'].sum()

# If we recovered 50% of the drop-off users
recovered_users = (funnel_combined['step2_click'] * drop_rate_cart * 0.5) * 2  # *2 for both variants
potential_revenue = recovered_users * avg_revenue_per_user
revenue_leakage_pct = (potential_revenue / current_revenue) * 100

print(f"\nCurrent Total Revenue: ${current_revenue:,.2f}")
print(f"Potential Revenue from 50% Drop-off Recovery: ${potential_revenue:,.2f}")
print(f"Estimated Revenue Leakage: {revenue_leakage_pct:.1f}%")

# Create visualizations
fig, axes = plt.subplots(2, 2, figsize=(14, 10))

# 1. CTR Comparison
ax1 = axes[0, 0]
variants = ctr_df['variant']
ctrs = ctr_df['ctr_percent']
colors = ['#3498db', '#e74c3c']
bars = ax1.bar(variants, ctrs, color=colors, alpha=0.7, edgecolor='black')
ax1.set_ylabel('CTR (%)', fontsize=12, fontweight='bold')
ax1.set_title('A/B Test: Click-Through Rate Comparison', fontsize=14, fontweight='bold')
ax1.set_ylim(0, max(ctrs) * 1.2)
for i, bar in enumerate(bars):
    height = bar.get_height()
    ax1.text(bar.get_x() + bar.get_width()/2., height,
             f'{height:.2f}%', ha='center', va='bottom', fontweight='bold')

# 2. Funnel Visualization
ax2 = axes[0, 1]
stages = ['Page View', 'Click', 'Add to Cart', 'Purchase']
variant_a_funnel = funnel_df[funnel_df['variant'] == 'A'].iloc[0]
variant_b_funnel = funnel_df[funnel_df['variant'] == 'B'].iloc[0]

a_values = [variant_a_funnel['step1_page_view'], variant_a_funnel['step2_click'], 
            variant_a_funnel['step3_add_to_cart'], variant_a_funnel['step4_purchase']]
b_values = [variant_b_funnel['step1_page_view'], variant_b_funnel['step2_click'], 
            variant_b_funnel['step3_add_to_cart'], variant_b_funnel['step4_purchase']]

x = np.arange(len(stages))
width = 0.35
ax2.bar(x - width/2, a_values, width, label='Variant A', color='#3498db', alpha=0.7)
ax2.bar(x + width/2, b_values, width, label='Variant B', color='#e74c3c', alpha=0.7)
ax2.set_xlabel('Funnel Stage', fontsize=12, fontweight='bold')
ax2.set_ylabel('Users', fontsize=12, fontweight='bold')
ax2.set_title('Conversion Funnel by Variant', fontsize=14, fontweight='bold')
ax2.set_xticks(x)
ax2.set_xticklabels(stages, rotation=15, ha='right')
ax2.legend()

# 3. Drop-off rates
ax3 = axes[1, 0]
dropoff_stages = ['Page→Click', 'Click→Cart', 'Cart→Purchase']
dropoff_a = [variant_a_funnel['drop_page_to_click'], 
             variant_a_funnel['drop_click_to_cart'],
             variant_a_funnel['drop_cart_to_purchase']]
dropoff_b = [variant_b_funnel['drop_page_to_click'], 
             variant_b_funnel['drop_click_to_cart'],
             variant_b_funnel['drop_cart_to_purchase']]

x = np.arange(len(dropoff_stages))
ax3.bar(x - width/2, dropoff_a, width, label='Variant A', color='#3498db', alpha=0.7)
ax3.bar(x + width/2, dropoff_b, width, label='Variant B', color='#e74c3c', alpha=0.7)
ax3.set_xlabel('Transition', fontsize=12, fontweight='bold')
ax3.set_ylabel('Drop-off Rate (%)', fontsize=12, fontweight='bold')
ax3.set_title('Drop-off Rates at Each Stage', fontsize=14, fontweight='bold')
ax3.set_xticks(x)
ax3.set_xticklabels(dropoff_stages)
ax3.legend()
ax3.axhline(y=28, color='red', linestyle='--', label='28% Threshold', alpha=0.5)

# 4. Revenue comparison
ax4 = axes[1, 1]
variants_rev = revenue_df['variant']
revenue_per_user = revenue_df['revenue_per_user']
bars = ax4.bar(variants_rev, revenue_per_user, color=colors, alpha=0.7, edgecolor='black')
ax4.set_ylabel('Revenue per User ($)', fontsize=12, fontweight='bold')
ax4.set_title('Revenue per User by Variant', fontsize=14, fontweight='bold')
for i, bar in enumerate(bars):
    height = bar.get_height()
    ax4.text(bar.get_x() + bar.get_width()/2., height,
             f'${height:.2f}', ha='center', va='bottom', fontweight='bold')

plt.tight_layout()
plt.savefig('ab_test_analysis.png', dpi=300, bbox_inches='tight')
plt.show()
```

</details>

#

# **Insights**

## **1) CTR Analysis**

Extracted from Python output:

| Variant | CTR   | Users | Clickers |
| ------- | ----- | ----- | -------- |
| A       | 3.20% | 5000  | 160      |
| B       | 3.20% | 5000  | 160      |

➡️ **CTR uplift = 0.00%** (Expected uplift from SQL simulation was ~0.2%)

➡️ **Z-test Result:**

* z = 0.0000
* p = 1.0000
* **NOT statistically significant** at α = 0.05

# 

## **2) Funnel Analysis**

| Stage       | A    | B    |
| ----------- | ---- | ---- |
| Page View   | 5000 | 5000 |
| Click       | 160  | 160  |
| Add to Cart | 120  | 110  |
| Purchase    | 70   | 60   |

### **Drop-Off Rates**

* Page → Click: **96.8%**
* Click → Cart: **25-31%**
* Cart → Purchase: **41-45%**

🔍 **Largest leakage:**
➡️ **Add-to-Cart stage - ~28% average drop-off**

# 

## **3) Revenue Analysis**

| Variant | Purchasing Users | Total Revenue | AOV    | Revenue/User |
| ------- | ---------------- | ------------- | ------ | ------------ |
| A       | 70               | $4105.75      | $58.65 | **0.82**     |
| B       | 60               | $3399.18      | $56.65 | **0.68**     |

➡️ **Variant A generates slightly higher revenue per user.**

# 

## **4) Revenue Leakage**

As calculated:

* **Drop-off at Cart stage:** ~28%
* **Estimated recoverable revenue:** $33.75
* **Revenue Leakage:** **0.4%**

# 

# **Conclusion**

This A/B testing simulation demonstrates:

* **CTR between variants A and B is identical** in the final result and **not statistically significant**.
* The major opportunity lies in **reducing funnel drop-offs**, especially at the **Add-to-Cart stage**.
* Even a **50% improvement in the cart drop-off** yields noticeable revenue recovery.
* SQL + Python provide a complete reproducible pipeline for experiment simulation, analysis, statistics, and visualization.

# 

