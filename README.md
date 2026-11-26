# SaaS-B2B-Customer-Cohort-Retention-Revenue-Analytics


### 1. Data Quality Issues Found and their handling
### 2. SQL Analysis
QUERY 1: Churn Risk Score
Identify high-risk accounts using: support escalations, downgrades, low satisfaction. Create a risk_level column (High/Medium/Low)
```
Support_summary measures customer support friction by aggragting support ticket data at the account levelWITH support_summary AS (   -- support_summary cte Aggregates Operational Friction indicators
    SELECT [Account ID],
        COUNT(CASE WHEN [Escalation Flag] = TRUE THEN 1 END) AS escalation_count,
        COUNT(CASE WHEN [Satisfaction Score] <= 2 THEN 1 END) AS low_sat_count,
        ROUND(AVG([Satisfaction Score]), 2) AS avg_satisfaction_score
    FROM support GROUP BY [Account ID]
), 
churn_summary AS ( -- churn_summary (aggregates Contract Event History)
    SELECT [Account ID],
        MAX(CASE WHEN [Is Reactivation] = TRUE THEN 1 ELSE 0 END) AS is_reactivation_flag,
        MAX(CASE WHEN [Preceding Downgrade Flag] = TRUE THEN 1 ELSE 0 END) AS preceding_downgrade_flag,
        MAX(CASE WHEN [Preceding Upgrade Flag] = TRUE THEN 1 ELSE 0 END) AS preceding_upgrade_flag
    FROM churn GROUP BY [Account ID]
),
usage_summary AS ( --aggregates Product Error Rates across Tables
    SELECT  s.[Account ID],
        COUNT(CASE WHEN u.[Error Count] > 5 AND u.[Is Beta Feature] = FALSE THEN 1 END) AS stable_high_error_count,
        COUNT(CASE WHEN u.[Error Count] > 5 AND u.[Is Beta Feature] = TRUE THEN 1 END) AS beta_high_error_count
    FROM feature_usage u
    JOIN subscriptions s ON u.[Subscription Id] = s.[Subscription Id]
    GROUP BY s.[Account ID]
),
account_risk_scoring AS (
    SELECT 
        a.[Account ID],a.[Account Name], a.[Plan Tier],a.[Churn Flag],
        (COALESCE(c.is_reactivation_flag, 0) * 5) +
        (COALESCE(c.preceding_downgrade_flag, 0) * 4) +
        (COALESCE(c.preceding_upgrade_flag, 0) * 3) +
        (COALESCE(s.escalation_count, 0) * 2) +
        (COALESCE(s.low_sat_count, 0) * 2) +
        (COALESCE(u.stable_high_error_count, 0) * 2) +
        (COALESCE(u.beta_high_error_count, 0) * 1) AS churn_risk_score
    FROM accounts a
    LEFT JOIN churn_summary c ON a.[Account ID] = c.[Account ID]
    LEFT JOIN support_summary s ON a.[Account ID] = s.[Account ID]
    LEFT JOIN usage_summary u ON a.[Account ID] = u.[Account ID]
)
SELECT 
    [Account ID], [Account Name], [Plan Tier], [Churn Flag], churn_risk_score,
    CASE 
        WHEN churn_risk_score >= 5 THEN 'High Risk'
        WHEN churn_risk_score BETWEEN 2 AND 4 THEN 'Medium Risk'
        ELSE 'Low Risk'
    END AS risk_level
FROM account_risk_scoring
ORDER BY churn_risk_score DESC, [Account ID] ASC;
```

Query 2: Feature Adoption
Which features have highest adoption? Which features do churned customers use? Do beta features correlate with churn?
``` WITH account_status AS (
    SELECT [Account ID],CASE WHEN [Churn Flag] = TRUE THEN 1 ELSE 0 END AS is_churned
    FROM accounts
),

feature_usage_with_account AS (
    SELECT s.[Account ID],f.[Feature Name],f.[Is Beta Feature],f.[Usage Count]
    FROM feature_usage f
    JOIN subscriptions s ON f.[Subscription Id] = s.[Subscription Id]
),

account_feature_usage AS (
    SELECT fu.[Account ID],fu.[Feature Name],fu.[Is Beta Feature],
        SUM(fu.[Usage Count]) AS total_account_usage
    FROM feature_usage_with_account fu
    GROUP BY fu.[Account ID], fu.[Feature Name], fu.[Is Beta Feature]
),

-- Feature-level adoption and churn metrics
feature_metrics AS (
    SELECT f.[Feature Name],f.[Is Beta Feature],
        -- Unique accounts using this feature
        COUNT(DISTINCT f.[Account ID]) AS total_adopting_accounts,
        
        -- Unique churned accounts using this feature
        COUNT(DISTINCT CASE WHEN a.is_churned = 1 THEN f.[Account ID] END) AS churned_accounts_using_feature,
        
        -- Feature Churn Rate (%)
        ROUND( COUNT(DISTINCT CASE WHEN a.is_churned = 1 THEN f.[Account ID] END) * 100.0 / NULLIF(COUNT(DISTINCT f.[Account ID]), 0), 2
        ) AS feature_churn_rate_pct
    FROM account_feature_usage f
    JOIN account_status a ON f.[Account ID] = a.[Account ID]
    GROUP BY f.[Feature Name], f.[Is Beta Feature]
)

-- Final SELECT Query
SELECT [Feature Name], [Is Beta Feature], total_adopting_accounts,
    churned_accounts_using_feature, feature_churn_rate_pct,
    -- 1. Answers "Which features have highest adoption?"
    DENSE_RANK() OVER (ORDER BY total_adopting_accounts DESC) AS overall_adoption_rank,
    
    -- 2. Answers "Which features do churned customers use most?"
    DENSE_RANK() OVER (ORDER BY churned_accounts_using_feature DESC) AS churned_usage_rank,
    
    -- 3. Answers "Do beta features correlate with churn?" (Average churn rate for Beta vs Non-Beta)
    ROUND( AVG(feature_churn_rate_pct) OVER (PARTITION BY [Is Beta Feature]),2) AS avg_beta_group_churn_rate_pct

FROM feature_metrics
ORDER BY overall_adoption_rank ASC;
```

