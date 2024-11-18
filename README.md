---

# **Atliq SQL Data Analysis Project**

### **Difficulty Level: Advanced**

---

## **Project Overview**

I have worked on analyzing a dataset of approx 1.5 million sales records for the company called Atliq Technologies which sells hardware through different platforms like through retailers, distributors, e-comm companies and directly through its online portal as well. 
This project involves extensive querying of customer behavior, product performance, market analysis and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, optimizing queries by creating additional tables,views,stored procedures, triggers and user defined functions. Tackle multiple business problems including revenue analysis, customer segmentation,forecast accuracy, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.

---

![ERD Scratch](https://github.com/najirh/amazon_usa_project5/blob/main/erd2.png)

---



## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer Analysis
- Sales trends
- Inventory management
- Forecasting and product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Inconsistent product availability because of wrong forecast.
2. Customer performance analysis.

---

## **Solving Business Problems**

### Solutions Implemented:
Q1. As a product owner I want to generate a report of individual product sales(aggregated on monhtly basis 
    at product level ) for croma  customer for fy = 2021, so i can track individual product sales
    and run further product analysis on excel
 
Query the top 10 products by total sales value.
Challenge: Include date, product code, product name, variant, sold_quantity, gross_price, gross_price_total.

```sql
SELECT
      fsm.date,
      fsm.product_code,
      p.product,
      variant,
      sold_quantity,
      gross_price ,
      gross_price * sold_quantity as gross_price_total
FROM fact_sales_monthly fsm
JOIN dim_product p
ON fsm.product_code = p.product_code
JOIN gdb056.gross_price gp 
ON fsm.product_code = gp.product_code 
AND get_fiscal_year(fsm.date)  = gp.fiscal_year
WHERE customer_code = 90002002  
AND get_fiscal_year(date) = 2021
ORDER BY date asc;
 ```

Q2. As a product owner I want to generate a report of aggregated monthly sales report for croma india customer
so that we can track how much sales this particular customer is generating for atliq . It should have month and
gross sales amount generated each month

```sql
SELECT
      fsm.date as month,  
      ROUND(SUM(gross_price * sold_quantity),2) as gross_price_total
FROM fact_sales_monthly fsm
JOIN gdb056.gross_price gp 
ON fsm.product_code = gp.product_code 
AND get_fiscal_year(fsm.date)  = gp.fiscal_year
WHERE customer_code = 90002002  
GROUP BY fsm.date
ORDER BY month asc;
```

Q3. As a product owner I want to generate a report of aggregated yearly sales report for croma india customer
so that we can track how much sales this particular customer is generating for atliq . It should have year and
gross sales amount generated each year. 
CHALLENGE: Use get_fiscal_year user defined function to derive the result

```sql
SELECT
      get_fiscal_year(fsm.date) as fiscal_year,  
      ROUND(SUM(gross_price * sold_quantity),2) as gross_price_total
FROM fact_sales_monthly fsm
JOIN gdb056.gross_price gp 
ON fsm.product_code = gp.product_code 
AND get_fiscal_year(fsm.date)  = gp.fiscal_year
WHERE customer_code = 90002002  
GROUP BY get_fiscal_year(fsm.date)
ORDER BY fiscal_year asc;
```

Q4. As a product owner I want to generate a report of aggregated monthly sales report for different customer 
   like amazon, ezone, than for some other customer, It should have year and gross sales amount generated each month, 
   Challenge: Use stored procedure to derive the result which will ask for customer_code and all the months for that fiscal
           and the output will include month and gross revenue for that month.

```sql
CREATE PROCEDURE `get_monthly_sales_for_customer` (
in_customer_code Text,
fiscal_year year
)
BEGIN
SELECT
      fsm.date as month,  
      ROUND(sum(gross_price * sold_quantity),2) as gross_price_total
FROM fact_sales_monthly fsm
JOIN gdb056.gross_price gp 
ON fsm.product_code = gp.product_code 
AND get_fiscal_year(fsm.date)  = gp.fiscal_year
WHERE fsm.customer_code = in_customer_code
GROUP BY fsm.date
ORDER BY month asc;
END
```


Q5. As a product owner I want to generate a report of aggregated monthly sales report for Atliq customers 
   who have multiple customer code for the same customer name. 
   Challenge: Use stored procedure to derive the result which will ask for list of customer_code and fiscal_year
              and the output will include month and gross revenue for that month agregated for both the customers

```sql
CREATE DEFINER=`root`@`localhost` PROCEDURE `get_monthly_sales_for_customer`(
in_customer_codes Text,
fiscal_year year
)
BEGIN
SELECT fsm.date as month,  
       ROUND(SUM(gross_price * sold_quantity),2) as gross_price_total
FROM fact_sales_monthly fsm
JOIN gdb056.gross_price gp 
ON fsm.product_code = gp.product_code 
AND get_fiscal_year(fsm.date)  = gp.fiscal_year
WHERE find_in_set(customer_code,in_customer_codes)  > 0 and get_fiscal_year(fsm.date) = fiscal_year
GROUP BY fsm.date
ORDER BY month asc;

END```

Q6.  Create a stored procdure to find the market badge, if qty sold> 5 million then market is 
     gold else silver
     Challenge: If no market is provided then take India as default value 

```sql

CREATE DEFINER=`root`@`localhost` PROCEDURE `get_market_badge`(
IN in_market varchar(45),
In in_fiscal_year year,
OUT out_badge varchar(45)
)
BEGIN
  DECLARE qty int  default 0;

IF in_market = ""
THEN SET in_market = "India";
END IF;

# declare intermediate variable

SELECT 
     SUM(sold_quantity) as total_qty into qty
FROM gdb041.fact_sales_monthly fsm
JOIN dim_customer c 
ON fsm.customer_code = c.customer_code  
WHERE get_fiscal_year(fsm.date) = in_fiscal_year and c.market = in_market
GROUP BY c.market
;

IF qty>5000000
THEN set out_badge = "Gold";

ELSE 
SET out_badge = "Silver";
 
END IF;
END

```

Q7. Create a comparison for the forecast accuracy analysis for the year 2020 vs 2021


```sql

WITH error_pct as (
SELECT customer_code,
SUM(sold_quantity) as total_sold_qty,
fiscal_year,
SUM(forecast_quantity) as total_forecast_qty,
ABS(SUM(forecast_quantity) - SUM(sold_quantity)) as abs_error,
ABS(SUM(forecast_quantity) - SUM(sold_quantity))*100/SUM(forecast_quantity) as abs_error_pct
FROM gdb0041.fact_act_est
GROUP BY customer_code, fiscal_year),

forecast_accuracy_2020 as (
SELECT *,
      if(abs_error_pct >=100,0,100- abs_error_pct) as forecast_accuracy_2020
FROM error_pct
WHERE fiscal_year = 2020
ORDER BY forecast_accuracy_2020 desc),

forecast_accuracy_2021 as (
SELECT * ,
      if(abs_error_pct >=100,0,100- abs_error_pct) as forecast_accuracy_2021
FROM error_pct
WHERE fiscal_year = 2021
ORDER BY forecast_accuracy_2021 desc)

SELECT
     f21.customer_code,
     f20.forecast_accuracy_2020 as forecast_accuracy_2020,
     f21.forecast_accuracy_2021 as forecast_accuracy_2021
FROM forecast_accuracy_2021 f21
JOIN forecast_accuracy_2020 f20 ON 
f21.customer_code = f20.customer_code;
```



Q8. Create net sales table after joining all the neccesary tables
    Challenge: Create a view to optimize your database experience, without using extra storage, to accelerate data analysis and provide your data extra security.
    Solution: To Create Net sales, we will create sales pre_inv discount view, this will create net invoice sales, and then will create post_inv_discount view 
              derieved from pre_inv discount view, and then we will derive net sales views from post_inv discount view

```sql
CREATE 
    ALGORITHM = UNDEFINED 
    DEFINER = `root`@`localhost` 
    SQL SECURITY DEFINER
VIEW `sales_preinv_discount` AS
    SELECT 
        `s`.`date` AS `date`,
        `s`.`product_code` AS `product_code`,
        `s`.`fiscal_year` AS `fiscal_year`,
        `c`.`market` AS `market`,
        `c`.`customer_code` AS `customer_code`,
        `p`.`product` AS `product`,
        `p`.`variant` AS `variant`,
        `s`.`sold_quantity` AS `sold_quantity`,
        `gp`.`gross_price` AS `gross_price_per_item`,
        ROUND((`s`.`sold_quantity` * `gp`.`gross_price`),
                2) AS `gross_price_total`,
        `pre`.`pre_invoice_discount_pct` AS `pre_invoice_discount_pct`
    FROM
        ((((`fact_sales_monthly` `s`
        JOIN `dim_customer` `c` ON ((`c`.`customer_code` = `s`.`customer_code`)))
        JOIN `dim_product` `p` ON ((`p`.`product_code` = `s`.`product_code`)))
        JOIN `fact_gross_price` `gp` ON (((`gp`.`product_code` = `s`.`product_code`)
            AND (`gp`.`fiscal_year` = `s`.`fiscal_year`))))
        JOIN `fact_pre_invoice_deductions` `pre` ON (((`s`.`customer_code` = `pre`.`customer_code`)
            AND (`pre`.`fiscal_year` = `s`.`fiscal_year`))))



CREATE 
    ALGORITHM = UNDEFINED 
    DEFINER = `root`@`localhost` 
    SQL SECURITY DEFINER
VIEW `sales_postinv_discount` AS
    SELECT 
        `pre`.`date` AS `date`,
        `pre`.`fiscal_year` AS `fiscal_year`,
        `pre`.`product_code` AS `product_code`,
        `pre`.`market` AS `market`,
        `pre`.`customer_code` AS `customer_code`,
        `pre`.`product` AS `product`,
        `pre`.`variant` AS `variant`,
        `pre`.`sold_quantity` AS `sold_quantity`,
        `pre`.`gross_price_per_item` AS `gross_price_per_item`,
        `pre`.`gross_price_total` AS `gross_price_total`,
        `pre`.`pre_invoice_discount_pct` AS `pre_invoice_discount_pct`,
        `post`.`discounts_pct` AS `discounts_pct`,
        `post`.`other_deductions_pct` AS `other_deductions_pct`,
        ((1 - `pre`.`pre_invoice_discount_pct`) * `pre`.`gross_price_total`) AS `net_invoice_sales`,
        (`post`.`discounts_pct` + `post`.`other_deductions_pct`) AS `total_postinv_discount_pct`
    FROM
        (`sales_preinv_discount` `pre`
        JOIN `fact_post_invoice_deductions` `post` ON (((`pre`.`customer_code` = `post`.`customer_code`)
            AND (`pre`.`product_code` = `post`.`product_code`)
            AND (`pre`.`date` = `post`.`date`))))

CREATE 
    ALGORITHM = UNDEFINED 
    DEFINER = `root`@`localhost` 
    SQL SECURITY DEFINER
VIEW `net_sales` AS
    SELECT 
        `sales_postinv_discount`.`date` AS `date`,
        `sales_postinv_discount`.`product_code` AS `product_code`,
        `sales_postinv_discount`.`market` AS `market`,
        `sales_postinv_discount`.`fiscal_year` AS `fiscal_year`,
        `sales_postinv_discount`.`customer_code` AS `customer_code`,
        `sales_postinv_discount`.`product` AS `product`,
        `sales_postinv_discount`.`variant` AS `variant`,
        `sales_postinv_discount`.`sold_quantity` AS `sold_quantity`,
        `sales_postinv_discount`.`gross_price_per_item` AS `gross_price_per_item`,
        `sales_postinv_discount`.`gross_price_total` AS `gross_price_total`,
        `sales_postinv_discount`.`pre_invoice_discount_pct` AS `pre_invoice_discount_pct`,
        `sales_postinv_discount`.`discounts_pct` AS `discounts_pct`,
        `sales_postinv_discount`.`other_deductions_pct` AS `other_deductions_pct`,
        `sales_postinv_discount`.`net_invoice_sales` AS `net_invoice_sales`,
        `sales_postinv_discount`.`total_postinv_discount_pct` AS `total_postinv_discount_pct`,
        ((1 - `sales_postinv_discount`.`total_postinv_discount_pct`) * `sales_postinv_discount`.`net_invoice_sales`) AS `net_sales`
    FROM
        `sales_postinv_discount`
```


Q9. Create a comparison for the forecast accuracy analysis for the year 2020 vs 2021


```sql

WITH error_pct as (
SELECT customer_code,
SUM(sold_quantity) as total_sold_qty,
fiscal_year,
SUM(forecast_quantity) as total_forecast_qty,
ABS(SUM(forecast_quantity) - SUM(sold_quantity)) as abs_error,
ABS(SUM(forecast_quantity) - SUM(sold_quantity))*100/SUM(forecast_quantity) as abs_error_pct
FROM gdb0041.fact_act_est
GROUP BY customer_code, fiscal_year),

forecast_accuracy_2020 as (
SELECT *,
      if(abs_error_pct >=100,0,100- abs_error_pct) as forecast_accuracy_2020
FROM error_pct
WHERE fiscal_year = 2020
ORDER BY forecast_accuracy_2020 desc),

forecast_accuracy_2021 as (
SELECT * ,
      if(abs_error_pct >=100,0,100- abs_error_pct) as forecast_accuracy_2021
FROM error_pct
WHERE fiscal_year = 2021
ORDER BY forecast_accuracy_2021 desc)

SELECT
     f21.customer_code,
     f20.forecast_accuracy_2020 as forecast_accuracy_2020,
     f21.forecast_accuracy_2021 as forecast_accuracy_2021
FROM forecast_accuracy_2021 f21
JOIN forecast_accuracy_2020 f20 ON 
f21.customer_code = f20.customer_code;
```

Q9. Create a trigger that will add data automatically to the fact_act_est table created above, if any new record is inserted into monthly sales table or in forecast table.


```sql

CREATE DEFINER=`root`@`localhost` TRIGGER `fact_sales_monthly_AFTER_INSERT` AFTER INSERT ON `fact_sales_monthly` FOR EACH ROW BEGIN
Insert into fact_act_est( date,product_code,customer_code,sold_quantity
)
values(
     New.date,
	 New.product_code,
     New.customer_code,
     New.sold_quantity)
     on duplicate key update 
     sold_quantity = values(sold_quantity);
END



```


---

---

## **Learning Outcomes**

This project enabled me to:
- Work with big data sets
- Create user defined functions, views, stored procedures, triggers
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This advanced SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. 
From improving customer analysis to query optimization by creating addition tables, or views, to provide the extra layer of security to the
database by creating stored procedures, user defined functions and views, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems, create stored procedures, views, triggers, user defined functions
and drive business decision-making.

---

### **Entity Relationship Diagram (ERD)**
![ERD](https://github.com/najirh/amazon_usa_project5/blob/main/erd.png)

---
