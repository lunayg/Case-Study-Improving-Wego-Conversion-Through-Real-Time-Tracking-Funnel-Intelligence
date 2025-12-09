## Case Study: Root-Cause Analysis of CTR Decline: Detecting FX Drift & Pricing Inconsistencies in Wego

_Focus on uncovering the drivers of CTR Drop, a real-time pricing diagnostics for Wego_

---

## 🗒️Overview

This case study simulates a real-world project for **Wego**, a leading travel platform, focused on solving key business intelligence challenges in travel data during peak seasons:  
- **Price volatility** and **data lag** 
- Potential loss of user trust suggested by **Conversion drop-offs** in the search-to-booking funnel  

The project demonstrates **how diagnosing root causes**  can **improve conversion performance over time**.

---

## 📍 1. Context & Problem Statement

Wego aggregates millions of flight and hotel options daily from multiple providers across regions. Internal analysis revealed a major challenge:

**Conversion Drop QoQ After Search**  
   Despite high search activity, the conversion rate from *search → click → booking redirect* was declining during  Ramadan travel week.  
   Product managers suspected **pricing inconsistencies** as main culprits.
---
## ✅ Previous Insights

1. Correlation & Lag Analysis

- FX Drift consistently increased before the CTR drop, establishing a directionally consistent correlation between pricing anomalies and user engagement decline.

2. System Health Check

- No cache delays were detected, and **API benchmarks were functioning normally**, indicating that system performance was not the root cause of the anomaly.

3. Route Isolation

- Abnormal FX Drift was isolated to the **Dubai → Cairo route**, while all other routes remained stable, confirming the issue was route-specific.

4. Provider Sensitivity Check

- No **abnormal CTR drops were observed at the individual provider level**, further supporting that the CTR decline was not provider-specific.

5. Overall Conclusion

- The analysis confirmed that **FX Drift preceded the CTR drop**.

- Route-level isolation highlighted the Dubai → Cairo route as the critical path for intervention.

- Provider-level checks validated that the anomaly was route-specific, making it a clear priority to validate the causal relationship between FX Drift and CTR decline for this route to inform targeted corrective actions.


---

## ✅ 2. Objective

To diagnose whether the conversion drop after the *search stage* was primarily caused by:

📌 Inaccurate or delayed pricing data 

This stage focuses on validating hypotheses and _isolating key variables (poor search relevance)_ impacting conversion rate.

---

## ✅ 2. Hypotheses

| ID | Hypothesis | Layer | Metric | Expected Outcome |
|----|-------------|------|--------|------------------|
| H1 | System latency spikes **latency (P95 > 5s)** cause **FX drift and stale pricing**. | System | p95 Latency, FX Drift %, Stale Price % | Positive correlation |
| H2 | FX drift and stale prices increase **price volatility**.| Data | FX Drift %, Stale Price %, Price Volatility Index | Positive trend |
| H3 | Higher price volatility reduces **Price accuracy**, which lowers **user trust, and therefore CTR** and **Search-to-Book Conversion**. | Behavioral | Price Volatility %, Price Accuracy %, CTR, Search-to-Book Conversion %, Abandonment % | Negative trend |
| H4 | Search relevance remains stable → **pricing is the key driver**.| Control | Relevance Score, CRI, CRR, Bounce % | Stable |

Flowchart
```
  A -->|H1: p95 latency 5s leads to| B
  B -->|H2: FX drift + stale →| C
  C -->|H3: Volatility →| D
  S -->|H4: If stable →| D
  D -->|H3: lower user trust, CTR, Search-to-Book Conversion 
```
---

##  3. ? Data Sources & Architecture

| Source | Type | Key Fields | Frequency |
|--------|------|-------------|------------|
| Flight API Logs | External | provider_id, route, price, timestamp | Real-time |
| Booking Clickstream | Event Data | user_id, session_id, event_type, timestamp | Continuous |
| Search Funnel Data | Internal DB | searches, clicks, redirects, bookings | Hourly |
| Supplier Metadata | Static | provider_id, region, latency_score | Monthly |

**Cloud:** BigQuery (GCP)  
**BI Tools:** Looker Studio, Tableau Public  
**ETL Simulation:** Airflow (Simulated)

---

## 🔍 4. Analytical Approach

