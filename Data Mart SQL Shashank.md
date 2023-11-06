## :shopping_cart: Case Study #5: Data Mart - Data Exploration based on the 8-week-sql challenge.

## Case Study Questions
1. What day of the week is used for each week_date value?
2. What range of week numbers are missing from the dataset?
3. How many total transactions were there for each year in the dataset?
4. What is the total sales for each region for each month?
5. What is the total count of transactions for each platform
6. What is the percentage of sales for Retail vs Shopify for each month?
7. What is the percentage of sales by demographic for each year in the dataset?
8. Which age_band and demographic values contribute the most to Retail sales?
9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

***

###  1. What day of the week is used for each week_date value?

```sql
SELECT DISTINCT dayname(week_date) AS day_of_week
FROM clean_weekly_sales;
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190210286-867b6b69-e9b8-41a9-845f-ac659ae21766.png)

***

###  2. What range of week numbers are missing from the dataset? 
- To get the current value of default_week_format variable : SHOW VARIABLES LIKE 'default_week_format';

```sql
-- Range 0 to 52

SELECT DISTINCT week(week_date) AS week_number
FROM clean_weekly_sales
ORDER BY week(week_date) ASC;

-- Missing week numbers: Week 1 to 11 and week 36 to 52
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190210714-2f2acc7d-2ec8-4343-af85-df737dc5944b.png)

***

###  3. How many total transactions were there for each year in the dataset?

```sql
SELECT year(week_date) AS YEAR,
       sum(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY year(week_date)
ORDER BY 1;
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190211018-64eff211-6485-46f1-9718-b0d71ed001b5.png)

***

###  4. What is the total sales for each region for each month?

```sql
SELECT region,
       month_number,
       monthname(week_date) as month_name,
       sum(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY region,
         month_number
ORDER BY 1,
         2;
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190211522-6141322a-be42-4211-8fec-72ec02c9e6d7.png)

***

###  5. What is the total count of transactions for each platform 

```sql
SELECT platform,
       sum(transactions) AS transactions_count
FROM clean_weekly_sales
GROUP BY 1;
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190211704-dc469d8b-06e9-4a57-aaff-7c969352c2a4.png)

***

###  6. What is the percentage of sales for Retail vs Shopify for each month?

Using GROUP BY and WINDOW FUNCTION
```sql
WITH sales_contribution_cte AS
  (SELECT calendar_year,
          month_number,
          platform,
          sum(sales) AS sales_contribution
   FROM clean_weekly_sales
   GROUP BY 1,
            2,
            3
   ORDER BY 1,
            2),
     total_sales_cte AS
  (SELECT *,
          sum(sales_contribution) over(PARTITION BY calendar_year, month_number) AS total_sales
   FROM sales_contribution_cte)
SELECT calendar_year,
       month_number,
       ROUND(sales_contribution/total_sales*100, 2) AS retail_percent,
       100-ROUND(sales_contribution/total_sales*100, 2) AS shopify_percent
FROM total_sales_cte
WHERE platform = "Retail"
ORDER BY 1,
         2;
``` 

Using GROUP BY AND CASE statements
```sql
WITH sales_cte AS
  (SELECT calendar_year,
          month_number,
          SUM(CASE
                  WHEN platform="Retail" THEN sales
              END) AS retail_sales,
          SUM(CASE
                  WHEN platform="Shopify" THEN sales
              END) AS shopify_sales,
          sum(sales) AS total_sales
   FROM clean_weekly_sales
   GROUP BY 1,
            2
   ORDER BY 1,
            2)
SELECT calendar_year,
       month_number,
       ROUND(retail_sales/total_sales*100, 2) AS retail_percent,
       ROUND(shopify_sales/total_sales*100, 2) AS shopify_percent
FROM sales_cte;
``` 
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190578851-5544de41-08cc-401f-adf6-12304e2c78a4.png)


***

###  7. What is the percentage of sales by demographic for each year in the dataset?

```sql
WITH sales_contribution_cte AS
  (SELECT calendar_year,
          demographic,
          sum(sales) AS sales_contribution
   FROM clean_weekly_sales
   GROUP BY 1,
            2
   ORDER BY 1),
     total_sales_cte AS
  (SELECT *,
          sum(sales_contribution) over(PARTITION BY calendar_year) AS total_sales
   FROM sales_contribution_cte)
SELECT calendar_year,
       demographic,
       ROUND(100*sales_contribution/total_sales, 2) AS percent_sales_contribution
FROM total_sales_cte
GROUP BY 1,
         2;
