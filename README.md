



![487090-company-logo](https://github.com/user-attachments/assets/7ce11b8a-0669-4dfe-a456-fcfc9127ddbf)





TASK 1 :- As a product owner, I want to generate a report of individual product sales (aggregated on a monthly
basis at the product code level) for Croma India customer for FY=2021 so that I can track individual
product sales and run further product analytics on it in excel.

The report should have the following fields,

1. Month
2. Product Name
3. Variant
4. Sold Quantity
5. Gross Price Per Item
6. Gross Price Total

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- step 1 
 select * from dim_customer where customer like "%croma%" and market = "india"
--  90002002 croma customer code

-- step 2
select * from fact_sales_monthly where customer_code =90002002 

-- fiscsal year
fiscal_year = YEAR(DATE_ADD(calendar_date, INTERVAL 4 MONTH));

-- you can also create the user devine function 
-- joining of 2 tables 
select s.date,s.fiscal_year,p.product,p.variant,s.sold_quantity,g.gross_price
	, round((s.sold_quantity/g.gross_price),2)as gross_per_item
 from fact_sales_monthly s 
join dim_product p
using (product_code)
join fact_gross_price g
ON g.fiscal_year=get_fiscal_year(s.date)
AND g.product_code=s.product_code
 where customer_code =90002002 ;
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
TASK :-2 Gross monthly total sales report for Croma
As a product owner, I need an aggregate monthly gross sales report for Croma India customer so that I can track
how much sales this particular customer is generating for AtliQ and manage our relationships accordingly.

The report should have the following fields,

1. Month

2. Total gross sales amount to Croma India in this month

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
SELECT      s.date, 
    	    SUM(ROUND(s.sold_quantity*g.gross_price,2)) as monthly_sales
	FROM fact_sales_monthly s
	JOIN fact_gross_price g
        ON g.fiscal_year=get_fiscal_year(s.date) AND g.product_code=s.product_code
	WHERE 
             customer_code=90002002
	GROUP BY date;
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

TASK 3 Created stored proc for monthly gross sales report

As a data analyst, I want to create a stored proc for monthly gross sales report so that I don't have to
manually modify the query every time. Stored proc can be run by other users to (who have limited access
to database) and they can generate this report without having to involve data analytics team.

The report should have the following columns,

1. Month

2. Total gross sales in that month from a given customer

 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Stored Procedure:-
CREATE DEFINER=`root`@`localhost` PROCEDURE `get_monthly_gross_sales_for_customerf`(
	In_customer_code text
)
BEGIN
select 
	s.date,
    s.product_code,
    sum(s.sold_quantity*gross_price) as total_gross_price

from fact_sales_monthly s
join fact_gross_price g 
	on s.product_code = g.product_code and
	g.fiscal_year = get_fiscal_year(s.date)
where 
	find_in_set(s.customer_code,In_customer_code)>0
    -- similar to where customer_code in (customer_code1, customer_code 2)
    -- we use 2 customer code due to the amazan name has 2 customer code so we use 2 any one is present in the set it give result
	group by s.date 
	order by s.date asc;


END

 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


TASK 4:-Stored proc for market badge

Description
Create a stored proc that can determine the market badge based on the following logic,

If total sold quantity > 5 million that market is considered Gold else it is Silver

My input will be,

. market

· fiscal year

Output

· market badge


 CREATE PROCEDURE `get_market_badge`(
        	IN in_market VARCHAR(45),
        	IN in_fiscal_year YEAR,
        	OUT out_level VARCHAR(45)
	)
	BEGIN
             DECLARE qty INT DEFAULT 0;
    
    	     # Default market is India
    	     IF in_market = "" THEN
                  SET in_market="India";
             END IF;
    
    	     # Retrieve total sold quantity for a given market in a given year
             SELECT 
                  SUM(s.sold_quantity) INTO qty
             FROM fact_sales_monthly s
             JOIN dim_customer c
             ON s.customer_code=c.customer_code
             WHERE 
                  get_fiscal_year(s.date)=in_fiscal_year AND
                  c.market=in_market;
        
             # Determine Gold vs Silver status
             IF qty > 5000000 THEN
                  SET out_level = 'Gold';
             ELSE
                  SET out_level = 'Silver';
             END IF;
	END
 -----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



 -- Include pre-invoice deductions in Croma detailed report
	SELECT 
    	   s.date, 
           s.product_code, 
           p.product, 
	   p.variant, 
           s.sold_quantity, 
           g.gross_price as gross_price_per_item,
           ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
           pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_product p
            ON s.product_code=p.product_code
	JOIN fact_gross_price g
    	    ON g.fiscal_year=get_fiscal_year(s.date)
    	    AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
            ON pre.customer_code = s.customer_code AND
            pre.fiscal_year=get_fiscal_year(s.date)
	WHERE 
	    s.customer_code=90002002 AND 
    	    get_fiscal_year(s.date)=2021     
	LIMIT 1000000;

Performance Improvement # 1

-- creating dim_date and joining with this table and avoid using the function 'get_fiscal_year()' to reduce the amount of time taking to run the query
	SELECT 
    	    s.date, 
            s.customer_code,
            s.product_code, 
            p.product, p.variant, 
            s.sold_quantity, 
            g.gross_price as gross_price_per_item,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
            pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_date dt
        	ON dt.calendar_date = s.date
	JOIN dim_product p
        	ON s.product_code=p.product_code
	JOIN fact_gross_price g
    		ON g.fiscal_year=dt.fiscal_year
    		AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
        	ON pre.customer_code = s.customer_code AND
    		pre.fiscal_year=dt.fiscal_year
	WHERE 
    		dt.fiscal_year=2021     
	LIMIT 1500000;






### Module: Performance Improvement # 2

-- Added the fiscal year in the fact_sales_monthly table itself
	SELECT 
    	    s.date, 
            s.customer_code,
            s.product_code, 
            p.product, p.variant, 
            s.sold_quantity, 
            g.gross_price as gross_price_per_item,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
            pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_product p
        	ON s.product_code=p.product_code
	JOIN fact_gross_price g
    		ON g.fiscal_year=s.fiscal_year
    		AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
        	ON pre.customer_code = s.customer_code AND
    		pre.fiscal_year=s.fiscal_year
	WHERE 
    		s.fiscal_year=2021     
	LIMIT 1500000;

Database Views: Introduction

-- Get the net_invoice_sales amount using the CTE's
	WITH cte1 AS (
		SELECT 
    		    s.date, 
    		    s.customer_code,
    		    s.product_code, 
                    p.product, p.variant, 
                    s.sold_quantity, 
                    g.gross_price as gross_price_per_item,
                    ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
                    pre.pre_invoice_discount_pct
		FROM fact_sales_monthly s
		JOIN dim_product p
        		ON s.product_code=p.product_code
		JOIN fact_gross_price g
    			ON g.fiscal_year=s.fiscal_year
    			AND g.product_code=s.product_code
		JOIN fact_pre_invoice_deductions as pre
        		ON pre.customer_code = s.customer_code AND
    			pre.fiscal_year=s.fiscal_year
		WHERE 
    			s.fiscal_year=2021) 
	SELECT 
      	    *, 
    	    (gross_price_total-pre_invoice_discount_pct*gross_price_total) as net_invoice_sales
	FROM cte1
	LIMIT 1500000;


-- Creating the view `sales_preinv_discount` and store all the data in like a virtual table
	CREATE  VIEW `sales_preinv_discount` AS
	SELECT 
    	    s.date, 
            s.fiscal_year,
            s.customer_code,
            c.market,
            s.product_code, 
            p.product, 
            p.variant, 
            s.sold_quantity, 
            g.gross_price as gross_price_per_item,
            ROUND(s.sold_quantity*g.gross_price,2) as gross_price_total,
            pre.pre_invoice_discount_pct
	FROM fact_sales_monthly s
	JOIN dim_customer c 
		ON s.customer_code = c.customer_code
	JOIN dim_product p
        	ON s.product_code=p.product_code
	JOIN fact_gross_price g
    		ON g.fiscal_year=s.fiscal_year
    		AND g.product_code=s.product_code
	JOIN fact_pre_invoice_deductions as pre
        	ON pre.customer_code = s.customer_code AND
    		pre.fiscal_year=s.fiscal_year

-- Now generate net_invoice_sales using the above created view "sales_preinv_discount"
	SELECT 
            *,
    	    (gross_price_total-pre_invoice_discount_pct*gross_price_total) as net_invoice_sales
	FROM gdb0041.sales_preinv_discount







### Module: Database Views: Post Invoice Discount, Net Sales

-- Create a view for post invoice deductions: `sales_postinv_discount`
	CREATE VIEW `sales_postinv_discount` AS
	SELECT 
    	    s.date, s.fiscal_year,
            s.customer_code, s.market,
            s.product_code, s.product, s.variant,
            s.sold_quantity, s.gross_price_total,
            s.pre_invoice_discount_pct,
            (s.gross_price_total-s.pre_invoice_discount_pct*s.gross_price_total) as net_invoice_sales,
            (po.discounts_pct+po.other_deductions_pct) as post_invoice_discount_pct
	FROM sales_preinv_discount s -- it is a view which we created before recently.
	JOIN fact_post_invoice_deductions po
		ON po.customer_code = s.customer_code AND
   		po.product_code = s.product_code AND
   		po.date = s.date;

-- Create a report for net sales
	SELECT 
            *, 
    	    net_invoice_sales*(1-post_invoice_discount_pct) as net_sales
	FROM gdb0041.sales_postinv_discount;

-- Finally creating the view `net_sales` which inbuiltly use/include all the previous created view and gives the final result
	CREATE VIEW `net_sales` AS
	SELECT 
            *, 
    	    net_invoice_sales*(1-post_invoice_discount_pct) as net_sales
	FROM gdb0041.sales_postinv_discount;






### Module: Top Markets and Customers 

-- Get top 5 market by net sales in fiscal year 2021
	SELECT 
    	    market, 
            round(sum(net_sales)/1000000,2) as net_sales_mln
	FROM gdb0041.net_sales
	where fiscal_year=2021
	group by market
	order by net_sales_mln desc
	limit 5

-- Stored proc to get top n markets by net sales for a given year
	CREATE PROCEDURE `get_top_n_markets_by_net_sales`(
        	in_fiscal_year INT,
    		in_top_n INT
	)
	BEGIN
        	SELECT 
                     market, 
                     round(sum(net_sales)/1000000,2) as net_sales_mln
        	FROM net_sales
        	where fiscal_year=in_fiscal_year
        	group by market
        	order by net_sales_mln desc
        	limit in_top_n;
	END

-- stored procedure that takes market, fiscal_year and top n as an input and returns top n customers by net sales in that given fiscal year and market
	CREATE PROCEDURE `get_top_n_customers_by_net_sales`(
        	in_market VARCHAR(45),
        	in_fiscal_year INT,
    		in_top_n INT
	)
	BEGIN
        	select 
                     customer, 
                     round(sum(net_sales)/1000000,2) as net_sales_mln
        	from net_sales s
        	join dim_customer c
                on s.customer_code=c.customer_code
        	where 
		    s.fiscal_year=in_fiscal_year 
		    and s.market=in_market
        	group by customer
        	order by net_sales_mln desc
        	limit in_top_n;
	END




   
### Module: Window Functions: OVER Clause

-- show % of total expense
	select 
             *,
    	     amount*100/sum(amount) over() as pct
	from random_tables.expenses 
	order by category;

-- show % of total expense per category
	select 
            *,
    	    amount*100/sum(amount) over(partition by category) as pct
	from random_tables.expenses 
	order by category,  pct desc;

-- Show expenses per category till date
	select 
             *,
             sum(amount) over(partition by category order by date) as expenses_till_date
	from random_tables.expenses;







### Module: Window Functions: Using it In a Task

-- find out customer wise net sales percentage contribution 
	with cte1 as (
		select 
                    customer, 
                    round(sum(net_sales)/1000000,2) as net_sales_mln
        	from net_sales s
        	join dim_customer c
                    on s.customer_code=c.customer_code
        	where s.fiscal_year=2021
        	group by customer)
	select 
            *,
            net_sales_mln*100/sum(net_sales_mln) over() as pct_net_sales
	from cte1
	order by net_sales_mln desc





### Module: Exercise: Window Functions: OVER Clause

-- Find customer wise net sales distibution per region for FY 2021
	with cte1 as (
		select 
        	    c.customer,
                    c.region,
                    round(sum(net_sales)/1000000,2) as net_sales_mln
                from gdb0041.net_sales n
                join dim_customer c
                    on n.customer_code=c.customer_code
		where fiscal_year=2021
		group by c.customer, c.region)
	select
             *,
             net_sales_mln*100/sum(net_sales_mln) over (partition by region) as pct_share_region
	from cte1
	order by region, pct_share_region desc




### Module: Window Functions: ROW_NUMBER, RANK, DENSE_RANK

-- Show top 2 expenses in each category
	select * from 
	     (select 
                  *, 
    	          row_number() over (partition by category order by amount desc) as row_num
	      from random_tables.expenses) x
	where x.row_num<3

--  If two items have same expense then row_number doesnt work. We need a true rank for which we need to use either a rank or dense_rank() function.(demo using student_marks table)
	select 
	     *,
             row_number() over (order by marks desc) as row_num,
             rank() over (order by marks desc) as rank_num,
             dense_rank() over (order by marks desc) as dense_rank_num
	from random_tables.student_marks;

-- Find out top 3 products from each division by total quantity sold in a given year
	with cte1 as 
		(select
                     p.division,
                     p.product,
                     sum(sold_quantity) as total_qty
                from fact_sales_monthly s
                join dim_product p
                      on p.product_code=s.product_code
                where fiscal_year=2021
                group by p.product),
           cte2 as 
	        (select 
                     *,
                     dense_rank() over (partition by division order by total_qty desc) as drnk
                from cte1)
	select * from cte2 where drnk<=3

-- Creating stored procedure for the above query
	CREATE PROCEDURE `get_top_n_products_per_division_by_qty_sold`(
        	in_fiscal_year INT,
    		in_top_n INT
	)
	BEGIN
	     with cte1 as (
		   select
                       p.division,
                       p.product,
                       sum(sold_quantity) as total_qty
                   from fact_sales_monthly s
                   join dim_product p
                       on p.product_code=s.product_code
                   where fiscal_year=in_fiscal_year
                   group by p.product),            
             cte2 as (
		   select 
                        *,
                        dense_rank() over (partition by division order by total_qty desc) as drnk
                   from cte1)
	     select * from cte2 where drnk <= in_top_n;
	END


---------------------------------------------------------------------------------------------------------------------------------------------------------------------
Chapter:- SQL Advanced: Supply Chain Analytics



### Module: Create a Helper Table

-- Create fact_act_est table
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





### Module: Database Triggers

-- create the trigger to automatically insert record in fact_act_est table whenever insertion happens in fact_sales_monthly 
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

-- create the trigger to automatically insert record in fact_act_est table whenever insertion happens in fact_forecast_monthly 
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

-- To see all the Triggers
        show triggers;

-- Insert the records in the fact_sales_monthly and fact_forecast_monthly tables and check whether records inserted in fact_act_est table
	insert into fact_sales_monthly
              (date, product_code, customer_code, sold_quantity)
	values 
	      ("2030-09-01", "HAHA", 99, 89);

	insert into fact_forecast_monthly
             (date, product_code, customer_code, forecast_quantity)
	values 
	      ("2030-09-01", "HAHA", 99, 43);

	select * from fact_act_est where customer_code = 99;







### Module: Database Events

-- To show all the events
	show events;

-- Show variable which have event in it
	show variables like "%event%";

-- Creating the table "session_logs" in the random table and also insert the records in it
	CREATE TABLE random_tables.session_logs (`ts` DATETIME, `session_id` INT, `user_id` INT, `log` TEXT);
	INSERT INTO `random_tables`.`session_logs` 
                (`ts`, `session_id`, `user_id`, `log`) 
	VALUES 
            	('2022-10-04 08:14:07', '898812', '523', 'CLICKED | Courses Buttom'),
        	('2022-10-14 08:18:35', '898812', '523', 'NAVIAGE BACK | Python course page , codebasics.io'),
        	('2022-10-16 12:07:00', '965345', '523', 'REVIEW GENERATED | Data analytics in power bi'),
        	('2022-10-22 14:09:22', '188567', '707', 'NEW LOGIN | New login, user name: tasty@jalebi.com'),
        	('2022-10-22 18:10:06', '188567', '707', 'COURSE PURCHASED | Data analytics in power bi, user name: tasty@jalebi.com');

-- Delete logs that are less than 5 days old
	delimiter |
	CREATE EVENT e_daily_log_purge
    	ON SCHEDULE
      	EVERY 5 SECOND
    	COMMENT 'Purge logs that are more than 5 days old'
    	DO
      	     BEGIN
        	delete from random_tables.session_logs 
        	where DATE(ts) < DATE("2022-10-22") - interval 5 day;
      	     END |
        delimiter ;

-- drop the event
       drop event if exists e_daily_log_purge;






### Module: Temporary Tables & Forecast Accuracy Report

-- Forecast accuracy report using cte (It exists at the scope of statements)
	with forecast_err_table as (
             select
                  s.customer_code as customer_code,
                  c.customer as customer_name,
                  c.market as market,
                  sum(s.sold_quantity) as total_sold_qty,
                  sum(s.forecast_quantity) as total_forecast_qty,
                  sum(s.forecast_quantity-s.sold_quantity) as net_error,
                  round(sum(s.forecast_quantity-s.sold_quantity)*100/sum(s.forecast_quantity),1) as net_error_pct,
                  sum(abs(s.forecast_quantity-s.sold_quantity)) as abs_error,
                  round(sum(abs(s.forecast_quantity-sold_quantity))*100/sum(s.forecast_quantity),2) as abs_error_pct
             from fact_act_est s
             join dim_customer c
             on s.customer_code = c.customer_code
             where s.fiscal_year=2021
             group by customer_code
	)
	select 
            *,
            if (abs_error_pct > 100, 0, 100.0 - abs_error_pct) as forecast_accuracy
	from forecast_err_table
        order by forecast_accuracy desc;

-- Write a stored proc for the same
	CREATE PROCEDURE `get_forecast_accuracy`(
        	in_fiscal_year INT
	)
	BEGIN
		with forecast_err_table as (
             	       select
                           s.customer_code as customer_code,
                           c.customer as customer_name,
                           c.market as market,
                           sum(s.sold_quantity) as total_sold_qty,
                           sum(s.forecast_quantity) as total_forecast_qty,
                           sum(s.forecast_quantity-s.sold_quantity) as net_error,
                           round(sum(s.forecast_quantity-s.sold_quantity)*100/sum(s.forecast_quantity),1) as net_error_pct,
                           sum(abs(s.forecast_quantity-s.sold_quantity)) as abs_error,
                           round(sum(abs(s.forecast_quantity-sold_quantity))*100/sum(s.forecast_quantity),2) as abs_error_pct
             	       from fact_act_est s
             	       join dim_customer c
                       on s.customer_code = c.customer_code
                       where s.fiscal_year=in_fiscal_year
                       group by customer_code
	        )
	        select 
                    *,
                    if (abs_error_pct > 100, 0, 100.0 - abs_error_pct) as forecast_accuracy
	        from forecast_err_table
                order by forecast_accuracy desc;
	END

-- Forecast accuracy report using temporary table (It exists for the entire session)
	drop table if exists forecast_err_table;
	create temporary table forecast_err_table
             select
                  s.customer_code as customer_code,
                  c.customer as customer_name,
                  c.market as market,
                  sum(s.sold_quantity) as total_sold_qty,
                  sum(s.forecast_quantity) as total_forecast_qty,
                  sum(s.forecast_quantity-s.sold_quantity) as net_error,
                  round(sum(s.forecast_quantity-s.sold_quantity)*100/sum(s.forecast_quantity),1) as net_error_pct,
                  sum(abs(s.forecast_quantity-s.sold_quantity)) as abs_error,
                  round(sum(abs(s.forecast_quantity-sold_quantity))*100/sum(s.forecast_quantity),2) as abs_error_pct
             from fact_act_est s
             join dim_customer c
             on s.customer_code = c.customer_code
             where s.fiscal_year=2021
             group by customer_code;

	select 
            *,
            if (abs_error_pct > 100, 0, 100.0 - abs_error_pct) as forecast_accuracy
	from forecast_err_table
        order by forecast_accuracy desc;
	
	
	

### Module: User Accounts and Privileges

-- Show all grants available for a particular user(wanda)
	show grants for 'wanda';

-- Create a new user 'thor' 
	create user 'thor'@'localhost' identified by 'thor';

-- Allow certain access to 'thor' user for the database 'gdb041'
     	grant select on gdb041.dim_customer to 'thor'@'localhost';
     	grant select on gdb041.dim_product to 'thor'@'localhost';
     	grant execute on procedure gdb041.get_forecast_accuracy_report to 'thor'@'localhost';

-- See all the access for 'thor' user
	show grants for 'thor'@'localhost';




### Module: Database Indexes: Index Types (make sakila database as default one)
	
-- Query1
	select * from film where description like "%car%" or "%boat%";

-- Query2
	select * from sakila.film 
	where match(description) against("car boat")
	limit 1000

-- Query3
	select * from sakila.film 
	where match(description) against("car -boat" in boolean mode)
	limit 1000