### A. Data Exploration & Cleaning
- Used **SQL (CTEs + Window Functions)** to unify logs and filter invalid price entries (>3 hours stale).
- Applied data deduplication to remove duplicate redirects from API retries.
- Built aggregated tables(flight_api_log, fx_conversion_table etc) in **BigQuery** for each flight route combining search, click, and booking data.
- Query the latest flight API log from each provider.
- Example:
```sql
WITH flight_api_log AS (
  SELECT
    ROW_NUMBER() OVER (PARTITION BY provider_id, route ORDER BY timestamp DESC) AS number
    provider_id,
    route,
    fare,
    currency,
    timestamp,
    response_status,
    latency_ms,
  FROM raw_logs
  WHERE fare IS NOT NULL
)
SELECT * FROM flight_api_log WHERE rn = 1;
```

---

### **B. Main Funnel Overview**

**Purpose:** Quantify *where* conversion drop occurs and who it affects most.

- Constructed user funnel: **Search → Click → Detail → Booking**
- Segmented by **route in MENA region**.
- Measured conversion per stage and trend over time

**🧩 Finding:**  
CTR declined **35%** across UAE searches, particularly from route Dubai to Cairo in 1st and 2nd week of the ramadan month, indicating reduced user trust. 

---

### **C. Price Accuracy & Volatility Monitor**

**Purpose:** Validate if price inconsistencies directly correlate with conversion drop among SEA users.

| Metric | Formula | Description |
|--------|----------|-------------|
| **FX Drift %** | `(price_api - price_fx_normalized) / price_fx_normalized * 100` | Currency normalization deviation |
| **Volatility Index** | `STDDEV(price)/AVG(price)*100` | Price fluctuation severity |
| **Stale Price Rate** | `SUM(flag_stale)/COUNT(*)*100` | % of outdated flight results |
| **Price–Click Correlation** | `CORR(price_dev_pct, CTR)` | Sensitivity of CTR to price deviation |

**Visualization Ideas:**
- Heatmap of **FX Drift % by Provider & Region**
- Scatter plot of **Volatility vs Conversion Rate**
- Tooltip overlay showing CTR trend and price deviation

**🧩 Finding:**  
Providers with **>10% FX drift** and **>20% stale rates** saw **CTR drop 18%**, confirming pricing latency as a conversion risk among SEA users.

---

### **D. Search Relevance & Conversion Quality***
**Purpose:** Rule out irrelevant search results as alternative cause.

| Metric | Formula |  
|--------|----------|
| **Relevance Score (RS)** | `clicked results / total results shown * normalized CTR` | 
| **Click Relevance Index (CRI)** | `Σ(wᵢ × CTRᵢ) / Σ(wᵢ)` | 
| **Conversion Relevance Rate(CRR)** | `conversions from top 5 results/total conversions` | 
| **Relevance Weighted Conversion Rate (RWCR)** | `∑(conversion Rate x relevance score)` | 
| **Bounce Rate** | `searches with no click/total searches` | 


**Visualization Ideas:**
- Conversion tree (**Search → Click → Detail → Booking**)  
- Region & route filters  
- Tooltip with price deviation and provider reliability

**🧩 Finding:**  
Relevance scores remained stable across most routes, confirming **pricing accuracy**, not content mismatch, as the main driver of the drop.

---
---

### **i. Pricing Inconsistencies Root Cause Deep Dive**

**Purpose:** Connect **business KPIs** with **system metrics** for root-cause attribution.

**Layers of Analysis:**

| Layer                 | Insight                                                                                 |
| --------------------- | --------------------------------------------------------------------------------------- |
| **FX System**         | FX Drift spiked 8–10% in Thailand due to outdated currency cache.                       |
| **Cache / Latency**   | Data refresh delayed by 6–8 minutes; stale price rate rose 22%.                         |
| **Provider Behavior** | Provider A’s reliability score dropped from 0.94 → 0.82, aligning with 17% CTR decline. |

**Visualization:**

* Multi-axis correlation plot (FX Drift %, Cache/Latency, CTR)
* Drill-down: Provider → Route → Currency
* Event annotations for cache and latency spikes

🧩 **Finding:**
System-level **FX cache staleness** and **latency bottlenecks** amplified price discrepancies, eroding user trust and click-through intent.

---

## **4. Forecasts**

????

---
## **5. Visualization Forecasting & Stakeholder Dashboards**

Developed two interactive dashboards in **Tableau**, integrated with **time-series forecasting (ARIMA–Prophet sketch)** for proactive anomaly detection.

### **A. Price Accuracy Monitor**
- Real-time % of outdated prices by provider & route  
- Alerts for latency spikes > 5 minutes, FX Drift > 2.5%
- Dual-axis line: FX Drift (%) vs Volatility Index (%)
- Control bands showing Prophet forecast intervals (expected drift range)

