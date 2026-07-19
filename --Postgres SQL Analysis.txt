--Postgres SQL Analysis
--Data assessment and understanding the data structure
SELECT * FROM line_downtime;
SELECT * FROM downtime_factors;
SELECT * FROM line_productivity_batches;
SELECT * FROM products;

--Number of rows
SELECT COUNT(*) AS numrows_downtime_factors 
FROM downtime_factors;
SELECT COUNT(*) AS numrows_line_downtime 
FROM line_downtime;
SELECT COUNT(*) AS numrows_products
FROM products;
SELECT COUNT(*) AS numrows_line_productivity_batches
FROM line_productivity_batches;

-- Check for nulls
-- Factors null counts
SELECT COUNT(*) AS factor1_null_count 
FROM line_downtime WHERE Factor_1 IS null;
SELECT COUNT(*) AS factor2_null_count 
FROM line_downtime WHERE Factor_2 IS null;

-- All factors null counts
SELECT COUNT(*) AS all_factors_null_count
FROM line_downtime
WHERE COALESCE(Factor_1, Factor_2, Factor_3, Factor_4, Factor_5, Factor_6, 
Factor_7, Factor_8, Factor_9, Factor_10, Factor_11, Factor_12, Factor_13) IS null

-- Start date repetition in "Date" and "Start_time" columns
WITH Start_Dates AS (
SELECT Date, CAST(Start_Time AS Date) AS Start_Date 
FROM line_productivity_batches)
SELECT * FROM Start_Dates
WHERE Date != Start_Date;

-- Data cleaning & transformation
-- Handle null line_downtime
CREATE VIEW downtimes AS
SELECT
    ld.Batch_ID,
    REPLACE(v.factor, 'Factor_', '')::INTEGER AS Factor_ID,
    v.minutes
FROM line_downtime ld
CROSS JOIN LATERAL (
    VALUES
        ('Factor_1', Factor_1),
        ('Factor_2', Factor_2),
        ('Factor_3', Factor_3),
        ('Factor_4', Factor_4),
        ('Factor_5', Factor_5),
        ('Factor_6', Factor_6),
        ('Factor_7', Factor_7),
        ('Factor_8', Factor_8),
        ('Factor_9', Factor_9),
        ('Factor_10', Factor_10),
        ('Factor_11', Factor_11),
        ('Factor_12', Factor_12),
        ('Factor_13', Factor_13)
) AS v(factor, minutes)
WHERE v.minutes IS NOT NULL;

SELECT * FROM downtimes
ORDER BY Batch_ID

-- New batch production table(Date as start date, extract time from start and end date)
CREATE VIEW batch_pd AS
SELECT
    Date AS Start_Date,
    Product_ID,
    Batch_ID,
    Operator,
    CAST(End_Time AS DATE) AS End_Date,
    CAST(Start_Time AS TIME) AS Start_Time,
    CAST(End_Time AS TIME) AS End_Time,
    Planned_Min_Batch_Hours,
    ROUND(EXTRACT(EPOCH FROM (End_Time - Start_Time)) / 3600)
    AS Actual_Duration,
    ROUND(
    (EXTRACT(EPOCH FROM (End_Time - Start_Time)) / 3600)
    - Planned_Min_Batch_Hours
    )
	AS Extra_Time_Hr
FROM line_productivity_batches;

SELECT * FROM batch_pd;

-- Validating downtime minutes against planned production time and reported start/end time
SELECT batch_pd.Batch_ID, SUM(Minutes) AS accounted_delay_minutes,Extra_Time_Hr
FROM batch_pd
JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
GROUP BY batch_pd.Batch_ID, Extra_Time_Hr
ORDER BY batch_pd.Batch_ID;

-- Analysis
-- Downtime key factors
SELECT Factor_Name, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_mins
FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
GROUP BY Factor_Name
ORDER BY Frequency DESC;

-- Operator VS Non-Operator errors
SELECT 
CASE Operator_Error
WHEN 1 THEN 'Yes' ELSE 'No'
END AS Operator_Error,
COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_mins
FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
GROUP BY Operator_Error;

-- Downtimes operator errors
SELECT Factor_Name, Description, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_mins
FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
WHERE Operator_Error = 1
GROUP BY Factor_Name, Description
ORDER BY SUM(Minutes) DESC;

