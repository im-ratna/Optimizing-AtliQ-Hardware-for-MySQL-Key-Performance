# Atliq Hardware SQL Project
## Project Overview
### About Atliq Hardware
Atliq Hardware is a leading computer hardware producer based in India, with well-established operations globally. The company identified the need for better data insights to facilitate quick and informed decision-making. To address this, they aimed to expand their data analytics team and conducted an SQL challenge to assess candidates' technical and soft skills.

## Project Focus: Optimizing AtliQ Hardware for MySQL Key Performance
The project's primary goal was to optimize the Atliq Hardware MySQL database for enhanced performance, particularly focusing on key performance indicators for MySQL. The imaginary dataset closely mimics real-world scenarios, providing a practical environment to apply SQL techniques and solutions.

## AIMS Grid - Project Management
### Purpose
*   Unlock sales insights, automate decision support for the sales team, and reduce manual data gathering time.

### Stakeholders
*   Sales director, marketing team, customer service team, data analytics team, and IT.

### End Result
*   An automated dashboard providing quick and latest sales insights for data-driven decision-making.

### Success Criteria
*   Dashboard uncovering sales order insights with the latest data available.
*   Sales team able to make better decisions, proving a 10% cost saving of total spend.
*   Sales analysis stops manual data gathering, saving 20% of business time for value-added activities.
  ## Project Objectives
### Data Exploration
*   Thoroughly examine the dataset, understanding tables, relationships, and data types.

### Data Analysis
*  Perform SQL queries to answer critical business questions, such as sales trends and popular products.

### Understanding the Dataset
*   Tables include Dim_Customer, Dim_Product, Fact_Gross_Price, Fact_Manufacturing_Cost, Fact_Pre_Invoice_Deductions, and Fact_Sales_Monthly, interconnected through primary and foreign keys.

## SQL Queries and Insights
### Module: User-Defined SQL Functions

