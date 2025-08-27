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
