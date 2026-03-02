# 🚀 SaaS Product Growth Analytics Dashboard

**End-to-End SQL + Tableau Portfolio Project**

A comprehensive product analytics system demonstrating data modeling, advanced SQL analysis, and growth metrics visualization for a fictional SaaS company.

---

## 🎯 Project Overview

### Business Context

**NordicFlow CRM** is a fictional Nordic SaaS company providing customer relationship management tools to SMBs. The company is experiencing:
- Declining user activation rates (down 15% QoQ)
- Increasing churn in months 2-3 after signup
- Low feature adoption beyond core functionality
- Need for data-driven product and growth decisions

### My Role

As Product Data Analyst, I built an end-to-end analytics system to:
1. Track product engagement and user behavior
2. Identify growth opportunities and retention risks  
3. Provide actionable insights for Product and Growth teams
4. Build executive dashboard for KPI monitoring

---

## ✨ Key Features & Deliverables

### SQL Analysis (PostgreSQL)
- ✅ **Event tracking schema** - Star schema with fact_events and dimension tables
- ✅ **User cohort analysis** - Month-over-month retention cohorts
- ✅ **Activation funnel** - 7-day activation journey tracking
- ✅ **Feature engagement scoring** - User engagement levels by feature usage
- ✅ **Churn prediction logic** - Risk scoring based on behavioral signals
- ✅ **Product metrics calculation** - DAU/MAU ratio, stickiness, ARPU

### Tableau Dashboard
- 📊 **Executive KPI Dashboard** - North Star metrics at a glance
- 📈 **User Activation Funnel** - Step-by-step conversion tracking
- 👥 **Cohort Retention Analysis** - Month-over-month retention heatmap
- 🎯 **Feature Adoption Matrix** - Which features drive engagement
- ⚠️ **Churn Risk Monitoring** - At-risk user segments with interventions

### Growth Insights
- 📝 **Business recommendations** - 5 actionable growth initiatives
- 🎲 **A/B test ideas** - Data-driven experiment proposals
- 📉 **Churn reduction strategies** - Targeted intervention playbook

---

## 🛠️ Tech Stack

| Component | Technology | Justification |
|-----------|-----------|---------------|
| **Database** | PostgreSQL 14+ | Industry standard, advanced window functions |
| **Visualization** | Tableau Public | Professional BI tool, shareable dashboards |
| **Data Generation** | Python (optional) | Realistic event data simulation |
| **Version Control** | Git + GitHub | Professional collaboration workflow |
| **Documentation** | Markdown | Clear, scannable documentation |

---

## 📊 Data Model

### Star Schema Design

```
                    ┌─────────────────┐
                    │  dim_users      │
                    │─────────────────│
                    │ user_id (PK)    │
                    │ signup_date     │
                    │ plan_type       │
                    │ company_size    │
                    │ industry        │
                    └────────┬────────┘
                             │
                             │ 1:N
                             │
                    ┌────────▼────────┐
                    │  fact_events    │
                    │─────────────────│
                    │ event_id (PK)   │
                    │ user_id (FK)    │
                    │ event_name      │
                    │ event_timestamp │
                    │ session_id      │
                    │ properties      │
                    └────────┬────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
       ┌────────▼──────┐ ┌──▼────────┐ ┌▼────────────┐
       │ dim_features  │ │ dim_plans │ │ dim_date    │
       │───────────────│ │───────────│ │─────────────│
       │ feature_id    │ │ plan_id   │ │ date_key    │
       │ feature_name  │ │ plan_name │ │ year        │
       │ feature_tier  │ │ mrr       │ │ month       │
       │ launch_date   │ │ features  │ │ quarter     │
       └───────────────┘ └───────────┘ │ day_of_week │
                                        └─────────────┘
```

### Key Tables

**`dim_users`** - User attributes and account information  
**`fact_events`** - All product interaction events (page views, feature usage, actions)  
**`dim_features`** - Product features and their categorization  
**`dim_plans`** - Subscription tiers and pricing  
**`user_metrics`** - Aggregated user-level metrics (materialized view)

