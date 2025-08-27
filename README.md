-- step 1 
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
