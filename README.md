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


QUERY 3: Support Quality
Which priority tickets resolve slowest? Do escalations correlate with low satisfaction? Build a support_quality_score

### Tableau Dashboards

<img src="Dashboard Images/Dashboard_1__.png" width="800" height="600">
<img src="Dashboard Images/dashboard_2__.png" width="800" height="600">
<img src="Dashboard Images/Dashboard_3.png" width="800" height="600">
<img src="Dashboard Images/dashboard_3__.png" width="800" height="600">
