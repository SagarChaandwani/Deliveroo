/*
=========================================================================
PROJECT: Deliveroo Trust & Safety Operations
TECHNOLOGY: Power BI / DAX
DESCRIPTION: Key measures driving the Operational Console visuals.
=========================================================================
*/

/* -----------------------------------------------------------------------
   SECTION 1: HEADS-UP DISPLAY (KPIs)
   Logic for the top-row status indicators.
   ----------------------------------------------------------------------- */

-- Measure: Total Active Riders
-- Simple count of the dimension table.
Total Riders = COUNTROWS('Dim_Riders')

-- Measure: Substitute Account %
-- Calculates the share of fleet using rented accounts (High Risk).
Substitute % = 
DIVIDE(
    CALCULATE(COUNTROWS('Dim_Riders'), 'Dim_Riders'[Rider_Account_Type] = "Substitute"),
    [Total Riders],
    0
)

-- Measure: Phantom Order Rate (Fraud)
-- The % of orders marked as 'Phantom' (Stolen/Undelivered).
Phantom Order Rate = 
DIVIDE(
    CALCULATE(COUNTROWS('Fact_Orders'), 'Fact_Orders'[Is_Phantom_Order] = 1),
    COUNTROWS('Fact_Orders'),
    0
)

-- Measure: Total Capital Loss
-- Financial impact of refunds.
Total Refund Loss = SUM('Fact_Orders'[Refund_Amount])


/* -----------------------------------------------------------------------
   SECTION 2: COMPLIANCE & RISK LOGIC
   Complex filtering for gauges and charts.
   ----------------------------------------------------------------------- */

-- Measure: Suspended Accounts
-- Logic: Riders with Risk Score > 90 OR a failed Identity Check.
Suspended Accounts = 
CALCULATE(
    COUNTROWS('Dim_Riders'),
    FILTER(
        'Dim_Riders',
        'Dim_Riders'[Rider_Risk_Score] > 90 || 
        CALCULATE(COUNTROWS('Fact_Verifications'), 'Fact_Verifications'[Is_Failed] = 1) > 0
    )
)

-- Measure: Verification Pass Rate
-- Used for the Gauge Visual (Target: 95%).
Verification Pass Rate = 
DIVIDE(
    CALCULATE(COUNTROWS('Fact_Verifications'), 'Fact_Verifications'[Is_Failed] = 0),
    COUNTROWS('Fact_Verifications'),
    0
)

/* -----------------------------------------------------------------------
   SECTION 3: VISUAL HELPERS
   Measures used purely for UI effects (Rings and Colours).
   ----------------------------------------------------------------------- */

-- Measure: Phantom Remainder
-- Used to create the grey portion of the 'Donut Ring' KPI.
Phantom Remainder = 1 - [Phantom Order Rate]

-- Measure: Substitute Remainder
-- Used to create the grey portion of the 'Donut Ring' KPI.
Substitute Remainder = 1 - [Substitute %]