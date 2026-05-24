# Multifamily Real Estate Data Analytics Portfolio

## 📊 Project Overview
This project simulates a comprehensive data analytics lifecycle for a **400-unit multifamily housing asset** over a 12-month period. Acting as a Lead Property Data Analyst, the objective is to monitor, analyze, and optimize core property management Key Performance Indicators (KPIs) including **occupancy tracking, lease retention, financial delinquency, and resident satisfaction sentiment.**

The project demonstrates an end-to-end analytics workflow:
1. **Data Generation:** Python (`pandas`, `numpy`) to engineer a realistic, relational dataset.
2. **Exploratory Data Analysis:** Cloud-based Data Warehousing using **Google BigQuery** to extract operational business insights.
3. **Data Modeling & Visualization:** Power BI (Star Schema & DAX) to deploy executive-ready dashboards.

---

## 🗄️ Relational Database Architecture & Schema
Rather than using a flat, unrealistic spreadsheet, this project relies on a relational schema consisting of four interconnected tables inside Google BigQuery (`multifamily-data-project.Property_Leases`):

* **`property_units` (Dimension):** Static data detailing the physical asset mix (Square footage, Bedrooms, Market Rent).
* **`property_leases` (Fact/Dimension):** Tracks individual lease lifecycles, resident retention patterns, and renewal logic.
* **`property_transactions` (Fact):** Monthly ledger recording rent charges, payments, late fees, and financial statuses.
* **`property_surveys` (Fact):** Captures multi-touchpoint resident satisfaction tracking (1-5 scale scores).

```
       [ property_units ]
               │ (1)
               ▼ (*)
       [ property_leases ] ──(*)──► [ property_transactions ]
               │ (1)
               ▼ (*)
       [ property_surveys ]
```

---

## 🛠️ Tech Challenge: Adapting to BigQuery's Schema Inference
During data ingestion, Google BigQuery automatically and intelligently optimized column data types instead of reading everything as raw text strings. This required writing robust, platform-specific SQL logic:
* **Boolean Parsing:** Column flags like `Renewal_Offered` and `Renewal_Accepted` were natively typed as `BOOL` (`true`/`false`), replacing standard `'Y'`/`'N'` string lookups with cleaner `IS TRUE` syntax.
* **Temporal Precision:** Transaction dates were recognized as native `DATE` objects, meaning string functions like `SUBSTRING` were replaced with native `FORMAT_DATE()` operations to preserve query optimization.

---

## 🚀 Exploratory Data Analysis (SQL)

### Phase 1: Broad Asset Validation

#### Query 1: Property Asset & Rent Mix Verification
**Business Context:** Validate the baseline configuration of the physical real estate asset to confirm floorplan diversity and ensure average baseline market rents sit flush with regional underwriting assumptions.

```sql
SELECT 
  COUNT(DISTINCT Unit_ID) AS total_units,
  COUNT(DISTINCT Bedrooms) AS unique_floorplans,
  ROUND(MIN(Market_Rent), 2) AS lowest_rent,
  ROUND(MAX(Market_Rent), 2) AS highest_rent,
  ROUND(AVG(Market_Rent), 2) AS average_market_rent
FROM `multifamily-data-project.Property_Leases.property_units`;
```

#### Query 2: Warehouse Integrity Row Counts
**Business Context:** A mandatory sanity check to ensure full data pipeline ingestion across all relational tables before running complex aggregation models.

```sql
SELECT 'property_units' AS table_name, COUNT(*) AS row_count FROM `multifamily-data-project.Property_Leases.property_units`
UNION ALL
SELECT 'property_leases', COUNT(*) FROM `multifamily-data-project.Property_Leases.property_leases`
UNION ALL
SELECT 'property_transactions', COUNT(*) FROM `multifamily-data-project.Property_Leases.property_transactions`
UNION ALL
SELECT 'property_surveys', COUNT(*) FROM `multifamily-data-project.Property_Leases.property_surveys`;
```

---

### Phase 2: Operational Analytics (Leasing & Retention)

