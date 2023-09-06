#Part 1 and 2: Loading csv Files into Database and Cleaning the data

##Introduction:

Given 5 separate datasets (`all_sessions`, `analytics`, `products`, `sales_by_sku`, sales_report), our goal is to create a new PostgreSQL database called `ecommerce` by setting up tables and importing data into the tables for each .csv file. Then, we will perform data cleaning and transformation on the `ecommerce` database prior to answering any possible business problems in the dataset.

###Section 1: Loading csv Files into Database

The first step involves filtering and combining the datasets into a single workable format for analysis:

1. Create the `ecommerce` database and the tables with corresponding columns to our 5 datasets in PostgreSQL using pdadmin4 and the ```CREATE TABLE`` statement.
2. Import the csv files into the table using ```COPY``` statement.
We haven't set up the constraints and the keys at this stage, because there are some messy missing data and duplicates in some of the unique identifier columns.


###Section 2: Data Cleaning and Transforming

The overall approach to clean the data across our 5 tables:

- Remove irrelevant, duplicated data
- Fix structural errors
- Do type conversion
- Handle missing data

Let's perform the data cleaning process!

1. Clean and transform the `all_sessions` table

Before deciding to drop any irrelevant or missing data, it's better to summarize how many values there are in `all_sessions`.

```SQL
SELECT 
    j.column_name, COUNT(j.value)
FROM all_sessions a
	CROSS JOIN LATERAL jsonb_each_text(jsonb_strip_nulls(to_jsonb(a))) AS j(column_name, value)
GROUP BY j.column_name
ORDER BY j.column_name;
``` 

The return output shows that there are missing values in some of the columns, mostly the transaction related columns such as transactionid, transactionrevenue, totaltransactionrevenue, transactions. Missing values also scatter across the set of ecommerce, product columns and columns like ecommerceaction_option, sessionqualitydim.

|     column_name       |      count        |
| --------------------  |:----------------- |
|channelgrouping        |	15134           |
|city                   |	15134           |
|country                |	15134           |  
|currencycode           |	14862           |
|date                   |	15134           |
|ecommerceaction_option |	31              |
|ecommerceaction_step   |	15134           |
|ecommerceaction_type   |	15134           |
|fullvisitorid          |	15134           |
|pagepathlevel1         |	15134           |
|pagetitle              |	15133           |
|pageviews              |	15134           |
|productprice           |	15134           |
|productquantity        |	53              |
|productrevenue         |	4               |
|productsku             |	15134           |
|productvariant         |	15134           |
|sessionqualitydim      |	1228            |
|time                   |	15134           |
|timeonsite             |	11834           |
|totaltransactionrevenue|	81              |
|transactionid          |	9               |
|transactionrevenue     |	4               |
|transactions           |	81              |
|type                   |	15134           |
|v2productcategory      |	15134           |
|v2productname          |	15134           |
|visitid                |	15134           |

- Find full duplicate in all columns in the and drop them if any. The query results in 0 rows.

```SQL
SELECT
	fullVisitorId,
	channelGrouping,
	time,
	country,
	city,
	totalTransactionRevenue,
	transactions,
	timeOnSite,
	pageviews,
	sessionQualityDim ,
	date,
	visitId,
	type,
	productRefundAmount,
	productQuantity,
	productPrice,
	productRevenue,
	productSKU,
	v2ProductName,
	v2ProductCategory,
	productVariant,
	currencyCode,
	itemQuantity,
	itemRevenue,
	transactionRevenue,
	transactionId,
	pageTitle,
	searchKeyword,
	pagePathLevel1,
	eCommerceAction_type,
	eCommerceAction_step,
	eCommerceAction_option
