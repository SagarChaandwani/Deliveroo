/*
========================================================================================
PROJECT:    Deliveroo Trust & Safety Operations
AUTHOR:     [Sagar Chaandwani]
OBJECTIVE:  ETL Script to transform raw order logs into a Rider Risk Profile.
            
KEY LOGIC:
1. CTEs: Used to modularise Order, Rider, and Verification data.
2. WINDOW FUNCTIONS: 
   - LAG: To detect "Spiraling" fraud behaviour (MoM change in Phantom Orders).
   - DENSE_RANK: To rank cities by Refund Loss magnitude.
3. JOINS: Linking Financial data (Orders) with Identity data (Verifications).
========================================================================================
*/

-- CTE 1: Aggregating Order Data (Financial Impact)
WITH Order_Metrics AS (
    SELECT 
        o.Rider_ID,
        -- Calculate Total Loss
        SUM(o.Refund_Amount) AS Total_Refund_Loss,
        COUNT(o.Order_ID) AS Total_Orders,
        
        -- Calculate Phantom Order Rate (Using 1.0 to force float division)
        CAST(SUM(CASE WHEN o.Is_Phantom_Order = 1 THEN 1 ELSE 0 END) AS FLOAT) 
        / NULLIF(COUNT(o.Order_ID), 0) AS Phantom_Rate,

        -- Identify Latest Activity
        MAX(o.Order_Date) AS Last_Active_Date
    FROM Fact_Orders o
    GROUP BY o.Rider_ID
),

-- CTE 2: Verification Compliance (Identity Check)
Verification_Status AS (
    SELECT 
        v.Rider_ID,
        COUNT(v.Verification_ID) AS Total_Checks,
        SUM(v.Is_Failed) AS Failed_Checks,
        -- Determine current status based on most recent check
        TOP(1) v.Status_Icon AS Latest_Status_Icon
    FROM Fact_Verifications v
    GROUP BY v.Rider_ID, v.Status_Icon
),

-- CTE 3: Advanced Risk Profiling (Window Functions)
Risk_Engine AS (
    SELECT 
        r.Rider_ID,
        r.City,
        r.Rider_Account_Type,
        om.Total_Refund_Loss,
        om.Phantom_Rate,
        
        -- DENSE_RANK: Rank Riders by Loss within their City
        DENSE_RANK() OVER (
            PARTITION BY r.City 
            ORDER BY om.Total_Refund_Loss DESC
        ) AS City_Risk_Rank,

        -- LAG: Compare current month Phantom Rate vs Previous Month to detect trends
        -- (Assuming aggregated monthly data logic here for demonstration)
        LAG(om.Phantom_Rate, 1, 0) OVER (
            PARTITION BY r.Rider_ID 
            ORDER BY om.Last_Active_Date
        ) AS Previous_Phantom_Rate

    FROM Dim_Riders r
    LEFT JOIN Order_Metrics om ON r.Rider_ID = om.Rider_ID
)

-- FINAL OUTPUT: The "One Truth" Table for Power BI
SELECT 
    re.Rider_ID,
    re.City,
    re.Rider_Account_Type,
    re.Total_Refund_Loss,
    
    -- Logic: Create "Risk Bucket" based on calculations
    CASE 
        WHEN re.Phantom_Rate > 0.5 OR vs.Failed_Checks > 2 THEN 'High Risk - Ban Imminent'
        WHEN re.Phantom_Rate > 0.1 THEN 'Medium Risk - Monitor'
        ELSE 'Low Risk - Safe'
    END AS Calculated_Risk_Bucket,

    -- Logic: Flag Spiraling Behaviour (If Rate increased from previous month)
    CASE 
        WHEN re.Phantom_Rate > re.Previous_Phantom_Rate THEN 1 
        ELSE 0 
    END AS Is_Risk_Escalating,

    re.City_Risk_Rank

FROM Risk_Engine re
LEFT JOIN Verification_Status vs ON re.Rider_ID = vs.Rider_ID
WHERE re.Total_Refund_Loss > 0
ORDER BY re.Total_Refund_Loss DESC;