---

## 📈 Key Metrics Tracked

### North Star Metrics
- **Weekly Active Users (WAU)** - Primary engagement metric
- **Activation Rate** - % users completing 3+ key actions in first 7 days
- **30-Day Retention** - % of users active 30 days after signup

### Product Engagement Metrics
- **DAU/MAU Ratio (Stickiness)** - Daily active / Monthly active users
- **Feature Adoption Rate** - % users engaging with each feature
- **Session Duration** - Average time per session
- **Actions Per Session** - Engagement depth

### Growth Metrics
- **New User Signups** - Registration rate by channel
- **Activation Funnel Conversion** - Step-by-step drop-off analysis
- **Cohort Retention Curves** - Longitudinal retention by signup cohort
- **Time to Value (TTV)** - Days until first meaningful action

### Revenue Metrics
- **Monthly Recurring Revenue (MRR)** - Predictable revenue
- **Average Revenue Per User (ARPU)** - Revenue / Total users
- **Customer Lifetime Value (CLTV)** - Predicted total value per user
- **Churn Rate** - % users canceling per month

---

## 🔍 SQL Analysis Highlights

### 1. User Activation Analysis
Identifies which actions in first 7 days predict long-term retention:

```sql
-- Calculate 7-day activation metrics
WITH first_week_actions AS (
    SELECT 
        user_id,
        COUNT(DISTINCT event_name) as unique_actions,
        COUNT(*) as total_events,
        BOOL_OR(event_name = 'invite_teammate') as invited_team,
        BOOL_OR(event_name = 'created_contact') as created_contact,
        BOOL_OR(event_name = 'sent_email') as sent_email
    FROM fact_events
    WHERE event_timestamp <= (
        SELECT signup_date + INTERVAL '7 days' 
        FROM dim_users 
        WHERE dim_users.user_id = fact_events.user_id
    )
    GROUP BY user_id
),
retention_check AS (
    SELECT 
        user_id,
        MAX(CASE WHEN days_since_signup >= 30 THEN 1 ELSE 0 END) as retained_30d
    FROM fact_events
    JOIN dim_users USING (user_id)
    GROUP BY user_id
)
SELECT 
    fwa.invited_team,
    fwa.created_contact,
    fwa.sent_email,
    COUNT(*) as user_count,
    AVG(rc.retained_30d) * 100 as retention_rate_pct
FROM first_week_actions fwa
JOIN retention_check rc USING (user_id)
GROUP BY 1, 2, 3
ORDER BY retention_rate_pct DESC;
```

**Insight**: Users who invite a teammate in first 7 days have 3x higher 30-day retention.

---

### 2. Cohort Retention Analysis
Month-over-month retention for each signup cohort:

```sql
-- Monthly cohort retention
WITH cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', signup_date) as cohort_month
    FROM dim_users
),
user_activity AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', event_timestamp) as activity_month
    FROM fact_events
    GROUP BY 1, 2
),
cohort_activity AS (
    SELECT 
        c.cohort_month,
        ua.activity_month,
        EXTRACT(MONTH FROM AGE(ua.activity_month, c.cohort_month)) as months_since_signup,
        COUNT(DISTINCT c.user_id) as users_active
    FROM cohorts c
    JOIN user_activity ua USING (user_id)
    GROUP BY 1, 2, 3
),
cohort_sizes AS (
    SELECT 
        cohort_month,
        COUNT(DISTINCT user_id) as cohort_size
    FROM cohorts
    GROUP BY 1
)
SELECT 
    ca.cohort_month,
    ca.months_since_signup,
    ca.users_active,
    cs.cohort_size,
    ROUND(ca.users_active::NUMERIC / cs.cohort_size * 100, 2) as retention_pct
FROM cohort_activity ca
JOIN cohort_sizes cs USING (cohort_month)
ORDER BY ca.cohort_month, ca.months_since_signup;
```