FROM all_sessions
GROUP BY 
	fullVisitorId,
	channelGrouping,
	time,
	country,
	city,
	totalTransactionRevenue,
	transactions,
	timeOnSite,
	pageviews,
	sessionQualityDim,
	date,
	visitId,
	type,
	productRefundAmount,
	productQuantity,
	productPrice,
	productRevenue,
	productSKU,
	v2ProductName,
	v2ProductCategory,
	productVariant,
	currencyCode,
	itemQuantity,
	itemRevenue,
	transactionRevenue,
	transactionId,
	pageTitle,
	searchKeyword,
	pagePathLevel1,
	eCommerceAction_type,
	eCommerceAction_step,
	eCommerceAction_option
HAVING COUNT(*) > 1;
```

- fullvisitorid looks like a primary key candidate for the `all_sessions` table and there are 794 duplicates in this column. Let's investigate further

```SQL
SELECT fullVisitorId
FROM all_sessions
GROUP BY fullVisitorId
HAVING COUNT(*) > 1;
```
We might need to pick a combination of columns as the unique identifer for the `all_sessions` table, which can be (fullVisitorId, visitId, channelGrouping, productSKU) considering the relations to other tables in the database. Within the scope of this project, we won't emphasize on setting up the primary key and other constraints for each table. Our priority would just be defining the connection with other tables.

Next, the goal is to fix strange naming conventions, typos, or incorrect capitalization. We'll go through each column in this section.

- The fullVisitorId column: Was previously set as VARCHAR, fullVisitorId has 16 to 18 character length varying for each id which is beyond the capacity storage of BIGINT data type. On the otherhand, we need this column values format to be consistent and sortable. One solution is to keep data as VARCHAR and pad the leading zeros in front within an 18-character length range.

```SQL
UPDATE all_sessions
SET fullVisitorId = LPAD(FullVisitorId, 18, '0009999999999999999');
```
- The visitId column: Initially set as BIGINT data type, we choose to convert into VARCHAR for consistency with other id columns in the `all_sessions`table.

```SQL
ALTER TABLE all_sessions
ALTER COLUMN visitId TYPE VARCHAR USING visitId::VARCHAR;
```

- The productSKU column: Its data type as the same VARCHAR since there's a mix of numeric values and characters such as '9182569' or 'GGOEGOAJ021599'.

- The channelGrouping column: Includes these distinct values: 'Organic Search', 'Display', 'Referral', 'Affiliates', 'Direct', 'Paid Search', '(Other)'. Changing the text format of '(Other)' to remove the () is necessary and less confusing.

```SQL
--- View the unique values of channelGrouping
SELECT DISTINCT channelGrouping
FROM all_sessions;

--- Update '(Other)' to 'Other'
UPDATE all_sessions SET channelGrouping = BTRIM(channelGrouping,'()')
```

- The time column: The values here looks like Unix Timestamps with values ranging from 0 to 3192410. When we attempted to convert this column to timestamp without time zone, the result are dated from 1970-01-01 00:00:00 to 1970-02-06 22:46:50. We can drop the time column because it doesn't make sense when compared to the `date` column (with data from 2016-08-01 to 2017-08-01 below in the date column investigation). We also don't have any other source of information to fill in the data later on.

```SQL
--- Convert `time` column to timestamp
SELECT DISTINCT TO_TIMESTAMP(time::BIGINT) AT TIME ZONE 'UTC' AS time_converted
FROM all_sessions
ORDER BY time_converted;

--- Drop `time` column from the `all_sessions` table
ALTER TABLE all_sessions 
DROP COLUMN time;
```

- The timeOnSite column: To determine what data type to convert into, we'll find out the max and min time spent on the website. The maximum value is 4661 while the min is 1. Assuming our data is in seconds, that would be equal to a visitor spending 1 hr 17 min 40 sec at the longest. It's best to convert `timeOnSite` to INTERVAL.

```SQL
--- Check min and max value of `timeOnSite` in seconds:
SELECT 
	MIN(DISTINCT (timeOnSite::INT)) AS min_timeOnSite,
	MAX(DISTINCT (timeOnSite::INT)) AS max_timeOnSite
FROM all_sessions;