##### a. First, grab customer codes for Croma India
```
SELECT * FROM dim_customer WHERE customer LIKE "%croma%" AND market="india";
```
##### b. Get all sales transaction data for Croma India in fiscal year 2021
```
SELECT * FROM fact_sales_monthly 
WHERE customer_code=90002002 AND YEAR(DATE_ADD(date, INTERVAL 4 MONTH))=2021 
ORDER BY date ASC
LIMIT 100000;
```
##### c. Create a function 'get_fiscal_year' to get fiscal year by passing the date
```
CREATE FUNCTION `get_fiscal_year`(calendar_date DATE) 
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE fiscal_year INT;
    SET fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));
    RETURN fiscal_year;
END
```
##### d. Replacing the function created in step b
```
SELECT * FROM fact_sales_monthly 
WHERE customer_code=90002002 AND get_fiscal_year(date)=2021 
ORDER BY date ASC
LIMIT 100000;
```
### Module: Gross Sales Report: Monthly Product Transactions
##### a. Perform joins to pull product information
```
SELECT s.date, s.product_code, p.product, p.variant, s.sold_quantity 
FROM fact_sales_monthly s
JOIN dim_product p ON s.product_code=p.product_code
WHERE customer_code=90002002 AND get_fiscal_year(date)=2021
LIMIT 1000000;
```
##### b. Performing join with 'fact_gross_price' table with the above query and generating required fields
```
SELECT s.date, s.product_code, p.product, p.variant, s.sold_quantity, g.gross_price,
       ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total
FROM fact_sales_monthly s
JOIN dim_product p ON s.product_code=p.product_code
JOIN fact_gross_price g ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
WHERE customer_code=90002002 AND get_fiscal_year(s.date)=2021
LIMIT 1000000;
```
### Module: Gross Sales Report: Total Sales Amount
##### Generate monthly gross sales report for Croma India for all years
```
SELECT s.date, SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
FROM fact_sales_monthly s
JOIN fact_gross_price g ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
WHERE customer_code=90002002
GROUP BY date;
```
### Module: Stored Procedures: Monthly Gross Sales Report
##### Generate monthly gross sales report for any customer using a stored procedure
```
CREATE PROCEDURE `get_monthly_gross_sales_for_customer`(
    in_customer_codes TEXT
)
BEGIN
    SELECT s.date, SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
    FROM fact_sales_monthly s
    JOIN fact_gross_price g ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
    WHERE FIND_IN_SET(s.customer_code, in_customer_codes) > 0
    GROUP BY s.date
    ORDER BY s.date DESC; 
END
```
### Module: Stored Procedure: Market Badge
```
-- Write a stored procedure to retrieve market badge (Gold/Silver) based on total sold quantity
CREATE PROCEDURE `get_market_badge`(
    IN in_market VARCHAR(45),
    IN in_fiscal_year YEAR,
    OUT out_level VARCHAR(45))
BEGIN
    DECLARE qty INT DEFAULT 0;
    
    -- Default market is India
    IF in_market = "" THEN
        SET in_market="India";
    END IF;

    -- Retrieve total sold quantity for a given market in a given year
    SELECT SUM(s.sold_quantity) INTO qty
    FROM fact_sales_monthly s
    JOIN dim_customer c ON s.customer_code=c.customer_code
    WHERE get_fiscal_year(s.date)=in_fiscal_year AND c.market=in_market;

    -- Determine Gold vs Silver status
    IF qty > 5000000 THEN
        SET out_level = 'Gold';
    ELSE
        SET out_level = 'Silver';
    END IF;
END
```
### Module: Problem Statement and Pre-Invoice Discount Report
##### Include pre-invoice deductions in Croma detailed report
```
SELECT s.date, s.product_code, p.product, p.variant, s.sold_quantity, g.gross_price as gross_price_per_item,
       ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total, pre.pre_invoice_discount_pct
FROM fact_sales_monthly s
JOIN dim_product p ON s.product_code=p.product_code
JOIN fact_gross_price g ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
JOIN fact_pre_invoice_deductions pre ON pre.customer_code = s.customer_code AND pre.fiscal_year=get_fiscal_year(s.date)
WHERE s.customer_code=90002002 AND get_fiscal_year(s.date)=2021 LIMIT 1000000;
```
##### Same report but for all customers
```
SELECT s.date, s.product_code, p.product, p.variant, s.sold_quantity, g.gross_price as gross_price_per_item,
       ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total, pre.pre_invoice_discount_pct
FROM fact_sales_monthly s
JOIN dim_product p ON s.product_code=p.product_code
JOIN fact_gross_price g ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
JOIN fact_pre_invoice_deductions pre ON pre.customer_code = s.customer_code AND pre.fiscal_year=get_fiscal_year(s.date)
WHERE get_fiscal_year(s.date)=2021 LIMIT 1000000;
```
### Module: Performance Improvement #1
##### Creating dim_date and joining to reduce query execution time
```
SELECT s.date, s.customer_code, s.product_code, p.product, p.variant, s.sold_quantity,
       g.gross_price as gross_price_per_item, ROUND(s.s

```
### Module: Create a Helper Table
##### Create fact_act_est table
```
	drop table if exists fact_act_est;

	create table fact_act_est
	(
        	select 
                    s.date as date,
                    s.fiscal_year as fiscal_year,
                    s.product_code as product_code,
                    s.customer_code as customer_code,
                    s.sold_quantity as sold_quantity,
                    f.forecast_quantity as forecast_quantity
        	from 
                    fact_sales_monthly s
        	left join fact_forecast_monthly f 
        	using (date, customer_code, product_code)
	)
	union
	(
        	select 
                    f.date as date,
                    f.fiscal_year as fiscal_year,
                    f.product_code as product_code,
                    f.customer_code as customer_code,
                    s.sold_quantity as sold_quantity,
                    f.forecast_quantity as forecast_quantity
        	from 
		    fact_forecast_monthly  f
        	left join fact_sales_monthly s 
        	using (date, customer_code, product_code)
	);

	update fact_act_est
	set sold_quantity = 0
	where sold_quantity is null;

	update fact_act_est
	set forecast_quantity = 0
	where forecast_quantity is null;
```
### Module: Database Triggers

##### create the trigger to automatically insert record in fact_act_est table whenever insertion happens in fact_sales_monthly 
```
	CREATE DEFINER=CURRENT_USER TRIGGER `fact_sales_monthly_AFTER_INSERT` AFTER INSERT ON `fact_sales_monthly` FOR EACH ROW 
	BEGIN
        	insert into fact_act_est 
                        (date, product_code, customer_code, sold_quantity)
    		values (
                	NEW.date, 
        		NEW.product_code, 
        		NEW.customer_code, 
        		NEW.sold_quantity
    		 )
    		on duplicate key update
                         sold_quantity = values(sold_quantity);
	END
```
##### create the trigger to automatically insert record in fact_act_est table whenever insertion happens in fact_forecast_monthly 
```
	CREATE DEFINER=CURRENT_USER TRIGGER `fact_forecast_monthly_AFTER_INSERT` AFTER INSERT ON `fact_forecast_monthly` FOR EACH ROW 
	BEGIN
        	insert into fact_act_est 
                        (date, product_code, customer_code, forecast_quantity)
    		values (
                	NEW.date, 
        		NEW.product_code, 
        		NEW.customer_code, 
        		NEW.forecast_quantity
    		 )
    		on duplicate key update
                         forecast_quantity = values(forecast_quantity);
	END
```
## Conclusion
In conclusion, the Atliq Hardware SQL project successfully addressed the optimization of the MySQL database to enhance performance and facilitate data-driven decision-making. Key modules covered data exploration, user-defined SQL functions, gross sales reporting, stored procedures, and performance improvements. Noteworthy features included automated monthly gross sales reports, customer-specific functions, and a market badge determination procedure. The project demonstrated a holistic approach to database management, balancing performance, automation, and insightful reporting for Atliq Hardware's strategic goals.