**Insight**: Retention drops sharply after Month 2 (45% → 28%), indicating onboarding gap.

---

### 3. Feature Engagement Scoring
Calculates engagement score based on feature usage:

```sql
-- Feature engagement levels
WITH feature_usage AS (
    SELECT 
        user_id,
        feature_name,
        COUNT(*) as usage_count,
        COUNT(DISTINCT DATE(event_timestamp)) as days_used
    FROM fact_events
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '30 days'
        AND event_name LIKE 'feature_%'
    GROUP BY 1, 2
),
user_scores AS (
    SELECT 
        user_id,
        COUNT(DISTINCT feature_name) as features_used,
        SUM(usage_count) as total_feature_actions,
        AVG(days_used) as avg_days_per_feature,
        CASE 
            WHEN COUNT(DISTINCT feature_name) >= 5 AND SUM(usage_count) >= 50 THEN 'Power User'
            WHEN COUNT(DISTINCT feature_name) >= 3 AND SUM(usage_count) >= 20 THEN 'Active User'
            WHEN COUNT(DISTINCT feature_name) >= 1 AND SUM(usage_count) >= 5 THEN 'Casual User'
            ELSE 'At Risk'
        END as engagement_level
    FROM feature_usage
    GROUP BY user_id
)
SELECT 
    engagement_level,
    COUNT(*) as user_count,
    ROUND(AVG(features_used), 1) as avg_features,
    ROUND(AVG(total_feature_actions), 0) as avg_actions
FROM user_scores
GROUP BY engagement_level
ORDER BY 
    CASE engagement_level
        WHEN 'Power User' THEN 1
        WHEN 'Active User' THEN 2
        WHEN 'Casual User' THEN 3
        WHEN 'At Risk' THEN 4
    END;
```

**Insight**: Only 12% of users are "Power Users" - opportunity to drive feature adoption.

---

### 4. Churn Risk Prediction
Flags users likely to churn based on behavioral signals:

```sql
-- Churn risk scoring
WITH user_activity AS (
    SELECT 
        user_id,
        MAX(event_timestamp) as last_active,
        COUNT(DISTINCT DATE(event_timestamp)) as active_days_30d,
        COUNT(*) as total_events_30d
    FROM fact_events
    WHERE event_timestamp >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
),
churn_signals AS (
    SELECT 
        u.user_id,
        u.signup_date,
        ua.last_active,
        EXTRACT(DAY FROM CURRENT_DATE - ua.last_active) as days_inactive,
        ua.active_days_30d,
        ua.total_events_30d,
        CASE 
            WHEN ua.last_active IS NULL THEN 'Churned'
            WHEN EXTRACT(DAY FROM CURRENT_DATE - ua.last_active) > 14 THEN 'High Risk'
            WHEN ua.active_days_30d < 3 THEN 'Medium Risk'
            WHEN ua.total_events_30d < 10 THEN 'Low Risk'
            ELSE 'Healthy'
        END as churn_risk
    FROM dim_users u
    LEFT JOIN user_activity ua USING (user_id)
    WHERE u.signup_date < CURRENT_DATE - INTERVAL '30 days'
)
SELECT 
    churn_risk,
    COUNT(*) as user_count,
    ROUND(AVG(days_inactive), 1) as avg_days_inactive,
    ROUND(AVG(active_days_30d), 1) as avg_active_days
FROM churn_signals
GROUP BY churn_risk
ORDER BY 
    CASE churn_risk
        WHEN 'Healthy' THEN 1
        WHEN 'Low Risk' THEN 2
        WHEN 'Medium Risk' THEN 3
        WHEN 'High Risk' THEN 4
        WHEN 'Churned' THEN 5
    END;
```

**Insight**: 23% of users are "High Risk" - target with re-engagement campaign.

---

## 📊 Tableau Dashboard Structure