---  Convert `timeOnSite` to INTERVAL
ALTER TABLE all_sessions
ALTER COLUMN timeOnSite TYPE INTERVAL USING timeOnSite::INTERVAL;
```

- The totalTransactionRevenue, transactions, transactionRevenue, transactionId columns: Such columns fall within the same domain of transaction, we can analyze them together. The main issue here with transaction-related columns is a large amount of missing data. `totalTransactionRevenue` and `transactions` most likely come from the same source with 81 records. There are only 9 `transactionId`` and 4 `transactionRevenue` values. 

We may need to check if `totalTransactionRevenue` and `transactionRevenue` are reflecting the same metric. The output confirms the duplication which means dropping `transactionRevenue` won't have any impact on our table.

```SQL
--- Check if `totalTransactionRevenue` and `transactionRevenue` are duplicated
SELECT DISTINCT totaltransactionrevenue, transactionRevenue
FROM all_sessions;

--- Drop `transactionRevenue` column
ALTER TABLE all_sessions 
DROP COLUMN transactionRevenue;
```

All the revenue and price column values might have had a calculation error where a reusable shopping bag costs 3,500,000 USD and 50 of those bags make 1,015,480,000 USD in transaction revenue by 1 transaction. The transaction payment cap per transaction may go up to only several thousands USD for small value items in one single transaction. It's an indicator to reduce the amount by 1,000,000 for such monetary columns.

```SQL
--- Investigate the inflated value of `totalTransactionRevenue` and other justifying columns
SELECT
	DISTINCT transactionId, 
	transactions,
	totalTransactionRevenue,
	currencyCode,
	productPrice,
	productQuantity,
	v2ProductName
FROM all_sessions
WHERE transactionId IS NOT NULL;

--- Update new values in `totalTransactionRevenue` and `productPrice` by dividing by 1,000,000
UPDATE all_sessions
SET 
	totalTransactionRevenue = ROUND(totalTransactionRevenue/1000000, 2),
	productPrice = ROUND(productPrice/1000000, 2);
```

- The currencyCode column: We'll fill in the missing values in `currencyCode` with USD, as when grouping `country` and `currencyCode`, countries without currency code are already tagged as USD.

```SQL
UPDATE all_sessions 
	SET currencyCode = CASE 
					WHEN currencyCode IS NULL THEN 'USD'
					ELSE currencyCode
				  END;
```

- The productRefundAmount, productQuantity, productPrice, productRevenue, productSKU, v2ProductName, v2ProductCategory, productVariant columns:

Correct the productRevenue values by dividing productRevenue by 1,000,000 and calculate the productRefundAmount with all missing values based on the relationship with productRevenue, productQuantity and productPrice.

```SQL
--- Divide productRevenue / 1000000
UPDATE all_sessions
SET 
	productRevenue = ROUND(productRevenue/1000000, 2);

--- Calculate productRefundAmount = productRevenue - (productQuantity * productPrice)
UPDATE all_sessions
SET 
	productRefundAmount = productRevenue - (productQuantity * productPrice);
```

Group different categories tiers and variations together based mostly on the first tier of v2productcategory (Home/[Category]), to flatten the product category structure:

