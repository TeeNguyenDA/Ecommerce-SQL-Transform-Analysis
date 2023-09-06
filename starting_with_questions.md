# Part 3: Starting with Questions

This database provides data on revenue by product as well as data on how each visitor to the site interacted with the products (when they visited, where they were based, how many pages they viewed, how long they stayed on the site, and the number of transactions they entered).

In the starting_with_questions.md file there are 5 questions to answer using the data. Each question includes: The queries used to answer the question, and the answer to the question

    
**Question 1: Which cities and countries have the highest level of transaction revenues on the site?**

SQL Queries:
```SQL
SELECT 
	city, country,
	SUM(totalTransactionRevenue) AS totalTransactionRevenue
FROM all_sessions
WHERE 
	country != '(not set)' AND city != 'not available in demo dataset'
	AND city != '(not set)' 
	AND totalTransactionRevenue IS NOT NULL
GROUP BY city, country
ORDER BY SUM(totalTransactionRevenue) DESC
LIMIT 10;
```

Answer: The country and city with have the highest level of transaction revenues on the site is the United States/ San Francisco with the totalTransactionRevenue is 1564.32.


**Question 2: What is the average number of products ordered from visitors in each city and country?**


SQL Queries:
```SQL
SELECT 
	als.country, als.city, 
	ROUND(AVG(p.orderedQuantity)) AS avg_orderedQuantity
FROM all_sessions als
JOIN products p USING(productSKU)
WHERE 
	country != '(not set)' AND city != 'not available in demo dataset'
	AND city != '(not set)' 
GROUP BY als.country, als.city
ORDER BY als.country, als.city;
```

Answer: The output returns 268 rows with a combination of city, country and the corresponding average ordered quantity.


**Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**


SQL Queries:
```SQL
SELECT DISTINCT als.country,
       als.v2ProductCategory AS product_category,
	   ROUND(AVG(orderedQuantity)) AS avg_orderedQuantity
FROM all_sessions als
RIGHT JOIN products p USING(productSKU)
WHERE 
	als.country != '(not set)' AND als.city != 'not available in demo dataset'
	AND als.city != '(not set)'
GROUP BY als.country, als.v2ProductCategory
ORDER BY als.country, avg_orderedQuantity DESC, als.v2ProductCategory;


SELECT DISTINCT als.country, als.city,
       als.v2ProductCategory AS product_category,
	   ROUND(AVG(orderedQuantity)) AS avg_orderedQuantity
FROM all_sessions als
RIGHT JOIN products p USING(productSKU)
WHERE 
	als.country != '(not set)' AND als.city != 'not available in demo dataset'
	AND als.city != '(not set)'
GROUP BY als.country, als.city, als.v2ProductCategory
ORDER BY als.country, als.city, avg_orderedQuantity DESC, als.v2ProductCategory;
```


Answer: Due to the high volume of the number of cities and countries, let's limit our pattern finding to only the top country: the US. We use the average orderedquantity in stead of SUM, as we're comparing cities against the whole country differ in the their population sizes. The overall top 5 product category in the US based on products ordered are: Housewares, Nest, Lifestyle, Accessories and Office. However, each city in the US has a slight deviation from the country's top 5 product categories (based on average number of orders).

For example: San Francisco has Lifestyle, Nest, Office, Drinkware, Accessories ranked high in the top 5 product category whereas New York has Nest, Accessories, Office, Lifestyle and Others.


**Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


SQL Queries:
```SQL
SELECT 
	als.city, als.country, als.productsku, als.v2productname, SUM(p.orderedquantity) AS total_quantity_sold
FROM all_sessions als
JOIN products p USING (productsku)
WHERE country != '(not set)' AND city != 'not available in demo dataset' AND city != '(not set)' 
	  AND als.v2productcategory != '(not set)' 
GROUP BY als.city, als.country, als.productsku, v2productcategory, als.v2productname
HAVING SUM(p.orderedquantity) = (
    SELECT MAX(quantity_sold)
    FROM (
    SELECT city, country, productsku, SUM(p.orderedquantity) AS quantity_sold
    FROM all_sessions als
    GROUP BY city, country, productsku
) AS sq 
WHERE sq.city = als.city AND sq.country = als.country AND sq.productsku = als.productsku
)
ORDER BY als.country, als.city, total_quantity_sold DESC, als.productsku, als.v2productname;
```


Answer: The output returns 3997 rows. Due to the high volume of the combination of city, country, productsku and productname. The checking of any pattern in the products sold per city and country will be too big of a scope. If we rearrange our query above to order by the highest numbers of the products sold, we can observe that the Nest Cam Indoor Security Camera product and Google Kickball sell really well in terms of quantity in the US and other countries.


**Question 5: Can we summarize the impact of revenue generated from each city/country?**

SQL Queries:
```SQL
SELECT city, country,
	   ROUND(100 * totalTransactionRevenue / SUM(totalTransactionRevenue) OVER (PARTITION BY country), 2) AS rev_share
FROM (SELECT 
		city, country,
		SUM(totalTransactionRevenue) AS totalTransactionRevenue
	  FROM all_sessions
	  WHERE 
		country != '(not set)' AND city != 'not available in demo dataset'
		AND city != '(not set)' 
		AND totalTransactionRevenue IS NOT NULL
	  GROUP BY city, country
	  ORDER BY SUM(totalTransactionRevenue) DESC) AS sq
ORDER BY country, rev_share DESC;
```


Answer: Built upon the query in question 1, San Francisco, Sunnyvale and Atlanta altogether are the most impactful cities based on transaction revenue generated in the US (accounting for nearly 50%).