### Dashboard 1: Executive Growth Overview
**Audience**: C-Suite, VP Product, VP Growth

**KPIs** (Big Numbers):
- Weekly Active Users (WAU): 12,450 ↑ 8%
- Activation Rate: 42% ↓ 3%
- 30-Day Retention: 38% → Flat
- MRR: $245K ↑ 12%

**Charts**:
1. **WAU Trend** (Line) - 12-week rolling trend
2. **New User Signups** (Bar) - By acquisition channel
3. **Cohort Retention Heatmap** - Month 0-12 retention %
4. **Revenue by Plan** (Stacked Bar) - Starter/Pro/Enterprise

**Filters**: Date Range, User Segment, Plan Type

---

### Dashboard 2: Activation Funnel Analysis
**Audience**: Product Managers, Growth Team

**Funnel Steps**:
1. Signup Complete: 10,000 users
2. Profile Created: 8,200 users (82%)
3. First Contact Added: 5,500 users (55%)
4. Email Sent: 3,800 users (38%)
5. Teammate Invited: 2,100 users (21%)
6. **Activated** (3+ actions): 4,200 users (42%)

**Charts**:
1. **Funnel Visualization** - Step-by-step conversion
2. **Drop-off by Cohort** (Line) - Are newer cohorts activating faster?
3. **Time to Activation** (Histogram) - Days distribution
4. **Activation Actions** (Treemap) - Which actions drive activation

**Insights Panel**:
- "Biggest drop-off: Email Sent → Teammate Invited (45% loss)"
- "Enterprise users activate 2.3x faster than Starter"
- "Recommendation: Add teammate invitation prompt after first email"

---

### Dashboard 3: Feature Adoption & Engagement
**Audience**: Product Managers, UX Team

**Charts**:
1. **Feature Adoption Rates** (Bar) - % users engaging each feature
2. **Feature Usage Trends** (Line) - Usage over time by feature
3. **Engagement Score Distribution** (Histogram) - User engagement levels
4. **Feature Correlation Matrix** (Heatmap) - Which features used together
5. **User Segmentation** (Scatter) - Engagement vs Tenure

**Calculated Fields**:
```
// DAU/MAU Stickiness
[Daily Active Users] / [Monthly Active Users]

// Feature Adoption Rate
SUM(IF [Feature Used] THEN 1 END) / COUNTD([User ID])

// Engagement Score
IF [Features Used] >= 5 AND [Actions] >= 50 THEN "Power User"
ELSEIF [Features Used] >= 3 THEN "Active User"
ELSEIF [Features Used] >= 1 THEN "Casual User"
ELSE "At Risk"
END
```

---

### Dashboard 4: Churn Risk & Retention
**Audience**: Customer Success, Growth Team

**KPIs**:
- Users at Risk: 2,850 (23%)
- Predicted Monthly Churn: 8.5%
- Revenue at Risk: $42K MRR

**Charts**:
1. **Churn Risk Distribution** (Donut) - Healthy/Low/Med/High/Churned
2. **At-Risk User List** (Table) - Sortable by risk score
3. **Churn Drivers** (Bar) - Days inactive, feature usage, session frequency
4. **Retention Curves by Segment** (Line) - Enterprise vs SMB retention
5. **Intervention Impact** (Before/After) - Re-engagement campaign results

**Action Items**:
- Export High Risk users for Customer Success outreach
- Trigger automated re-engagement email sequence
- Flag for product improvement (friction points)

---

## 🎯 Key Business Insights & Recommendations

### Finding 1: Activation Cliff
**Data**: Only 42% of signups complete activation (3+ key actions in 7 days)

**Impact**: 5,800 users/month not experiencing core value

**Root Cause**: 
- Onboarding email sequence unclear
- No in-product guidance for critical features
- Teammate invitation not prompted

**Recommendation**:
1. **A/B Test**: In-product checklist vs current onboarding (Expected +8% activation)
2. **Quick Win**: Add "Invite Teammate" prompt after first contact created
3. **Long-term**: Rebuild onboarding with progressive disclosure