?⭐ Stakeholder Value?:
Ops can visually spot “forecast breaks” (actual > upper bound), triggering investigation into latency or API mismatch events before conversion loss occurs.

---

### **B. Funnel Conversion & Forecast Insights**

**Purpose**: Forecast short-term conversion recovery and identify volatility-driven drops.

## **i. Visuals**
- Sankey Funnel: Search → Click → Detail → Booking
- Overlay Prophet forecast for CTR & Conversion Rate (next 7 days)
- Highlight anomalies where actual performance diverges from model forecast

Forecasting Sketch (Concept):

- **Model Inputs**: CTR, Conversion Rate, Price Accuracy Rate (daily granularity)
- **Method**: Prophet model to account for seasonality and external regressors (e.g., provider latency, FX drift).
- **Output**: Tableau line chart with shaded 95% forecast confidence region.
- xxx
- ?Forecasting Sketch (Concept)?:
- **Model Inputs**: Historical FX Drift %, Volatility Index, and Stale Price Rate (hourly granularity)
- **Method**: ARIMA to capture short-term seasonality; Prophet for long-term trend + event sensitivity (e.g., cache refresh, provider outage).
- **Output**: Forecasted baseline and confidence interval visualized in Tableau as shaded prediction bands.


⭐ Stakeholder Value:
Product managers can anticipate when conversion may normalize after resolving latency issues, aiding prioritization of technical fixes.

---

## ✅ **5.Insights Summary**

| Insight | Impact | Recommendation |
|----------|---------|----------------|
| Price updates lagging for 3 key providers ( **>10% FX drift** and **>20% stale rates**)
 | 18% lower CTR | Prioritize API caching refresh or exclude outdated listings |
| High seasonality spikes not reflected in forecasts | Inaccurate demand predictions | Integrate ARIMA/Prophet models for booking volume |

---

## ✅ **6. Outcome Simulation**

After implementing **dynamic dashboard refresh** & **Forecasting Insights** :

| Metric | Before(Reactive Monitoring) | After(Predictive Intelligence with ARIMA-Prophet Sketch | Expected Impact |
|--------|--------|--------|-------------|
| **Price Accuracy Monitor** | Manual detection of FX Drift spikes after 3–6 hr delay | ARIMA–Prophet hybrid predicts drift >2.5% deviation ahead of time | Early anomaly detection → Alert Ops 4–6 hr sooner |
| **Stale Price Rate Tracking** | Static threshold alert when stale >15% | Forecasted stale rate trends with confidence intervals | Preventive cache refresh scheduling |
| **Funnel Conversion Insights** | Reactive CTR | Prophet model projects 7-day conversion recovery trend post  | Forecasts conversion rebound post-fix 
| **Decision Efficiency** | Post-issue reporting | Proactive prioritization of technical fixes  | +25% faster incident response time

---

##  **7. 📌 Tools & Skills Demonstrated**

| Category | Tools / Skills |
|-----------|----------------|
| **Data Management** | SQL (CTEs, aggregation, DDL), BigQuery |
| **Visualization** | Tableau, Looker Studio |
| **Exploration** | Funnel & Cohort Analysis |
| **Collaboration** | Requirements documentation with Product Managers |
| **Pipeline Simulation** | Airflow scheduling concepts |
| **Predictive Outlook** | ARIMA / Prophet for seasonality forecasting |

---

##  **8. Strategic Takeaway**

This project illustrates how **data freshness** and **funnel transparency** directly influence **user trust and revenue** in travel platforms.  
By maintaining **accurate real-time datasets** and delivering **data-driven dashboards with forecasting insights**, it enabled **Ops and Product teams** to detect pricing anomalies **before** user trust or conversion was impacted, ensuring faster response, higher reliability and sustained funnel stability.

---

##  **Keywords**
`Travel Analytics` · `Funnel Optimization` · `Data Quality` · `Real-Time Dashboards` · `SQL` · `Tableau` · `BigQuery` · `User Behavior Analysis` · `Conversion Insights` · `Stakeholder Reporting`

---

## 🧩 **Author**
**Samantha Yoong**  
📍 Kuala Lumpur, Malaysia  
🔗 [LinkedIn](https://www.linkedin.com/in/samantha-yoong-8551b4226/) | [Tableau Portfolio](https://public.tableau.com/app/profile/samantha.yoong/vizzes) | [GitHub Projects](https://samanthayoong.github.io/my-portfolio/)