-- Downtimes non-operator errors
SELECT Factor_Name, Description, COUNT(Batch_ID) AS Frequency, SUM(Minutes) AS Delay_mins
FROM downtimes
JOIN downtime_factors ON downtimes.Factor_ID = downtime_factors.Factor_ID
WHERE Operator_Error = 0
GROUP BY Factor_Name, Description
ORDER BY SUM(Minutes) DESC;

-- Products and downtimes frequency and delay(minutes)
SELECT batch_pd.Product_ID, Product_Name, COUNT(downtimes.Batch_ID) AS Frequency, SUM(Minutes) AS Delay_mins
FROM batch_pd
JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
JOIN products ON batch_pd.Product_ID = products.Product_ID
GROUP BY batch_pd.Product_ID, Product_Name
ORDER BY batch_pd.Product_ID;

--How many factors are involved in each product downtime?
SELECT batch_pd.Product_ID, Product_Name, COUNT(DISTINCT(Factor_ID)) AS Distinct_factor_count
FROM batch_pd
JOIN products ON batch_pd.Product_ID = products.Product_ID
JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
GROUP BY batch_pd.Product_ID, Product_Name;

-- Top five factors for product 1 downtime
SELECT
    Factor_Name,
    SUM(Minutes) AS Prd001_Delay_Mins
FROM downtimes
JOIN downtime_factors
    ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd
    ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD001'
GROUP BY Factor_Name
ORDER BY Prd001_Delay_Mins DESC
LIMIT 5;

-- Top five factors for product 2 downtime
SELECT
    Factor_Name,
    SUM(Minutes) AS Prd002_Delay_Mins
FROM downtimes
JOIN downtime_factors
    ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd
    ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD002'
GROUP BY Factor_Name
ORDER BY Prd002_Delay_Mins DESC
LIMIT 5;

-- Top five factors for product 3 downtime
SELECT
    Factor_Name,
    SUM(Minutes) AS Prd003_Delay_Mins
FROM downtimes
JOIN downtime_factors
    ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd
    ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD003'
GROUP BY Factor_Name
ORDER BY Prd003_Delay_Mins DESC
LIMIT 5;

-- Top five factors for product 4 downtime
SELECT
    Factor_Name,
    SUM(Minutes) AS Prd004_Delay_Mins
FROM downtimes
JOIN downtime_factors
    ON downtimes.Factor_ID = downtime_factors.Factor_ID
JOIN batch_pd
    ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Product_ID = 'PRD004'
GROUP BY Factor_Name
ORDER BY Prd004_Delay_Mins DESC
LIMIT 5;

-- Production Lead Operators
SELECT Operator, COUNT(Batch_ID) AS Number_of_batches, COUNT(DISTINCT(Product_ID)) AS Number_of_products
FROM batch_pd
GROUP BY Operator;

-- Production lead operator and downtime duration, percentage delayed batches
SELECT Operator, 
COUNT(DISTINCT batch_pd.Batch_ID) AS Total_batches,
COUNT(DISTINCT(downtimes.Batch_ID)) AS Number_of_delayed_batches,
COUNT(downtimes.Batch_ID) AS Number_of_downtimes,
SUM(Minutes) AS Delayed_mins,
ROUND(
COUNT(DISTINCT downtimes.Batch_ID)::numeric * 100 / 
COUNT(DISTINCT batch_pd.Batch_ID),2
) AS Percentage_Delayed_Batches
FROM batch_pd
LEFT JOIN downtimes ON batch_pd.Batch_ID = downtimes.Batch_ID
GROUP BY Operator
ORDER BY Percentage_Delayed_Batches DESC;

-- Factor causing downtime for Top 3 lead operators with the most delay duration
-- 1. Paul
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'Paul'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

-- 2. James
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'James'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

-- 3. Emily
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'Emily'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

-- Factors causing downtime for Top 3 operators with the most percentage batches
--1. Linda
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'Linda'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

--2. Sophia
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'Sophia'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;

--3. Rita
SELECT Factor_Name, SUM(Minutes) AS Delay_mins
FROM downtime_factors
JOIN downtimes ON downtime_factors.Factor_ID = downtimes.Factor_ID
JOIN batch_pd ON batch_pd.Batch_ID = downtimes.Batch_ID
WHERE Operator = 'Rita'
GROUP BY Factor_Name
ORDER BY SUM(Minutes) DESC;