QUERY 3: Support Quality
Which priority tickets resolve slowest? Do escalations correlate with low satisfaction? Build a support_quality_score
```

Query to know Which priority tickets resolve slowest?                                                                                                                                                                                       SELECT  [Priority],    COUNT([Ticket ID]) AS total_tickets, 
ROUND(AVG([Resolution Time Hours]), 2) AS avg_resolution_hours,
FROM support GROUP BY [Priority] ORDER BY avg_resolution_hours DESC; 

---- Do escalations correlate with low  satisfaction?   
SELECT 
    [Escalation Flag],
    COUNT(*) AS total_tickets,
    ROUND(AVG(CAST([Satisfaction Score] AS FLOAT)), 2) AS avg_csat,
    -- Low satisfaction rate (CSAT <= 3) calculating % of tickets in that group with a satisfaction score of 3 or below
    ROUND(SUM(CASE WHEN [Satisfaction Score] <= 3 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS low_csat_pct,
    -- High satisfaction rate (CSAT >= 4) calculating % of tickets in that group with a score of 4 or above
    ROUND(SUM(CASE WHEN [Satisfaction Score] >= 4 THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS high_csat_pct
FROM support
WHERE [Satisfaction Score] IS NOT NULL
GROUP BY [Escalation Flag]; 


-- Query to Build a support quality score 
  WITH binned_support_tickets AS (
    SELECT [Ticket ID], [Account ID], [Priority],
        -- Binning customer satisfaction scores and assigning points (1 to 5 Scale)
        CASE 
            WHEN [Satisfaction Score] >4 THEN 5
            WHEN [Satisfaction Score] between 3.1 and 4 THEN 4
            WHEN [Satisfaction Score] <=3 THEN 3 
            ELSE null
        END AS csat_pts,
        -- Binning Resolution Time and assigning points(1 to 5 Scale)
        CASE 
            WHEN [Resolution Time Hours] <= 11 THEN 5
            WHEN [Resolution Time Hours] between 12 and 23 THEN 4
            WHEN [Resolution Time Hours] between 24 and 47 THEN 3
            WHEN [Resolution Time Hours] between 48 and  72 THEN 2
            ELSE 1
        END AS resolution_pts,
        -- Binning First Response Time and assigning points  (1 to 5 Scale)
        CASE 
            WHEN [First Response Time Minutes] between 0 and 15 THEN 5
            WHEN [First Response Time Minutes] between 16 and 30 THEN 4
            WHEN [First Response Time Minutes] between 31 and 60 THEN 3
            WHEN [First Response Time Minutes] between 61 and 120 THEN 2
            ELSE 1
        END AS response_pts,
        -- Escalation Binary Flag
        CASE WHEN [Escalation Flag] = 0 THEN 5 ELSE 1 END AS escalation_pts
    FROM support
)

SELECT  [Ticket ID],  [Account ID], [Priority],
    -- Weighted Total Ticket Quality Score
    -- Weights: CSAT (40%), Resolution (30%), Response (20%), Escalation (10%)
    -- Maximum Possible Raw Points = (5*4) + (5*3) + (5*2) + (5*1) = 50
    -- (csat_pts * 4) + (resolution_pts * 3) + (response_pts * 2) + (escalation_pts * 1) AS raw_quality_score,
    -- Normalized Score Scaled to 0 - 100%
CASE WHEN csat_pts is NOT NULL THEN 
                ROUND(((csat_pts * 4) + (resolution_pts * 3) + (response_pts * 2) + (escalation_pts * 1)) / 50.0 * 100,  2 )
    -- Weights: Resolution (40%), Response (40%), Escalation (10%)
    ELSE   ROUND(( (resolution_pts * 4) + (response_pts * 4) + (escalation_pts * 1)) / 45.0 * 100,  2 ) 
END AS  support_quality_score_pct
FROM binned_support_tickets
```

### Tableau Dashboards

<img src="Dashboard Images/Dashboard_1__.png" width="800" height="600">
<img src="Dashboard Images/dashboard_2__.png" width="800" height="600">
<img src="Dashboard Images/Dashboard_3.png" width="800" height="600">
<img src="Dashboard Images/dashboard_3__.png" width="800" height="600">
