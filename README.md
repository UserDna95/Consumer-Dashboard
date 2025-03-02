# Consumer-Dashboard

Problem Statement:
Ad hoc requests for business insights.

Step 1) 
Run all ad-hoc requests in SQL 

1. Provide the list of markets in which the customer  "Atliq  Exclusive"  operates its 
business in the  APAC  region. 

  ```
SELECT market  
FROM gdb023.dim_customer
WHERE customer = 'Atliq Exclusive' AND region = 'APAC'
GROUP BY market;
```

2. What is the percentage of unique product increase in 2021 vs. 2020? 
  ```
SELECT 
    COUNT(DISTINCT CASE WHEN f.fiscal_year = 2020 AND p.product_code IS NOT NULL THEN p.product_code END) AS unique_products_2020,
    COUNT(DISTINCT CASE WHEN f.fiscal_year = 2021 AND p.product_code IS NOT NULL THEN p.product_code END) AS unique_products_2021,
    ((COUNT(DISTINCT CASE WHEN f.fiscal_year = 2021 AND p.product_code IS NOT NULL THEN p.product_code END) - COUNT(DISTINCT CASE WHEN f.fiscal_year = 2020 AND p.product_code IS NOT NULL THEN p.product_code END)) * 100.0 / COUNT(DISTINCT CASE WHEN f.fiscal_year = 2020 AND p.product_code IS NOT NULL THEN p.product_code END)) AS percentage_chg
FROM dim_product AS p
JOIN fact_gross_price AS f 
    ON p.product_code = f.product_code;
  ```

3. Provide a report with each segment's unique product counts  and 
sort them in descending order of product counts. The final output contains 

  ```
SELECT 
    segment, 
    COUNT(DISTINCT product_code) AS product_count 
FROM 
    dim_product
GROUP BY 
    segment 
ORDER BY 
    product_count DESC;
  ```

4. Which segment had the most increase in unique products in 
2021 vs 2020?
  ```
SELECT 
    segment,
    COUNT(DISTINCT CASE WHEN f.fiscal_year = 2020 AND p.product_code IS NOT NULL THEN p.product_code END) AS product_count_2020,
    COUNT(DISTINCT CASE WHEN f.fiscal_year = 2021 AND p.product_code IS NOT NULL THEN p.product_code END) AS product_count_2021,
    (COUNT(DISTINCT CASE WHEN f.fiscal_year = 2021 AND p.product_code IS NOT NULL THEN p.product_code END) - COUNT(DISTINCT CASE WHEN f.fiscal_year = 2020 AND p.product_code IS NOT NULL THEN p.product_code END)) AS difference
FROM dim_product AS p
JOIN fact_gross_price AS f 
    ON p.product_code = f.product_code
GROUP BY 
    segment 
ORDER BY 
    difference DESC;
  ```

5. Get the products that have the highest and lowest manufacturing costs. 
  ```
(
    SELECT 
        p.product_code, 
        p.product, 
        m.manufacturing_cost
    FROM 
        dim_product AS p
    JOIN 
        fact_manufacturing_cost AS m 
    ON 
        p.product_code = m.product_code
    ORDER BY 
        m.manufacturing_cost DESC
)
UNION ALL
(
    SELECT 
        p.product_code, 
        p.product, 
        m.manufacturing_cost
    FROM 
        dim_product AS p
    JOIN 
        fact_manufacturing_cost AS m 
    ON 
        p.product_code = m.product_code
    ORDER BY 
        m.manufacturing_cost ASC
);
  ```

6. Generate a report that contains the top 5 customers who received an 
average high  pre_invoice_discount_pct  for the  fiscal  year 2021  and in the 
Indian  market. 

  ```
SELECT 
    c.customer_code,
    c.customer, 
    AVG(f.pre_invoice_discount_pct) AS average_discount_percentage
FROM 
    dim_customer AS c
JOIN
    fact_pre_invoice_deductions AS f 
ON 
    c.customer_code = f.customer_code 
WHERE 
    f.fiscal_year = '2021'
    AND c.market = 'India'
GROUP BY 
    c.customer_code, c.customer
ORDER BY 
    average_discount_percentage DESC
LIMIT 5;
  ```

7. Get the complete report of the Gross sales amount for the customer  “Atliq 
Exclusive”  for each month.

  ```
(
    SELECT 
        MONTH(m.date) AS month, 
        m.fiscal_year,
        SUM(g.gross_price) AS gross_sales_amount 
    FROM 
        dim_customer AS c 
    JOIN 
        fact_sales_monthly AS m 
    ON 
        c.customer_code = m.customer_code
    JOIN
        fact_gross_price AS g
    ON 
        m.product_code = g.product_code
    WHERE 
        c.customer = 'Atliq Exclusive'
    GROUP BY 
        MONTH(m.date), m.fiscal_year
)
UNION ALL
(
    SELECT 
        MONTH(m.date) AS month,
        m.fiscal_year,
        SUM(g.gross_price) AS gross_sales_amount 
    FROM 
        fact_sales_monthly AS m 
    JOIN 
        fact_gross_price AS g 
    ON 
        m.product_code = g.product_code
    GROUP BY 
        MONTH(m.date), m.fiscal_year
);
  ```
