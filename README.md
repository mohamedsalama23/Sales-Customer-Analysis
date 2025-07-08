# 📦 Data Warehouse Sales & Customer Analysis – SQL Project

## 🧠 Overview
This project demonstrates a comprehensive data analysis using **SQL** over a **Data Warehouse schema**. The focus is on analyzing sales trends, customer behavior, product performance, and segmentation using structured queries and business logic.

---

## 🎯 Objectives
- Track total sales and running sales over time
- Monitor yearly customer acquisition
- Analyze product sales performance compared to previous years and average benchmarks
- Segment customers based on their lifetime value and purchase behavior
- Evaluate which product categories contribute the most to total sales
- Group products by cost range and analyze distribution
- Build customer reports with demographic insights

---

## 🗃️ Data Model Used
- **Fact Table**: `gold.fact_sales`
- **Dimension Tables**:
  - `gold.dim_products`
  - `gold.dim_customers`

---

## 🛠️ Tools & Technologies
- **SQL Server** / T-SQL
- **CTEs** and **Window Functions**
- **CASE WHEN**, **JOINs**, and **Date Functions**
- Power BI (optional, for visualization layer)

---

## 📊 Key Analyses Performed

### 📈 Sales Performance Over Time
```sql
SELECT 
  YEAR(order_date) AS year,
  MONTH(order_date) AS month,
  SUM(sales_amount) AS total_sales
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date), MONTH(order_date)
ORDER BY YEAR(order_date), MONTH(order_date);

### 🧍‍♂️ New Customers Each Year
```sql
SELECT 
  YEAR(order_date) AS year,
  COUNT(DISTINCT customer_key) AS customers
FROM gold.fact_sales
WHERE order_date IS NOT NULL
GROUP BY YEAR(order_date)
ORDER BY YEAR(order_date);
### 📈 Running Total Sales Over Time
SELECT 
  order_date,
  total_sales,
  SUM(total_sales) OVER (ORDER BY order_date) AS running_totalsales
FROM (
  SELECT 
    DATETRUNC(month, order_date) AS order_date,
    SUM(sales_amount) AS total_sales
  FROM gold.fact_sales
  WHERE order_date IS NOT NULL
  GROUP BY DATETRUNC(month, order_date)
) w;

### 📊 Yearly Product Performance vs Average and Last Year
WITH yearly_performance_products AS (
  SELECT 
    YEAR(f.order_date) AS year,
    product_name,
    SUM(f.sales_amount) AS current_sales
  FROM gold.fact_sales f
  LEFT JOIN gold.dim_products p ON f.product_key = p.product_key
  WHERE f.order_date IS NOT NULL
  GROUP BY product_name, YEAR(f.order_date)
)
SELECT 
  year,
  product_name,
  current_sales,
  AVG(current_sales) OVER (PARTITION BY product_name) AS avg_sales,
  current_sales - AVG(current_sales) OVER (PARTITION BY product_name) AS diff_sales,
  CASE 
    WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) > 0 THEN 'above avg'
    WHEN current_sales - AVG(current_sales) OVER (PARTITION BY product_name) < 0 THEN 'below avg'
    ELSE 'avg'
  END AS avg_change,
  LAG(current_sales) OVER (PARTITION BY product_name ORDER BY year) AS ly_sales,
  current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY year) AS variance_sales,
  CASE 
    WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY year) > 0 THEN 'increase'
    WHEN current_sales - LAG(current_sales) OVER (PARTITION BY product_name ORDER BY year) < 0 THEN 'decrease'
    ELSE 'no change'
  END AS result
FROM yearly_performance_products
ORDER BY product_name, year ASC;

### 🏷️ Sales by Category – Business Impact
WITH sales_category AS (
  SELECT 
    category,
    SUM(sales_amount) AS total_sales
  FROM gold.fact_sales f 
  LEFT JOIN gold.dim_products p ON f.product_key = p.product_key
  GROUP BY category
)
SELECT 
  category,
  total_sales,
  SUM(total_sales) OVER () AS overall_sales,
  CONCAT(ROUND(CAST(total_sales AS FLOAT) / SUM(total_sales) OVER () * 100, 2), ' %') AS percentage_of_total
FROM sales_category
ORDER BY total_sales DESC;

### 🧾 Product Segmentation by Cost Range
WITH cost_segment AS (
  SELECT 
    product_id,
    product_name,
    SUM(cost) AS total_cost,
    CASE 
      WHEN SUM(cost) < 100 THEN 'below 100'
      WHEN SUM(cost) BETWEEN 100 AND 500 THEN '100-500'
      WHEN SUM(cost) BETWEEN 500 AND 1000 THEN '500-1000'
      ELSE 'above-1000'
    END AS cost_segment
  FROM gold.dim_products
  GROUP BY product_id, product_name
)
SELECT 
  COUNT(product_name) AS n_of_product,
  cost_segment
FROM cost_segment
GROUP BY cost_segment
ORDER BY COUNT(DISTINCT product_name) DESC;

### 🧾 Product Segmentation by Cost Range
WITH cost_segment AS (
  SELECT 
    product_id,
    product_name,
    SUM(cost) AS total_cost,
    CASE 
      WHEN SUM(cost) < 100 THEN 'below 100'
      WHEN SUM(cost) BETWEEN 100 AND 500 THEN '100-500'
      WHEN SUM(cost) BETWEEN 500 AND 1000 THEN '500-1000'
      ELSE 'above-1000'
    END AS cost_segment
  FROM gold.dim_products
  GROUP BY product_id, product_name
)
SELECT 
  COUNT(product_name) AS n_of_product,
  cost_segment
FROM cost_segment
GROUP BY cost_segment
ORDER BY COUNT(DISTINCT product_name) DESC;

### 👥 Customer Segmentation by Spending Behavior
WITH customer_life AS (
  SELECT 
    customer_id,
    SUM(sales_amount) AS total_sales,
    MIN(order_date) AS first_order_date,
    MAX(order_date) AS last_order_date,
    DATEDIFF(month, MIN(order_date), MAX(order_date)) AS lifespan
  FROM gold.fact_sales f
  INNER JOIN gold.dim_customers c ON f.customer_key = c.customer_key
  GROUP BY customer_id
)
SELECT 
  customer_segment,
  COUNT(DISTINCT customer_id) AS customers
FROM (
  SELECT 
    customer_id,
    total_sales,
    lifespan,
    CASE 
      WHEN lifespan > 12 AND total_sales > 5000 THEN 'vip'
      WHEN lifespan > 12 AND total_sales <= 5000 THEN 'regular'
      ELSE 'new'
    END AS customer_segment
  FROM customer_life
) t
GROUP BY customer_segment
ORDER BY customers DESC;











