# Part 4: Starting with Data

Consider with the data available, write 3 - 5 new questions that could be answer with this database.

**Question 1: What is the total unique visitors and the avergage time that a unique visitor spend on the site?**

SQL Queries:

```SQL
SELECT
  COUNT(DISTINCT fullVisitorId) AS unique_visitors,
  SUM(timeOnSite)/ COUNT(DISTINCT fullVisitorId) AS avg_timeOnSite
FROM all_sessions;
```


Answer: There are 14223 unique visitors and the avergage time that a unique visitor spend on the site is 3 minutes 6 seconds.


**Question2: List all the unique product names available in the merchandise store alphabetically**

SQL Queries:

```SQL
SELECT
  DISTINCT v2ProductName AS product_name
FROM all_sessions
ORDER BY product_name;
```

Answer: There 471 rows as in the output.


**Question 3: What is the total unique visitors by the referring site (channelGrouping) in the descreasing order of total unique visitors. Which top 3 referring sites are the most effective in attracting unique visitors?** 

SQL Queries:

```SQL
SELECT
  channelGrouping,
  COUNT(DISTINCT fullVisitorId) AS unique_visitors
FROM all_sessions
GROUP BY channelGrouping
ORDER BY unique_visitors DESC;
```


Answer: Organic search leads bring in the total amount of unique visitors to the site with 8207 persons, followed by Direct and Referral
(2805 and 2419 unique visitors).


**Question 4: List all the eCommerceActiontype and the number of unique visitors associated with each type.**

SQL Queries:

```SQL
SELECT
  eCommerceAction_type,
  COUNT(DISTINCT fullVisitorId) AS unique_visitors_num
FROM all_sessions
GROUP BY eCommerceAction_type
ORDER BY eCommerceAction_type;
```

Answer: The output shows a pattern of of most unique vistors at eCommerceAction_type 0, and abruptly shrinking to much less unique visitors in eCommerceAction_type 1-6.


**Question 5: What are the top five products with the lowest average sentimentscore per product**

SQL Queries:

```SQL
SELECT
  p.productname, ROUND(AVG(s.sentimentScore), 2) AS avg_sentimentScore
FROM sales_report s
JOIN products p USING (productsku)
GROUP BY p.productname
ORDER BY AVG(s.sentimentScore)
LIMIT 10;
```

Answer: Products with the lowest average sentimentscore are: Men's Vintage Henley, Android Women's Short Sleeve Badge Tee Dark Heather, Android Men's Long & Lean Badge Tee Charcoal, Clip-on Compact Charger, High Capacity 10,400mAh Charger with the average sentimentScore all < -0.2.