8. In which quarter of 2020 did we get the maximum total_sold_quantity? 

  ```
SELECT 
    QUARTER(date) AS quarter, 
    SUM(sold_quantity) AS total_sold_quantity 
FROM 
    fact_sales_monthly 
WHERE 
    fiscal_year = '2020'
GROUP BY 
    QUARTER(date)
ORDER BY 
    total_sold_quantity DESC
LIMIT 1;
  ```

9. Which channel helped to bring more gross sales in the fiscal year 2021 
and the percentage of contribution? 

  ```
SELECT 
    c.channel, 
    SUM(g.gross_price) AS gross_sales_mln,
    (SUM(g.gross_price) / total_sales.total_gross_sales) * 100 AS percentage
FROM 
    dim_customer AS c 
JOIN 
    fact_sales_monthly AS m 
ON 
    c.customer_code = m.customer_code
JOIN 
    fact_gross_price AS g 
ON 
    m.product_code = g.product_code
JOIN 
    (SELECT SUM(gross_price) AS total_gross_sales 
     FROM fact_gross_price 
     JOIN fact_sales_monthly 
     ON fact_gross_price.product_code = fact_sales_monthly.product_code 
     WHERE fact_sales_monthly.fiscal_year = '2021') AS total_sales
ON 1=1
WHERE 
    m.fiscal_year = '2021'
GROUP BY 
    c.channel, total_sales.total_gross_sales
ORDER BY 
    gross_sales_mln DESC
LIMIT 1;
  ```

10. Get the Top 3 products in each division that have a high 
total_sold_quantity in the fiscal_year 2021? 

  ```
WITH RankedProducts AS (
    SELECT 
        p.division, 
        p.product_code,
        p.product, 
        SUM(m.sold_quantity) AS total_sold_quantity,
        RANK() OVER (PARTITION BY p.division ORDER BY SUM(m.sold_quantity) DESC) AS rank_order
    FROM 
        dim_product AS p 
    JOIN
        fact_sales_monthly AS m 
    ON 
        p.product_code = m.product_code
    WHERE 
        m.fiscal_year = '2021'
    GROUP BY 
        p.division, p.product_code, p.product
)
SELECT 
    division, 
    product_code, 
    product, 
    total_sold_quantity, 
    rank_order
FROM 
    RankedProducts
WHERE 
    rank_order <= 3
ORDER BY 
    division, rank_order;
  ```

11. Find the total price amount of products sold in a new table. 

  ```
Summary_Total_Price = 
SUMMARIZE(
    'gdb023 fact_sales_monthly',
    'gdb023 fact_sales_monthly'[customer_code],
    'gdb023 fact_sales_monthly'[fiscal_year],
    "Total_Price_Aggregated", SUM('gdb023 fact_sales_monthly'[Total_Price_Sold])
)

Total_Price_Sold = 
'gdb023 fact_sales_monthly'[sold_quantity] * 'gdb023 fact_sales_monthly'[Price_Per_Product]

Price_Per_Product = 
LOOKUPVALUE(
    'gdb023 fact_gross_price'[gross_price],
    'gdb023 fact_gross_price'[product_code], 'gdb023 fact_sales_monthly'[product_code],
   'gdb023 fact_gross_price'[fiscal_year], 'gdb023 fact_sales_monthly'[fiscal_year]
)

total_price_post_discount = 
    'gdb023 fact_pre_invoice_deductions'[pre_invoice_discount_pct] * 
    LOOKUPVALUE(
        Summary_Total_Price[Total_Price_Aggregated],
        Summary_Total_Price[customer_code], 'gdb023 fact_pre_invoice_deductions'[customer_code],
        Summary_Total_Price[fiscal_year], 'gdb023 fact_pre_invoice_deductions'[fiscal_year]
    )
  ```

Step 2)
Data Structure Overview in Power BI 

![Screen Shot 2025-03-02 at 5 31 38 AM](https://github.com/UserDna95/Consumer-Dashboard/blob/main/2025-03-02%20(6).png) 

Step 3) 
Dashboard in Power BI & Insights

> Page 1 "Sales"
When examining total sales by customer and market between 2020 and 2021, Amazon and India stand out as top performers. The Network and Storage product types have the highest sales, while PCs, including Notebooks and Desktops, rank lower. This trend is likely due to the frequent need for Storage and Network products compared to the longer lifespan of PCs.


> Page 2: "Product Details" 
When examining the discounted prices across various customers and markets, it becomes apparent that Amazon and India offer the most substantial discounts. Additionally, this page provides insights into the manufacturing costs and pricing of products by segment. Typically, Desktops and Notebooks emerge as the highest costing product types, while Accessories and Storage products tend to have the lowest costs. Interestingly, Accessories witnessed the most significant increase in unique product offerings from 2020 to 2021, followed by Notebooks and Peripherals. This trend could be attributed to the lower manufacturing costs associated with more affordable product types like Accessories, facilitating greater variety.

![Screen Shot 2025-03-02 at 5 31 38 AM](https://github.com/UserDna95/Consumer-Dashboard/blob/main/2025-03-02%20(4).png) 
![Screen Shot 2025-03-02 at 5 31 38 AM](https://github.com/UserDna95/Consumer-Dashboard/blob/main/2025-03-02%20(5).png) 

