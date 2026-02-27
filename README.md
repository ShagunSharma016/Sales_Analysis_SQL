# Sales Analysis - Contoso Customer Segmentation, Cohort Revenue & Retention Analysis

## Overview

Analysis of customer behavior, retention, and lifetime value for an e-commerce company to improve customer retention and maximize revenue.

# Business Questions

1. **Customer Segmentation:** Who are our most valuable customers?
2. **Cohort Analysis:** How do different customer groups generate revenue?
3. **Retention Analysis:** Which customers haven't purchased recently?

# Clean Up Data

- Aggregated sales and customer data into revenue metrics
- Calculated first purchase dates for cohort analysis
- Created view combining transactions and customer details

# Analysis

## Project Question - 1: Customer Lifetime Value Segmentation

- Categorized customers based on total lifetime value (LTV)
- Assigned customers to High, Mid, and Low-value segments
- Calculated key metrics like total revenue

## Key Findings

- High-value segment (25% of customers) drives 66% of revenue ($135.4M)
- Mid-value segment (50% of customers) generates 32% of revenue ($66.6M)
- Low-value segment (25% of customers) accounts for 2% of revenue ($4.3M)

## Business Insights

- High-Value (66% revenue): Offer premium membership program to 12,372 VIP customers, as losing one customer significantly impacts revenue
- Mid-Value (32% revenue): Create upgrade paths through personalized promotions, with potential $66.6M → $135.4M revenue opportunity
- Low-Value (2% revenue): Design re-engagement campaigns and price-sensitive promotions to increase purchase frequency
  
## SQL : Customer Lifetime Value Segmentation

```
WITH customer_ltv AS 
(
    SELECT
        customerkey,
        full_name,
        SUM(total_revenue) AS total_net_revenue
    FROM
        cohort_analysis 
    GROUP BY 
        customerkey, 
        full_name
),

customer_segments AS 
(
	SELECT
		PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY total_net_revenue) AS ltv_25,
		PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY total_net_revenue) AS ltv_75
	FROM 
	    customer_ltv
),

segement_values AS 
(
	SELECT 
		c.*, 
		CASE
			WHEN c.total_net_revenue < cs.ltv_25 THEN '3 - LOW '
			WHEN c.total_net_revenue >= cs.ltv_75 THEN '1 - High'
			ELSE '2 - MID'
		END AS customer_segement
	FROM 
		customer_ltv c, 
		customer_segments cs
) 

SELECT 
	customer_segement,
	'$ ' || SUM(total_net_revenue) AS total_ltv,
	COUNT(DISTINCT customerkey) AS total_customers,
	'$ ' || ROUND(SUM(total_net_revenue) / COUNT(DISTINCT customerkey)) AS avg_ltv
FROM
	segement_values 
GROUP BY 
	customer_segement
ORDER BY 
	customer_segement;

```

## Project Question - 2: Cohort-Based Revenue Analysis

- Tracked revenue and customer count per cohorts
- Cohorts were grouped by year of first purchase
- Analyzed customer revenue at a cohort level

## Key Findings:

- Customer revenue is declining, older cohorts (2016-2018) spent ~$2,800+, while 2024 cohort spending dropped to ~$1,970.
- Revenue and customers peaked in 2022-2023, but both are now trending downward in 2024.
- High volatility in revenue and customer count, with sharp drops in 2020 and 2024, signaling retention challenges.

## Business Insights:

- Boost retention & re-engagement by targeting recent cohorts (2022-2024) with personalized offers to prevent churn.
- Stabilize revenue fluctuations and introduce loyalty programs or subscriptions to ensure consistent spending.
- Investigate cohort differences by applying successful strategies from high-spending cohorts (2016-2018) to newer ones.

## SQL : Cohort-Based Revenue Analysis

```
SELECT 
    cohort_year,
    '$ ' || SUM(total_revenue) AS total_net_revenue,
    COUNT(DISTINCT customerkey) AS total_unique_customers,
    '$ ' || ROUND(SUM(total_revenue) / COUNT(DISTINCT customerkey)) AS customer_revenue
FROM 
    cohort_analysis 
WHERE
    orderdate = first_purchase_date
GROUP BY
    cohort_year 
ORDER BY
    cohort_year;

```

## Project Question - 3: Churn Analysis by Cohort

- Identified customers at risk of churning
- Analyzed last purchase patterns
- Calculated customer-specific metrics

## Key Findings:

- Cohort churn stabilizes at ~90% after 2-3 years, indicating a predictable long-term retention pattern.
- Retention rates are consistently low (8-10%) across all cohorts, suggesting retention issues are systemic rather than specific to certain years.
- Newer cohorts (2022-2023) show similar churn trajectories, signaling that without intervention, future cohorts will follow the same pattern.

## Business Insights:

- Strengthen early engagement strategies to target the first 1-2 years with onboarding incentives, loyalty rewards, and personalized offers to improve long-term retention.
- Re-engage high-value churned customers by focusing on targeted win-back campaigns rather than broad retention efforts, as reactivating valuable users may yield higher ROI.
- Predict & preempt churn risk and use customer-specific warning indicators to proactively intervene with at-risk users before they lapse.

## SQL :  Churn Analysis by Cohort

```
WITH customer_last_purchase AS
(
	SELECT 
		customerkey, 
		full_name, 
		orderdate, 
		row_number() OVER(PARTITION BY customerkey ORDER BY orderdate DESC) AS rn,
		first_purchase_date,
		cohort_year 
	FROM 
		cohort_analysis
),

churned_customers AS 
(
	SELECT
		customerkey, 
		full_name, 
		first_purchase_date,
		orderdate AS last_purchase_date, 
		CASE
			WHEN orderdate < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months' THEN 'Churned'
			ELSE 'Active'
		END AS customer_status, 
		cohort_year 
	FROM
		customer_last_purchase 
	WHERE
		rn = 1
		AND first_purchase_date < (SELECT MAX(orderdate) FROM sales) - INTERVAL '6 months'
) 

SELECT 
	cohort_year,
	customer_status,
	COUNT(customerkey) AS num_customers,
	SUM(COUNT(customerkey)) OVER(PARTITION BY cohort_year) AS total_customers,
	ROUND(COUNT(customerkey) / SUM(COUNT(customerkey)) OVER(PARTITION BY cohort_year), 2) AS status_percentage
FROM 
	churned_customers 
GROUP BY
	cohort_year,
	customer_status;

```

## Strategic Recommendations

1. **Customer Value Optimization** (Customer Segmentation)

   - Launch VIP program for 12,372 high-value customers (66% revenue)
   - Create personalized upgrade paths for mid-value segment ($66.6M → $135.4M opportunity)
   - Design price-sensitive promotions for low-value segment to increase purchase frequency

2. **Cohort Performance Strategy** (Customer Revenue by Cohort)

   - Target 2022-2024 cohorts with personalized re-engagement offers
   - Implement loyalty/subscription programs to stabilize revenue fluctuations
   - Apply successful strategies from high-spending 2016-2018 cohorts to newer customers

3. **Retention & Churn Prevention** (Customer Retention)

   - Strengthen first 1-2 year engagement with onboarding incentives and loyalty rewards
   - Focus on targeted win-back campaigns for high-value churned customers
   - Implement proactive intervention system for at-risk customers before they lapse

## Technical Details

- **Database:** PostgreSQL
- **Analysis Tools:** PostgreSQL, Dbeaver
