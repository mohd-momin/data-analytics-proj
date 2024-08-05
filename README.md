# Retail Orders Analysis Project

## Overview

Analyzing retail orders involves processing a dataset to extract insights about sales, revenue, and performance trends. You'll use Python for data cleaning and transformation, and SQL for querying and analysis.

## Setup

### Prerequisites

- Python 3.x
- MySQL Server
- Required libraries: `kaggle`, `pandas`, `sqlalchemy`, `pymysql`

### Installation

1. **Install Required Libraries:**

   ```bash
   pip install kaggle pandas sqlalchemy pymysql
   ```

2. **Download Dataset:**

   ```python
   import kaggle
   !kaggle datasets download ankitbansal06/retail-orders -f orders.csv
   ```

3. **Extract the Dataset:**

   ```python
   import zipfile
   zip_ref = zipfile.ZipFile('orders.csv.zip')
   zip_ref.extractall() 
   zip_ref.close()
   ```

## Data Processing

1. **Read and Clean Data:**

   ```python
   import pandas as pd
   df = pd.read_csv('orders.csv', na_values=['Not Available', 'unknown'])
   ```

2. **Transform Data:**

   ```python
   df.rename(columns={'Order Id': 'order_id', 'City': 'city'}, inplace=True)
   df.columns = df.columns.str.lower().str.replace(' ', '_')
   df['profit'] = df['sale_price'] - df['cost_price']
   ```

3. **Convert Dates:**

   ```python
   df['order_date'] = pd.to_datetime(df['order_date'], format="%Y-%m-%d")
   ```

4. **Drop Unnecessary Columns:**

   ```python
   df.drop(columns=['list_price', 'cost_price', 'discount_percent'], inplace=True)
   ```

5. **Load Data into SQL:**

   **Connect to MySQL Database:**

   ```python
   import sqlalchemy as sal
   engine = sal.create_engine('mysql://username:password@host:port/database')
   conn = engine.connect()
   ```

   **Upload Data:**

   ```python
   df.to_sql('df_orders', con=conn, index=False, if_exists='append')
   ```

## SQL Queries

1. **Top 10 Highest Revenue Generating Products:**

   ```sql
   SELECT product_id, SUM(sale_price) AS sales
   FROM df_orders
   GROUP BY product_id
   ORDER BY sales DESC
   LIMIT 10;
   ```

2. **Top 5 Highest Selling Products in Each Region:**

   ```sql
   WITH cte AS (
       SELECT region, product_id, SUM(sale_price) AS sales
       FROM df_orders
       GROUP BY region, product_id
   )
   SELECT * 
   FROM (
       SELECT *,
              ROW_NUMBER() OVER(PARTITION BY region ORDER BY sales DESC) AS rn
       FROM cte
   ) A
   WHERE rn <= 5;
   ```

3. **Month-over-Month Growth Comparison for 2022 and 2023:**

   ```sql
   WITH cte AS (
       SELECT YEAR(order_date) AS order_year, MONTH(order_date) AS order_month,
              SUM(sale_price) AS sales
       FROM df_orders
       GROUP BY YEAR(order_date), MONTH(order_date)
   )
   SELECT order_month,
          SUM(CASE WHEN order_year = 2022 THEN sales ELSE 0 END) AS sales_2022,
          SUM(CASE WHEN order_year = 2023 THEN sales ELSE 0 END) AS sales_2023
   FROM cte 
   GROUP BY order_month
   ORDER BY order_month;
   ```

4. **Highest Sales by Category for Each Month:**

   ```sql
   WITH cte AS (
       SELECT category, DATE_FORMAT(order_date, '%Y%m') AS order_year_month,
              SUM(sale_price) AS sales 
       FROM df_orders
       GROUP BY category, DATE_FORMAT(order_date, '%Y%m')
   )
   SELECT * 
   FROM (
       SELECT *,
              ROW_NUMBER() OVER(PARTITION BY category ORDER BY sales DESC) AS rn
       FROM cte
   ) A
   WHERE rn = 1;
   ```

5. **Subcategory with Highest Growth in Profit (2023 vs. 2022):**

   ```sql
   WITH cte AS (
       SELECT sub_category, YEAR(order_date) AS order_year,
              SUM(sale_price) AS sales
       FROM df_orders
       GROUP BY sub_category, YEAR(order_date)
   ),
   cte2 AS (
       SELECT sub_category,
              SUM(CASE WHEN order_year = 2022 THEN sales ELSE 0 END) AS sales_2022,
              SUM(CASE WHEN order_year = 2023 THEN sales ELSE 0 END) AS sales_2023
       FROM cte 
       GROUP BY sub_category
   )
   SELECT TOP 1 *,
          (sales_2023 - sales_2022) AS profit_growth
   FROM cte2
   ORDER BY profit_growth DESC;
   ```
