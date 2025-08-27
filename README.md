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