#### Query 3: Portfolio Retention & Renewal Conversions
**Business Context:** Property performance targets assume an **85% resident retention rate**. This script evaluates overall portfolio stability and measures the direct conversion efficacy of renewal notices distributed by the on-site team.

```sql
WITH RenewalCounts AS (
  SELECT 
    COUNT(*) AS total_expiring_leases,
    SUM(CASE WHEN Renewal_Offered IS TRUE THEN 1 ELSE 0 END) AS total_offered,
    SUM(CASE WHEN Renewal_Accepted IS TRUE THEN 1 ELSE 0 END) AS total_accepted,
    SUM(CASE WHEN UPPER(Lease_Status) = 'VACATED' THEN 1 ELSE 0 END) AS total_vacated
  FROM `multifamily-data-project.Property_Leases.property_leases`
)
SELECT 
  total_expiring_leases,
  total_offered,
  total_accepted,
  total_vacated,
  ROUND((total_accepted / NULLIF(total_offered, 0)) * 100, 2) AS renewal_offer_acceptance_rate_pct,
  ROUND((total_accepted / NULLIF(total_expiring_leases, 0)) * 100, 2) AS overall_retention_rate_pct
FROM RenewalCounts;
```
* **Operational Insight:** Evaluates if the property manager is successfully minimizing turnover costs (turns, painting, marketing) by retaining the target user base.

---

### Phase 3: Financial Analytics & Delinquency Tracking

#### Query 4: Monthly Financial Performance and Delinquency Analysis
**Business Context:** Real estate operations require tight accounts receivable controls. Underwriting parameters mandate maintaining general delinquency between **2% and 8%**. This dynamic analysis flags structural payment shortfalls month-over-month.

```sql
WITH MonthlyFinancials AS (
  SELECT 
    FORMAT_DATE('%Y-%m', Transaction_Date) AS year_month,
    SUM(CASE WHEN Category = 'Rent Charge' THEN Amount ELSE 0 END) AS gross_potential_rent,
    SUM(CASE WHEN Category = 'Rent Charge' AND Status = 'Unpaid' THEN Amount ELSE 0 END) AS delinquent_rent,
    SUM(CASE WHEN Category = 'Payment' THEN ABS(Amount) ELSE 0 END) AS total_payments_collected,
    SUM(CASE WHEN Category = 'Late Fee' THEN Amount ELSE 0 END) AS late_fees_assessed,
    SUM(CASE WHEN Category = 'Late Fee' AND Status = 'Unpaid' THEN Amount ELSE 0 END) AS unpaid_late_fees
  FROM `multifamily-data-project.Property_Leases.property_transactions`
  GROUP BY 1
)
SELECT 
  year_month,
  gross_potential_rent,
  total_payments_collected,
  delinquent_rent,
  ROUND((delinquent_rent / NULLIF(gross_potential_rent, 0)) * 100, 2) AS delinquency_rate_pct,
  late_fees_assessed,
  ROUND((total_payments_collected / NULLIF(gross_potential_rent, 0)) * 100, 2) AS collections_realization_rate_pct
FROM MonthlyFinancials
ORDER BY year_month ASC;
```
* **Financial Insight:** Tracks cash flow velocity and flags seasonal operational friction points where collections fall behind Gross Potential Rent (GPR).

---

### Phase 4: Customer Experience & Sentiment Modeling

#### Query 5: Average Customer Satisfaction (CSAT) by Touchpoint
**Business Context:** Isolates specific operational friction points in the property lifecycle (Move-In vs. Maintenance requests vs. Renewal notices) to detect operational efficiency issues.

```sql
SELECT 
  Touchpoint,
  COUNT(*) AS total_surveys_received,
  ROUND(AVG(Score), 2) AS average_satisfaction_score,
  SUM(CASE WHEN Score = 5 THEN 1 ELSE 0 END) AS highly_satisfied_count,
  SUM(CASE WHEN Score <= 2 THEN 1 ELSE 0 END) AS dissatisfied_count
FROM `multifamily-data-project.Property_Leases.property_surveys`
GROUP BY Touchpoint
ORDER BY average_satisfaction_score DESC;
```