```SQL
UPDATE all_sessions
SET v2productcategory = CASE
						  WHEN v2productcategory = '${escCatTitle}'THEN 'Others'
						  WHEN v2productcategory = '(not set)'THEN 'Others'
						  WHEN v2productcategory LIKE '%Accessories%' THEN 'Accessories'
						  WHEN v2productcategory LIKE '%Apparel%' OR v2productcategory LIKE '%Wearables%' THEN 'Apparel'
						  WHEN v2productcategory LIKE '%Bags%' THEN 'Bags'
						  WHEN v2productcategory LIKE '%Brands%' OR v2productcategory LIKE '%Brand%' OR v2productcategory LIKE '%Youtube' THEN 'Brands'
						  WHEN v2productcategory LIKE '%Clearance Sale%' THEN 'Clearance Sale'
						  WHEN v2productcategory LIKE '%Drinkware%' OR v2productcategory LIKE '%Bottles%' THEN 'Drinkware'
						  WHEN v2productcategory LIKE '%Electronics%' OR v2productcategory LIKE '%Headgear%' THEN 'Electronics'
						  WHEN v2productcategory LIKE '%Lifestyle%' OR v2productcategory LIKE '%Fruit Games%' OR v2productcategory LIKE '%Fun%' THEN 'Lifestyle'
						  WHEN v2productcategory LIKE '%Gift Cards%' THEN 'Gift Cards'
						  WHEN v2productcategory LIKE '%Kids%' THEN 'Kids'
						  WHEN v2productcategory LIKE '%Spring Sale%' THEN 'Spring Sale'
						  WHEN v2productcategory LIKE '%Limited Supply%' THEN 'Limited Supply'
						  WHEN v2productcategory LIKE '%Nest%' THEN 'Nest'
						  WHEN v2productcategory LIKE '%Office%' THEN 'Office'
						  WHEN v2productcategory LIKE '%Housewares%' THEN 'Housewares'
						  ELSE v2productcategory
						END;
```
Drop the rest of unnecessary columns filled with all missing values: itemQuantity, itemRevenue, searchKeyword

```SQL
ALTER TABLE all_sessions 
DROP COLUMN itemQuantity,
DROP COLUMN itemRevenue,
DROP COLUMN searchKeyword;
```

- The eCommerceAction_step and eCommerceAction_option columns: There are NULL values in the eCommerceAction_option column that can be safely filled in with 'Billing and Shipping'. The reason for that is when we map eCommerceAction_step and eCommerceAction_option unique values, a matching pattern arises: 1) eCommerceAction_step = 1, eCommerceAction_option gets'Billing and Shipping' and NULL, 2) eCommerceAction_step = 2, eCommerceAction_option gets 'Payment', 3) eCommerceAction_step = 3, eCommerceAction_option gets 'Review'.

```SQL
UPDATE all_sessions
SET eCommerceAction_option = CASE 
								WHEN eCommerceAction_step = 1 AND eCommerceAction_option IS NULL THEN COALESCE(eCommerceAction_option, 'Billing and Shipping')
								ELSE eCommerceAction_option
							 END;
```

2. Clean and transform the `analytics` table

We'll make a summary table of how many values there are (columns with all NULL values not included) in the `analytics` table prior to any futher study and data cleaning steps.

```SQL
SELECT 
    j.column_name, COUNT(j.value)
FROM analytics b
	CROSS JOIN LATERAL jsonb_each_text(jsonb_strip_nulls(to_jsonb(b))) AS j(column_name, value)
GROUP BY j.column_name
ORDER BY j.column_name;
``` 

At the first glance, this table contains a lot of rows - up to 4.3 million rows. The return output shows alerts missing values most spread out in the bounces, revenue, units_sold columns. 

|     column_name       |      count        |
|-----------------------|:------------------|
|bounces		        |	474839          |
|channelgrouping        |	4301122         |
|date	                |	4301122         |  
|fullvisitorid          |	4301122         |
|pageviews              |	4301050         |
|revenue		 		|	15355           |
|socialengagementtype   |	4301122         |
|timeonsite		        |	3823657         |
|unit_price		        |	4301122         |
|units_sold             |	95147           |
|visitid                |	4301122         |
|visitnumber	        |	4301122         |
|visitstarttime         |	4301122         |

The `analytics` table has 14 columns in total in which the userid column is fully NULL. We might consider dropping userid column later if it isn't connected to other columns and tables.

```SQL
ALTER TABLE analytics 
DROP COLUMN userid;
```
Let's do some cleaning on the columns before we are able to determine the primary key candidate and remove the duplicates.

- The visitNumber column: Inititally this column was set as VARCHAR, we need to cast it back to INT data type.