``` 

#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190579915-c6613674-730b-4611-9fdf-680a6dabc431.png)

```sql
WITH sales_cte AS
  (SELECT calendar_year,
          SUM(CASE
                  WHEN demographic="Couples" THEN sales
              END) AS couple_sales,
          SUM(CASE
                  WHEN demographic="Families" THEN sales
              END) AS family_sales,
          SUM(CASE
                  WHEN demographic="unknown" THEN sales
              END) AS unknown_sales,
          sum(sales) AS total_sales
   FROM clean_weekly_sales
   GROUP BY 1
   ORDER BY 1)
SELECT calendar_year,
       ROUND(couple_sales/total_sales*100, 2) AS couple_percent,
       ROUND(family_sales/total_sales*100, 2) AS family_percent,
       ROUND(unknown_sales/total_sales*100, 2) AS unknown_percent
FROM sales_cte;
``` 
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190579310-e5844171-9d75-46a9-b61f-eb14e32c69ca.png)


***

###  8. Which age_band and demographic values contribute the most to Retail sales?

```sql
SELECT age_band,
       demographic,
       ROUND(100*sum(sales)/
               (SELECT SUM(sales)
                FROM clean_weekly_sales
                WHERE platform="Retail"), 2) AS retail_sales_percentage
FROM clean_weekly_sales
WHERE platform="Retail"
GROUP BY 1,
         2
ORDER BY 3 DESC;
``` 
	
#### Result set:


***

###  9. Can we use the avg_transaction column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?

Let's try this mathematically.
Consider average of (4,4,4,4,4,4) = (4*6)/6 = 4 and average(5) = 5
Average of averages = (4+5)/2 = 4.5
Average of all numbers = (24+5)/ = 4.1428

Hence, we can not use avg_transaction column to find the average transaction size for each year and sales platform, because the result will be incorrect if we calculate average of an average to calculate the average.

```sql
SELECT calendar_year,
       platform,
       ROUND(SUM(sales)/SUM(transactions), 2) AS correct_avg,
       ROUND(AVG(avg_transaction), 2) AS incorrect_avg
FROM clean_weekly_sales
GROUP BY 1,
         2
ORDER BY 1,
         2;