#### Query 6: Correlating Survey Sentiment with Actual Resident Attrition
**Business Context:** Proves the structural link between resident experience and financial turnover. This query joins operational feedback directly to eventual physical lease outcomes to see if negative renewal interactions cause lease terminations.

```sql
SELECT 
  s.Score AS renewal_notice_survey_score,
  COUNT(DISTINCT l.Lease_ID) AS total_residents,
  SUM(CASE WHEN l.Renewal_Accepted IS TRUE THEN 1 ELSE 0 END) AS renewed_count,
  SUM(CASE WHEN UPPER(l.Lease_Status) = 'VACATED' THEN 1 ELSE 0 END) AS vacated_count,
  ROUND((SUM(CASE WHEN l.Renewal_Accepted IS TRUE THEN 1 ELSE 0 END) / COUNT(DISTINCT l.Lease_ID)) * 100, 2) AS conversion_rate_pct
FROM `multifamily-data-project.Property_Leases.property_surveys` s
JOIN `multifamily-data-project.Property_Leases.property_leases` l 
  ON s.Resident_ID = l.Resident_ID
WHERE s.Touchpoint = 'Renewal Notice Received'
GROUP BY 1
ORDER BY renewal_notice_survey_score DESC;
```
* **Strategic Value:** Proves to stakeholders that a low survey score isn't just a sentiment metric—it's a leading financial indicator of impending vacancy loss and turn expenses.

---
## 📊 Business Intelligence & Dashboard Deployment (Power BI)

The final phase of the project transitions raw warehouse insights into an interactive 3-page executive application using **Power BI Desktop**, structured as an optimized **Star Schema** with bidirectional filter propagation.

### 📐 Core Multifamily DAX Engines
To evaluate operational trends cleanly across varying calendar granularities without daily date dilution, the following custom DAX models were developed:

* **Context-Aware Portfolio Occupancy:** Evaluates active baseline leasing volume relative to structural turnover by tracking total active resident metrics against physical asset constraints.
* **Bidirectional Attrition Modeling:** Implements cross-filtering capabilities to allow sentiment analysis to filter upstream lease agreements.

---

### 🖥️ Dashboard Architecture & Visual Storytelling

#### Page 1: Executive Property Overview
* **Purpose:** High-level operational health checks for regional stakeholders.
* **Core Visuals:** Unified KPI cards capturing stabilized performance targets, paired with an interactive unit-mix selector. 
* **Data Narrative:** Visually surfaces how recurring collection shortfalls directly align with seasonal delinquency spikes.

<img width="1914" height="1074" alt="image" src="https://github.com/user-attachments/assets/a35c6a9d-4715-49b9-a5ae-0bf368d91dd1" />


#### Page 2: Financial Performance & Collections Deep-Dive
* **Purpose:** An operational workspace for on-site Property Managers to mitigate cash leaks.
* **Core Visuals:** An accounts receivable matrix embedded with conditional formatting, isolating localized unit-level balances.
* **Data Narrative:** Highlights variance between Gross Potential Rent and true cash velocity, exposing exactly which units require penalty fee enforcement.

<img width="1900" height="993" alt="image" src="https://github.com/user-attachments/assets/5527f881-a585-4127-869b-bf8dbbce7bef" />


#### Page 3: Resident Experience & Attrition Correlation
* **Purpose:** Strategic analysis proving the ROI of on-site property management operations.
* **Core Visuals:** Cross-filtered bar charts mapping satisfaction scores across critical tenant milestones.
* **Data Narrative:** **The Smoking Gun.** Statistically proves that high survey sentiment directly triggers physical lease renewals, translating customer satisfaction into concrete resident retention.

<img width="1885" height="1043" alt="image" src="https://github.com/user-attachments/assets/11f6a60f-2183-421e-bb41-01b34c317931" />