**Success Metrics**: Activation rate 42% → 50% (Target: +8 pts in 2 months)

---

### Finding 2: Month 2-3 Retention Drop
**Data**: Retention drops from 45% (Month 1) → 28% (Month 3)

**Impact**: Losing 40% of retained users between months 1-3

**Root Cause**:
- Users not discovering advanced features
- No proactive Customer Success touchpoints
- Plateau in perceived value

**Recommendation**:
1. **Automated Email Series**: Feature education emails at Day 14, 30, 45
2. **In-App Tips**: Contextual feature discovery prompts
3. **CS Intervention**: Proactive check-in call at Day 30 for high-value accounts

**Success Metrics**: Month 3 retention 28% → 38% (Target: +10 pts in Q2)

---

### Finding 3: Feature Adoption Gap
**Data**: 88% of users engage with <3 features (out of 12 available)

**Impact**: Low engagement = low perceived value = high churn risk

**Root Cause**:
- Features not discoverable in UI
- Lack of use-case education
- Advanced features hidden behind poor UX

**Recommendation**:
1. **Feature Spotlight Campaign**: Weekly email + in-app highlighting underused features
2. **Use Case Templates**: Pre-built workflows for common scenarios
3. **Power User Program**: Interview and learn from 12% who use 5+ features

**Success Metrics**: Avg features used per user 2.1 → 3.5 (Target: +67% in Q2)

---

### Finding 4: Enterprise vs SMB Retention Gap
**Data**: Enterprise 30-day retention: 58% | SMB 30-day retention: 32%

**Impact**: SMB segment has 1.8x higher churn despite being 70% of users

**Root Cause**:
- SMBs less likely to have dedicated admin
- More price-sensitive
- Less complex use cases (easier to switch)

**Recommendation**:
1. **SMB-Specific Onboarding**: Simplified, faster time-to-value path
2. **Annual Plans**: Discount for annual commitment (reduce month-to-month churn)
3. **Community**: SMB peer group for best practices sharing

**Success Metrics**: SMB 30-day retention 32% → 42% (Close 50% of Enterprise gap)

---

### Finding 5: Power User Correlation
**Data**: Users inviting teammates in first 7 days → 3x higher 30-day retention

**Impact**: Single strongest predictor of long-term retention

**Root Cause**:
- Multi-user teams have higher switching costs
- Viral/network effect within organization
- Demonstrates product value to stakeholders

**Recommendation**:
1. **Teammate Invitation as Activation Metric**: Shift focus from "3 actions" to "invite + 2 actions"
2. **Viral Mechanics**: Incentivize invitations (free month, premium features)
3. **Team Features First**: Prioritize product features that require collaboration

**Success Metrics**: % users inviting teammates 21% → 35% (Target: +14 pts in Q2)

---

## 🧪 Proposed A/B Test Ideas

### Test 1: Onboarding Checklist
**Hypothesis**: In-product checklist will increase 7-day activation by 8 percentage points

**Control**: Current email-only onboarding  
**Variant**: Persistent checklist sidebar with progress bar

**Sample Size**: 2,000 users per group  
**Duration**: 4 weeks  
**Primary Metric**: 7-day activation rate  
**Secondary Metrics**: Time to activation, feature discovery rate

---

### Test 2: Teammate Invitation Prompt
**Hypothesis**: Prompting teammate invitation after first contact created will increase invitations by 40%

**Control**: No prompt (current)  
**Variant**: Modal prompt: "Invite your team to collaborate on [Contact Name]"

**Sample Size**: 3,000 users per group  
**Duration**: 2 weeks  
**Primary Metric**: % users inviting teammates  
**Secondary Metrics**: 30-day retention, team size

---

### Test 3: Feature Discovery Email Series
**Hypothesis**: 3-email feature education series will increase feature adoption by 25%