```SQL
ALTER TABLE analytics
ALTER COLUMN visitNumber TYPE INT USING visitNumber::INT;
```
- The visitStartTime and date columns: 

Next, we convert visitStartTime into the TIMESTAMP data type from the Unix time. Interestingly after the conversion, there's a date component inside the timestamp which doesn't fully match with the date column, sometimes 1 day later. That may have something to do timezone. We'll have to drop the existing date column to be consistent, and compensate by extracting the date from visitStartTime. 

```SQL
--- Intermediary step to convert the initial setting of VARCHAR into BIGINT
ALTER TABLE analytics
ALTER COLUMN visitStartTime TYPE BIGINT USING visitStartTime::BIGINT;

--- Convert from BIGINT to TIMESTAMP WITH TIME ZONE
ALTER TABLE analytics
ALTER COLUMN visitStartTime SET DATA TYPE TIMESTAMP WITH TIME ZONE
USING TIMESTAMP WITH TIME ZONE 'EPOCH' + visitStartTime * INTERVAL '1 SECOND'; 

--- Check how many rows when date in `visitStartTime` different from `date` column
SELECT COUNT(*)
FROM analytics
WHERE visitStartTime::DATE != date -- 455642

--- Drop the existing `date` column
ALTER TABLE analytics 
DROP COLUMN date;

--- Create a new `date` column with the date from `visitStartTime`
ALTER TABLE analytics  
ADD COLUMN date DATE;

UPDATE analytics 
SET date = visitStartTime::DATE
```
- The fullvisitorId column: Use padding to add leading zeros to the fullvisitorId columns in order to be consistent with the format of the `all_sessions` table.

```SQL
UPDATE analytics
SET fullVisitorId = LPAD(FullVisitorId, 18, '0009999999999999999');
```

- The visitId column: We first observe visitId, visitStartTime are identical, which means visitId values possibly have been copied from visitStartTime as a mistake. We should find the appropriate values to fill in, as the column seems to be important as a unique identifer at a later stage. The common columns between `analytics` and `all_sessions` tables are (fullVisitorId, channelGrouping, visitId). Thus, together they comprise of a composite key candidate for the `analytics` table.

```SQL
--- Fill in the missing visitId column in the `analytics` table with a lookup table CTE extracted from the `fullVisitorId` and `channelGrouping` from the `all_sessions` table
WITH visitId_lookup_CTE AS(
SELECT *
FROM (SELECT al.fullVisitorId, al.channelGrouping, al.visitId
	  FROM all_sessions al
	  GROUP BY al.fullVisitorId, al.visitId, al.channelGrouping) AS a
JOIN
	  (SELECT an.fullVisitorId, an.channelGrouping, an.visitId
	   FROM analytics an
	 GROUP BY an.fullVisitorId, an.visitId, an.channelGrouping) AS b
USING (fullVisitorId, visitId, channelGrouping)
)

INSERT INTO analytics(visitId)
SELECT v.visitId FROM visitId_lookup_CTE v
WHERE NOT EXISTS (SELECT * FROM analytics ana WHERE ana.visitId = v.visitId);
```

- The units_sold column: Fix the odd -89 value by 89 units sold, as data ranges from -89 to 4324, nothing unusal except for this one.

```SQL
UPDATE analytics 
	SET units_sold = CASE 
						WHEN units_sold = -89 THEN 89
						ELSE units_sold
				  	 END;
```

- The timeonsite column (named timeOnSite in the `all_sessions` table): Assuming the data here is in seconds similar to timeOnSite in the `all_sessions` table, we will convert it to INTERVAL.

```SQL
--- Convert `timeonsite` from the initial setting to INTERVAL
ALTER TABLE analytics
ALTER COLUMN timeonsite TYPE INTERVAL USING timeonsite::INTERVAL;
```

- The unit_price, revenue columns: Just like in the `all_sessions` table, those 2 columns are overinflated with 1000000 times more than the usual amount. We'll fix them with a more reasonable amount by dividing by 1000000.