```

 C. Before & After Analysis

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the `week_date` value of `2020-06-15` as the baseline week where the Data Mart sustainable packaging changes came into effect. We would include all `week_date` values for `2020-06-15` as the start of the period after the change and the previous week_date values would be before.

Using this analysis approach - answer the following questions:

**1. What is the total sales for the 4 weeks before and after `2020-06-15`? What is the growth or reduction rate in actual values and percentage of sales?**

Before we proceed, we determine the the week_number corresponding to '2020-06-15' to use it as a filter in our analysis. 

````sql
SELECT DISTINCT week_number
FROM clean_weekly_sales
WHERE week_date = '2020-06-15' 
  AND calendar_year = '2020';
````

|week_number|
|:----|
|25|
 
The `week_number` is 25. I created 2 CTEs:
- `packaging_sales` CTE: Filter the dataset for 4 weeks before and after `2020-06-15` and calculate the sum of sales within the period.
- `before_after_changes` CTE: Utilize a `CASE` statement to capture the sales for 4 weeks before and after `2020-06-15` and then calculate the total sales for the specified period.

````sql
WITH packaging_sales AS (
  SELECT 
    week_date, 
    week_number, 
    SUM(sales) AS total_sales
  FROM clean_weekly_sales
  WHERE (week_number BETWEEN 21 AND 28) 
    AND (calendar_year = 2020)
  GROUP BY week_date, week_number
)
, before_after_changes AS (
  SELECT 
    SUM(CASE 
      WHEN week_number BETWEEN 21 AND 24 THEN total_sales END) AS before_packaging_sales,
    SUM(CASE 
      WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS after_packaging_sales
  FROM packaging_sales
)

SELECT 
  after_packaging_sales - before_packaging_sales AS sales_variance, 
  ROUND(100 * 
    (after_packaging_sales - before_packaging_sales) 
    / before_packaging_sales,2) AS variance_percentage
FROM before_after_changes;
````

**Answer:**

|sales_variance|variance_percentage|
|:----|:----|
|-26884188|-1.15|

Since the implementation of the new sustainable packaging, there has been a decrease in sales amounting by $26,884,188 reflecting a negative change at 1.15%. Introducing a new packaging does not always guarantee positive results as customers may not readily recognise your product on the shelves due to the change in packaging.

***

**2. What about the entire 12 weeks before and after?**

We can apply a similar approach and solution to address this question. 

````sql
WITH packaging_sales AS (
  SELECT 
    week_date, 
    week_number, 
    SUM(sales) AS total_sales
  FROM clean_weekly_sales
  WHERE (week_number BETWEEN 13 AND 37) 
    AND (calendar_year = 2020)
  GROUP BY week_date, week_number
)
, before_after_changes AS (
  SELECT 
    SUM(CASE 
      WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS before_packaging_sales,
    SUM(CASE 
      WHEN week_number BETWEEN 25 AND 37 THEN total_sales END) AS after_packaging_sales
  FROM packaging_sales
)

SELECT 
  after_packaging_sales - before_packaging_sales AS sales_variance, 
  ROUND(100 * 
    (after_packaging_sales - before_packaging_sales) / before_packaging_sales,2) AS variance_percentage
FROM before_after_changes;
````

**Answer:**

|sales_variance|variance_percentage|
|:----|:----|
|-152325394|-2.14|

Looks like the sales have experienced a further decline, now at a negative 2.14%! If I'm Danny's boss, I wouldn't be too happy with the results.

***

**3. How do the sale metrics for these 2 periods before and after compare with the previous years in 2018 and 2019?**

I'm breaking down this question to 2 parts.

**Part 1: How do the sale metrics for 4 weeks before and after compare with the previous years in 2018 and 2019?**
- Basically, the question is asking us to find the sales variance between 4 weeks before and after `'2020-06-15'` for years 2018, 2019 and 2020. Perhaps we can find a pattern here.
- We can apply the same solution as above and add `calendar_year` into the syntax. 

````sql
WITH changes AS (
  SELECT 
    calendar_year,
    week_number, 
    SUM(sales) AS total_sales
  FROM clean_weekly_sales
  WHERE week_number BETWEEN 21 AND 28
  GROUP BY calendar_year, week_number
)
, before_after_changes AS (
  SELECT 
    calendar_year,
    SUM(CASE 
      WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS before_packaging_sales,
    SUM(CASE 
      WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS after_packaging_sales
  FROM changes
  GROUP BY calendar_year
)

SELECT 
  calendar_year, 
  after_packaging_sales - before_packaging_sales AS sales_variance, 
  ROUND(100 * 
    (after_packaging_sales - before_packaging_sales) 
    / before_packaging_sales,2) AS variance_percentage
FROM before_after_changes;
````

**Answer:**

|calendar_year|sales_variance|variance_percentage|
|:----|:----|:----|
|2018|4102105|0.19|
|2019|2336594|0.10|
|2020|-26884188|-1.15|

In 2018, there was a sales variance of $4,102,105, indicating a positive change of 0.19% compared to the period before the packaging change.

Similarly, in 2019, there was a sales variance of $2,336,594, corresponding to a positive change of 0.10% when comparing the period before and after the packaging change.

However, in 2020, there was a substantial decrease in sales following the packaging change. The sales variance amounted to $26,884,188, indicating a significant negative change of -1.15%. This reduction represents a considerable drop compared to the previous years.

**Part 2: How do the sale metrics for 12 weeks before and after compare with the previous years in 2018 and 2019?**
- Use the same solution above and change to week 13 to 24 for before and week 25 to 37 for after.

````sql
WITH changes AS (
  SELECT 
    calendar_year, 
    week_number, 
    SUM(sales) AS total_sales
  FROM clean_weekly_sales
  WHERE week_number BETWEEN 13 AND 37
  GROUP BY calendar_year, week_number
)
, before_after_changes AS (
  SELECT 
    calendar_year,
    SUM(CASE 
      WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS before_packaging_sales,
    SUM(CASE 
      WHEN week_number BETWEEN 25 AND 37 THEN total_sales END) AS after_packaging_sales
  FROM changes
  GROUP BY calendar_year
)

SELECT 
  calendar_year, 
  after_packaging_sales - before_packaging_sales AS sales_variance, 
  ROUND(100 * 
    (after_packaging_sales - before_packaging_sales) 
    / before_packaging_sales,2) AS variance_percentage
FROM before_after_changes;
````

**Answer:**

|calendar_year|sales_variance|variance_percentage|
|:----|:----|:----|
|2018|104256193|1.63|
|2019|-20740294|-0.30|
|2020|-152325394|-2.14|

There was a fair bit of percentage differences in all 3 years. However, now when you compare the worst year to their best year in 2018, the sales percentage difference is even more stark at a difference of 3.77% (1.63% + 2.14%).

When comparing the sales performance across all three years, there were noticeable variations in the percentage differences. However, the most significant contrast emerges when comparing the worst-performing year in 2020 to the best-performing year in 2018. In this comparison, the sales percentage difference becomes even more apparent with a significant gap of 3.77% (1.63% + 2.14%).

***
	
#### Result set:
![image](https://user-images.githubusercontent.com/77529445/190840979-c7e92b09-898d-43d5-a9f6-8f12394fabbe.png)


***