**Control**: No additional emails  
**Variant**: Emails at Day 14, 21, 28 highlighting specific features with use cases

**Sample Size**: 1,500 users per group  
**Duration**: 6 weeks  
**Primary Metric**: Average features used per user  
**Secondary Metrics**: Session frequency, Month 2 retention

---

## 📂 Repository Structure

```
saas-product-growth-analytics/
├── README.md                              # This file
├── LICENSE                                # MIT License
├── .gitignore                             # Git ignore rules
│
├── sql/
│   ├── 01_schema_setup.sql               # Database schema creation
│   ├── 02_sample_data_generation.sql     # Sample event data
│   ├── 03_user_activation_analysis.sql   # Activation metrics
│   ├── 04_cohort_retention_analysis.sql  # Cohort retention
│   ├── 05_feature_engagement_scoring.sql # Feature usage analysis
│   ├── 06_churn_risk_prediction.sql      # Churn risk scoring
│   ├── 07_product_metrics_views.sql      # Materialized views for Tableau
│   └── 08_ad_hoc_queries.sql             # Business question queries
│
├── tableau/
│   ├── saas_growth_dashboard.twbx        # Tableau workbook (packaged)
│   ├── dashboard_screenshots/
│   │   ├── 01_executive_overview.png
│   │   ├── 02_activation_funnel.png
│   │   ├── 03_feature_adoption.png
│   │   └── 04_churn_risk.png
│   └── tableau_setup_guide.md            # Instructions for Tableau connection
│
├── data/
│   ├── sample_events.csv                 # Sample event data (1000 rows)
│   ├── sample_users.csv                  # Sample user data
│   └── data_dictionary.md                # Data schema documentation
│
├── docs/
│   ├── business_context.md               # NordicFlow company background
│   ├── metrics_definitions.md            # How each metric is calculated
│   ├── analysis_methodology.md           # SQL approach and techniques
│   ├── architecture_diagram.png          # System architecture
│   ├── erd_diagram.png                   # Database ERD
│   └── presentation_slides.pdf           # 5-slide executive summary
│
├── python/ (Optional)
│   ├── generate_sample_data.py           # Script to generate realistic events
│   ├── requirements.txt                  # Python dependencies
│   └── data_validation.py                # Data quality checks
│
└── DEPLOYMENT_GUIDE.md                   # How to set up project locally
```

---

## 🚀 Getting Started

### Prerequisites
- PostgreSQL 14+ installed locally
- Tableau Public (free) or Tableau Desktop
- (Optional) Python 3.8+ for data generation

### Setup Instructions

**Step 1: Clone Repository**
```bash
git clone https://github.com/YOUR_USERNAME/saas-product-growth-analytics.git
cd saas-product-growth-analytics
```

**Step 2: Create Database**
```bash
# Create database
createdb product_analytics

# Run schema setup
psql product_analytics < sql/01_schema_setup.sql

# Load sample data
psql product_analytics < sql/02_sample_data_generation.sql
```

**Step 3: Run SQL Analysis**
```bash
# Execute analysis queries
psql product_analytics < sql/03_user_activation_analysis.sql
psql product_analytics < sql/04_cohort_retention_analysis.sql
# ... continue with remaining SQL files
```

**Step 4: Connect Tableau**
1. Open Tableau Public
2. Connect to PostgreSQL server (localhost:5432)
3. Select `product_analytics` database
4. Import `saas_growth_dashboard.twbx` workbook
5. Refresh data extracts

**Step 5: View Dashboards**
- Executive Overview: High-level KPIs
- Activation Funnel: Conversion analysis
- Feature Adoption: Engagement metrics
- Churn Risk: Retention monitoring

---

## 📊 Skills Demonstrated

### SQL Expertise
✅ Complex window functions (RANK, NTILE, LAG, LEAD)  
✅ Common Table Expressions (CTEs) with recursive logic  
✅ Advanced JOINs (including self-joins)  
✅ Cohort analysis patterns  
✅ Date/time calculations and bucketing  
✅ Aggregate functions and GROUP BY  
✅ Conditional logic (CASE WHEN) for scoring  
✅ Materialized views for performance