```SQL
--- Fix revenue and unit_price columns' amount from the `analytics` table by dividing by 1000000
UPDATE analytics
SET 
	revenue = ROUND(revenue/1000000, 2),
	unit_price = ROUND(unit_price/1000000, 2);
```

- Delete the full duplicate rows in the `analytics table` after cleaning all the tables:

```SQL
--- Delete repeated rows in `analytics` table by using CONCAT
DELETE FROM analytics an1
WHERE CONCAT( visitNumber, ' ',
			  visitId, ' ', 
			  visitStartTime, ' ',
			  date, ' ',
			  fullvisitorId, ' ',
			  channelGrouping, ' ',
			  socialEngagementType, ' ',
			  units_sold, ' ',
			  pageviews, ' ',
			  timeonsite, ' ',
			  bounces, ' ',
			  revenue, ' ',
			  unit_price)
NOT IN (
		SELECT DISTINCT CONCAT( visitNumber, ' ',
								visitId, ' ', 
								visitStartTime, ' ',
								date, ' ',
								fullvisitorId, ' ',
								channelGrouping, ' ',
								socialEngagementType, ' ',
								units_sold, ' ',
								pageviews, ' ',
								timeonsite, ' ',
								bounces, ' ',
								revenue, ' ',
								unit_price)
		FROM analytics an2
		GROUP BY CONCAT(		visitNumber, ' ',
								visitId, ' ', 
								visitStartTime, ' ',
								date, ' ',
								fullvisitorId, ' ',
								channelGrouping, ' ',
								socialEngagementType, ' ',
								units_sold, ' ',
								pageviews, ' ',
								timeonsite, ' ',
								bounces, ' ',
								revenue, ' ',
								unit_price)
		HAVING COUNT(*) > 1
);
```
We might have not set the primary key for this table yet, but the composite key using (fullVisitorId, channelGrouping, visitId) seems to be a good candidate. They can be used to find connection with the `all_sessions` table.

3. Clean and transform the `products` table

A summary table of how many values there are (columns with all NULL values not included) in the `products` table prior to any futher study and data cleaning steps:

```SQL
SELECT 
    j.column_name, COUNT(j.value)
FROM products p
	CROSS JOIN LATERAL jsonb_each_text(jsonb_strip_nulls(to_jsonb(p))) AS j(column_name, value)
GROUP BY j.column_name
ORDER BY j.column_name;
``` 

The result looks pretty clean with just a few missing values

|     column_name       |      count        |
|-----------------------|:------------------|
|name	        		|	1092         	|
|orderedquantity        |	1092         	|
|restockingleadtime     |	1092          	|
|sentimentmagnitude		|	1091        	|
|sentimentscore	 		|	1091           	|
|sku	 				|	1092        	|
|stocklevel		        |	1092         	|

- The sku (changed name into productsku) column: We have to change the column name into productsku to be uniform with other tables.

```SQL
--- Change column name from `sku` to `productsku`
ALTER TABLE products
RENAME COLUMN sku TO productsku;
```

- The name (changed name into productname) column: We have to change the column name into productname for readability, and remove the leading white space in some of the product names.

```SQL
--- Change column name from `name` to `productname`
ALTER TABLE products
RENAME COLUMN name TO productname;

--- Remove the leading whitespace from `productname`
UPDATE products
   SET productname = REGEXP_REPLACE(productname, '(^\s+)', '');
```

4. No cleaning needed for the `sales_by_sku` table

5. Clean and transform the `sales_report` table

This table is pretty clean already with repeated columns probally drawn from the `products` and `sales_report` tables. We just need to change minor details in the `name` colum as we performed in the previous ones.

```SQL
--- Change column name from `name` to `productname`
ALTER TABLE sales_report
RENAME COLUMN name TO productname;

--- Remove the leading whitespace from `productname`
UPDATE sales_report
   SET productname = REGEXP_REPLACE(productname, '(^\s+)', '');
```