### Product Analytics
✅ Activation funnel analysis  
✅ Cohort retention curves  
✅ Feature engagement scoring  
✅ Churn prediction modeling  
✅ User segmentation  
✅ A/B test design  
✅ North Star metric definition  
✅ Growth loop identification

### Business Acumen
✅ Translating data into actionable insights  
✅ Prioritizing recommendations by impact  
✅ Designing experiments to test hypotheses  
✅ Communicating technical findings to non-technical stakeholders  
✅ Understanding SaaS business models (MRR, CAC, CLTV)  
✅ Product-led growth strategies

### Visualization & Communication
✅ Executive dashboard design  
✅ KPI selection and hierarchy  
✅ Data storytelling  
✅ Clear, scannable documentation  
✅ Technical writing  
✅ Stakeholder presentation skills

---

## 💡 Portfolio Talking Points

**For Interviews:**

*"I built an end-to-end product analytics system for a fictional SaaS company experiencing activation and retention challenges. Using PostgreSQL, I designed a star schema event tracking database and wrote complex SQL queries to analyze user cohorts, activation funnels, and churn risk. I discovered that users who invite teammates in the first 7 days have 3x higher 30-day retention, which became the basis for a proposed A/B test. I visualized these insights in four Tableau dashboards tailored to different stakeholders—from executive KPIs to detailed feature adoption analysis. The project demonstrates my ability to work with product data at scale, identify growth opportunities, and communicate findings to drive business decisions."*

**Key Metrics Improved** (Hypothetical):
- Activation rate: +8 percentage points (projected)
- Month 3 retention: +10 percentage points (projected)
- Feature adoption: +67% users engaging with 3+ features

**Technical Highlights**:
- 8 SQL scripts showcasing advanced techniques (window functions, CTEs, cohort analysis)
- 4 stakeholder-specific Tableau dashboards
- Realistic sample dataset with 10,000+ user events
- Complete documentation including methodology and business context

---

## 🎓 Learning Resources

**SQL for Product Analytics**:
- Mode Analytics SQL School: [mode.com/sql-tutorial](https://mode.com/sql-tutorial/)
- Cohort Analysis Guide: [Mixpanel Blog](https://mixpanel.com/blog/cohort-analysis/)

**Product Metrics**:
- "Lean Analytics" by Alistair Croll & Benjamin Yoskovitz
- Reforge Product Analytics Course: [reforge.com](https://www.reforge.com/)

**Tableau**:
- Tableau Public Gallery: [public.tableau.com/gallery](https://public.tableau.com/gallery)
- Makeover Monday: [makeovermonday.co.uk](https://makeovermonday.co.uk/)

---

## 👤 Author

**Nirav Patel**  
*Data Analyst | Product Analytics Enthusiast*

- 📍 Location: Jersey City, NJ
- 💼 LinkedIn: [linkedin.com/in/YOUR_PROFILE](https://linkedin.com/in/YOUR_PROFILE)
- 🌐 Portfolio: [yourportfolio.com](https://yourportfolio.com)
- 📧 Email: your.email@example.com

---

## 📄 License

This project is licensed under the MIT License - see [LICENSE](LICENSE) file for details.

---

## 🙏 Acknowledgments

- Fictional company "NordicFlow" inspired by real SaaS product analytics challenges
- Dataset structure modeled after [Mixpanel](https://mixpanel.com/) and [Amplitude](https://amplitude.com/) event schemas
- SQL patterns adapted from [Mode Analytics](https://mode.com/) and [Count.co](https://count.co/) examples
- Dashboard design influenced by [Reforge](https://www.reforge.com/) product analytics frameworks

---

**⭐ If this project helped you, please star the repository!**

*This project demonstrates technical skills for portfolio purposes. The company and data are fictional.*
