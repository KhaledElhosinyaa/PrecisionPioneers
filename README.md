# Pharmaceutical Waste & Profit Leakage Analysis

> *Where Data Meets the Pharmacy Supply Chain*

## Overview

This is a **comprehensive data analytics graduation project** developed as part of the **Digital Egypt Pioneers Initiative — Data Analytics Track**. It analyzes daily, branch-level pharmaceutical sales transactions across a multi-branch pharmacy network to quantify how much revenue is generated, how much profit is retained, and how much value is lost to expired and damaged stock.

The full pipeline follows the **Medallion Architecture (Bronze → Silver → Gold)**, and culminates in **"PharmAnalytics"** — a seven-page interactive **Power BI** dashboard, prototyped first in **Figma**, alongside a supervised **Random Forest** classification model that flags individual transactions as Low, Medium, or High waste-and-profit risk.

**Key Focus Areas:**

- Branch-level profitability & revenue analysis
- Expired & damaged stock waste quantification
- Branch performance benchmarking
- Inventory dead-stock & ABC (Pareto) classification
- **ML-powered Transaction Risk Classifier (Random Forest — 98.5% Accuracy)**

---

## Project Team

| Name |
|---|
| Yusuf Mohamed Mohamed |
| Nada Ahmed Kamal |
| Khaled Mohamed Elhosiny |
| Youssef Waleed |

**Supervisor:** Eng. Mohamed Elemam

---

## Dataset

### Pharmaceutical Retail Transactions Dataset

| Attribute | Detail |
|---|---|
| **Dataset Type** | Branch-level daily pharmaceutical sales transactions (synthetic) |
| **Granularity** | One row per Branch × Drug × Day |
| **Total Branches** | 13 active branches (15 branch keys defined in dimension table) |
| **Total Products** | 50 distinct drugs / medical products |
| **Total Rows (Fact Table)** | 821,250 transaction records |
| **Reporting Period** | January 2021 onward — dashboard default view: Jan–Dec 2024 |
| **File Format** | CSV, transformed to structured Bronze / Silver / Gold tables |
| **Currency** | Egyptian Pound (EGP) |

The raw (Bronze) dataset contains **8 columns**:

- **Date** — Calendar date of the transaction
- **Branch** — Branch identifier (Branch_01 – Branch_15)
- **Drug_Name** — Name and dosage of the drug or medical product
- **Sales_Quantity** — Units sold on that day at that branch
- **Unit_Price** — Retail selling price per unit (EGP)
- **Unit_Cost** — Procurement cost per unit (EGP)
- **Expired_Quantity** — Units removed from stock due to expiry
- **Damaged_Quantity** — Units removed from stock due to physical damage

---

## Tools & Technologies

| Component | Technology | Purpose |
|---|---|---|
| **Data Engineering** | Python (Pandas) | Ingesting, cleaning, feature engineering, Bronze/Silver/Gold tables |
| **Machine Learning** | Scikit-learn | Random Forest risk-classification model |
| **Development Environment** | Jupyter / Google Colab | End-to-end ETL and modeling workflow, ipywidgets prediction simulator |
| **Data Architecture** | Medallion Architecture (Bronze → Silver → Gold) | Structured, scalable data pipeline |
| **Data Modeling** | Star Schema (1 Fact + 3 Dimensions) | Optimized relational design |
| **Visualization** | Power BI / PharmAnalytics BI Platform | Interactive 7-page dashboard |
| **Dashboard Design** | Figma | UI/UX wireframing & layout planning |

---

## Architecture & Steps

### Step 1: Data Pipeline (Medallion Architecture)

```
Bronze Layer → Silver Layer → Gold Layer
  (Raw)         (Cleaned)      (Refined)
```

#### Bronze Layer

- Raw CSV import — no transformation applied
- Permanent unmodified archive of the 8 raw transaction fields

#### Silver Layer

- One standardization pass applied (zero rows dropped):

| Field | Issue Addressed | Correction Applied |
|---|---|---|
| `Drug_Name` | Inconsistent capitalization across source rows | Standardized to title case |
| `Date` | Stored as text in the raw export | Parsed to date type, decomposed into Year / Month / Month_Name |
| `Branch` | Free-text label | Mapped to a surrogate `BranchKey` via `dim_branch` |

- Eight composite KPI columns engineered (see Feature Engineering below)
- All 821,250 Bronze rows carry through — no rows dropped

#### Gold Layer

- Star schema built from the enriched Silver table
- Split into **1 fact table** (`fact_sales`) and **3 dimension tables**, joined via surrogate keys

---

### Step 2: Data Modeling (Star Schema)

The Gold layer implements one central **Fact Table** joined to three **Dimension Tables** via surrogate keys (`BranchKey`, `DateKey`, `ProductKey`):

```
                    ┌─────────────────┐
                    │   fact_sales    │
         ┌──────────│  (Revenue, Cost,│──────────┐
         │          │   Net_Profit,   │          │
         │          │  Waste_Rate...) │          │
         ▼          └────────┬────────┘          ▼
   dim_branch                │                dim_product
  (Branch_01–15)             ▼               (50 drug names)
                         dim_date
                    (Date, Year, Month,
                       Quarter)
```

| Table | Type | Key Columns | Description |
|---|---|---|---|
| `fact_sales` | Fact | `DateKey`, `ProductKey`, `BranchKey` | Sales_Quantity, Unit_Price, Unit_Cost, Revenue, Total_Cost, Net_Profit, Profit_Margin, Expired_Quantity, Damaged_Quantity, Waste_Quantity, Waste_Rate |
| `dim_branch` | Dimension | `BranchKey` | Branch identifier lookup (Branch_01 – Branch_15) |
| `dim_date` | Dimension | `DateKey` | Date, Year, Month, MonthName, Quarter |
| `dim_product` | Dimension | `ProductKey` | Drug_Name lookup for all 50 products |

---

### Step 3: Feature Engineering

Eight composite KPI columns were engineered in the Silver layer from the four raw numeric fields:

| KPI | Formula | Purpose |
|---|---|---|
| **Revenue** | `Sales_Quantity × Unit_Price` | Gross sales value of the transaction |
| **Total_Cost** | `Sales_Quantity × Unit_Cost` | Procurement cost of the units sold |
| **Expired_Loss** | `Expired_Quantity × Unit_Cost` | Value written off due to expiry |
| **Damaged_Loss** | `Damaged_Quantity × Unit_Cost` | Value written off due to physical damage |
| **Net_Profit** | `Revenue − Total_Cost − Expired_Loss − Damaged_Loss` | Profit after accounting for both waste channels |
| **Waste_Quantity** | `Expired_Quantity + Damaged_Quantity` | Total units lost to waste |
| **Waste_Rate** | `Waste_Quantity ÷ Sales_Quantity` | Waste as a share of units sold |
| **Profit_Margin** | `Net_Profit ÷ Revenue` | Profitability of the transaction, net of waste |

---

### Step 4: Machine Learning

A rule-based `Risk_Level` label was constructed directly from the Silver-layer financial fields, then a **Random Forest Classifier** was trained to predict it from operational fields alone (without the fields used to construct the label).

#### Risk Scoring Rule

| Condition | Points Added |
|---|---|
| `Waste_Rate ≥ 0.10` | +2 |
| `Expired_Quantity ≥ 3` | +2 |
| `Damaged_Quantity ≥ 2` | +1 |
| `Profit_Margin < 0.20` | +1 |

| Total Score | Risk_Level |
|---|---|
| 4+ | High Risk |
| 2–3 | Medium Risk |
| 0–1 | Low Risk |

#### Class Distribution (821,250 transactions)

| Risk_Level | Transaction Count | Share |
|---|---|---|
| Low Risk | 755,445 | 91.97% |
| Medium Risk | 33,663 | 4.10% |
| High Risk | 32,142 | 3.91% |

#### Model Configuration

| Parameter | Value |
|---|---|
| Algorithm | Random Forest Classifier (scikit-learn) |
| Trees (n_estimators) | 200 |
| Random State | 42 |
| Features | ProductKey, BranchKey, Sales_Quantity, Revenue, Total_Cost |
| Target | Risk_Level (label-encoded) |
| Train / Test Split | 80% / 20% — 657,000 / 164,250 rows |
| **Accuracy** | **98.50%** |

#### Per-Class Performance

| Risk_Level | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| High Risk | 0.85 | 0.85 | 0.85 | 6,388 |
| Low Risk | 1.00 | 1.00 | 1.00 | 151,196 |
| Medium Risk | 0.84 | 0.84 | 0.84 | 6,666 |
| **Weighted Avg** | **0.99** | **0.98** | **0.98** | 164,250 |

**Why Random Forest?**
- Risk_Level is built from threshold-based conditions — naturally step-like decision boundaries that tree ensembles capture well
- Robust to class imbalance (Low Risk outnumbers High Risk >23:1) without aggressive rebalancing
- No feature scaling required — features sit on very different numeric scales
- Strong out-of-the-box performance: 98.5% accuracy, 0.90 macro F1 with minimal tuning

**Confusion Matrix Insight:** Misclassifications concentrate almost entirely between High Risk and Medium Risk tiers (both boundary conditions in the scoring rule), while confusion with the dominant Low Risk class is minimal in both directions.

#### Interactive Risk Prediction Tool

An `ipywidgets`-based simulator was built directly in the notebook — enter Product, Branch, Sales Quantity, Revenue, and Total Cost for a hypothetical transaction, and the trained model returns a predicted `Risk_Level` on demand.

---

## PharmAnalytics Dashboard (7 Pages)

### Page 1: Landing
Project identity, team, headline scale metrics — Total Branches, Total Products, Total Revenue, Profit Margin

### Page 2: Profitability
Revenue, net profit, margin by drug — Highest Revenue Product, Total Revenue, Total Net Profit, Avg. Profit Margin

### Page 3: Waste & Loss
Expired and damaged stock cost — Total Waste, Expired Loss, Damaged Loss, Waste % of Revenue

### Page 4: Branch Performance
Branch-level profit and waste comparison — Total Branches, Top Branch, Avg. Net Profit, Network Net Profit

### Page 5: Inventory
Dead stock and ABC / Pareto classification — Total Inventory Cost, Dead Stock Rate, A/B/C-Class Products

### Page 6: Executive Summary
CEO-level roll-up — Total Revenue, Total Net Profit, Total Waste, Revenue Growth

### Page 7: Style Guide
Design system reference — Color Palette, Icon Library

---

## Key KPIs

### Profitability
| KPI | Value |
|---|---|
| Total Revenue | 2.69bn EGP |
| Total Net Profit | 982.62M EGP |
| Average Profit Margin | 36.49% |
| Highest Revenue Product | Cough Syrup |
| Highest Margin Product | Vitamin D3 |
| Loss-Making Drugs | 3 drugs |

### Waste & Loss
| KPI | Value |
|---|---|
| Total Waste (Cost) | 26.96M EGP |
| Expired Loss | 16.78M EGP |
| Damaged Loss | 10.18M EGP |
| Waste % of Revenue | 1.00% |
| Waste % of Cost | 1.58% |
| Profit Lost to Waste | 2.74% |
| Highest Waste Month | January |

### Branch Performance
| KPI | Value |
|---|---|
| Total Branches | 13 |
| Top Branch | Branch_03 (91M EGP) |
| Lowest Branch | Branch_09 (55M EGP) |
| Average Net Profit / Branch | 69.5M EGP |
| Network Total Net Profit | 903.5M EGP |

### Inventory
| KPI | Value |
|---|---|
| Total Inventory Cost | 1.68bn EGP |
| Dead Stock Rate | 0.81% |
| A-Class Products | 8 products |
| B-Class Products | 24 products |
| C-Class Products | 18 products |

---

## SMART Research Questions

| # | Question | KPI / Measure |
|---|---|---|
| 1 | Does total pharmaceutical waste exceed 1% of total revenue, and which loss type dominates? | Total Waste, Waste % of Revenue, Expired vs Damaged Loss |
| 2 | How much does Net Profit vary between the top and lowest-performing branches? | Net Profit by Branch, Top Branch, Lowest Branch |
| 3 | Do insulin products show waste rates above 15%, and what share of total waste cost do they represent? | Waste Rate by Drug, Top Contributors to Waste Cost |
| 4 | What proportion of the 50-drug portfolio qualifies as Class A inventory, and does dead stock concentrate among low-turnover drugs? | ABC Distribution, Dead Stock Rate |
| 5 | How closely does actual monthly profit track potential (waste-free) profit? | Potential Profit vs Actual Profit, Profit Lost to Waste |

---

## Project File Structure

```
Pharma_Waste_Analysis/
├── Medallion_architecture/
│   ├── Bronze/
│   │   └── bronze_layer.ipynb          # Raw CSV ingestion — no transformation
│   ├── Silver/
│   │   └── silver_layer.ipynb          # Cleaning + Feature Engineering (8 KPI columns)
│   └── Gold/
│       └── gold_layer.ipynb            # Star schema build (fact_sales + 3 dims)
├── ML_Models/
│   └── Risk_Level_ML/
│       └── Risk_Level_RandomForest.ipynb   # Production Random Forest model
├── Dashboard/
│   └── PharmAnalytics.pbix             # 7-page Power BI dashboard
└── README.md
```

---

## How to Run

### Prerequisites

- Python 3.x environment (Jupyter / Google Colab)
- Power BI Desktop
- Python libraries: `pandas`, `scikit-learn`, `ipywidgets`

### Execution Steps

#### Step 1: Run Bronze Layer
Ingests the raw CSV export exactly as received — no transformation, permanent unmodified archive.

#### Step 2: Run Silver Layer
Standardizes `Drug_Name` to title case, parses `Date`, and engineers the 8 derived KPI columns (Revenue, Total_Cost, Expired_Loss, Damaged_Loss, Net_Profit, Waste_Quantity, Waste_Rate, Profit_Margin). No rows dropped.

#### Step 3: Run Gold Layer
Splits the enriched Silver table into `fact_sales` + `dim_branch`, `dim_date`, `dim_product`, joined via surrogate keys.

#### Step 4: Run ML Model
Constructs the rule-based `Risk_Level` label, trains Random Forest Classifier (200 trees), evaluates on held-out test set, and launches the interactive `ipywidgets` prediction simulator.

#### Step 5: Open Power BI Dashboard
```
# Connect Power BI to the Gold-layer star schema
# Open the .pbix file and refresh the data source
```

---

## Key Findings & Insights

### Profitability
- Cough Syrup and ORS Sachets are the top two revenue and profit generators network-wide
- 3 drugs post negative aggregate Net Profit despite the network operating at a healthy 36.49% average margin

### Waste & Loss
- Four cold-chain / injectable products (Ceftriaxone 1G, Insulin Glargine, Insulin Aspart, Insulin Regular) drive ~96% of total waste cost
- Expired loss (16.78M EGP) outweighs damaged loss (10.18M EGP) as the dominant waste channel
- January is consistently the highest-waste month across the network

### Branch Performance
- Branch_03 leads the network on both revenue and profit (91M EGP net profit)
- Branch_01 shows the highest waste among top-performing branches despite strong profit — a waste-efficiency gap worth investigating

### Machine Learning
- Random Forest achieves **98.50% accuracy** and **0.90 macro F1**, confirming the five input features (ProductKey, BranchKey, Sales_Quantity, Revenue, Total_Cost) carry genuine predictive signal for waste-and-profit risk
- Model errors concentrate almost entirely at the High/Medium Risk boundary — consistent with these being genuinely ambiguous, borderline-risk transactions rather than gross misclassifications

---

## Learning Outcomes

| Skill | Application |
|---|---|
| **Data Pipeline Design** | Medallion Architecture (Bronze → Silver → Gold) |
| **Data Modeling** | Star schema with 1 fact table and 3 dimension tables |
| **Data Engineering** | Pandas transformations, KPI derivation, zero-row-loss cleaning |
| **Machine Learning** | Random Forest classification, rule-based label construction, confusion matrix analysis |
| **Data Visualization** | 7-page interactive Power BI dashboard with KPI cards, donut/bar/line charts |
| **Dashboard Design** | Figma wireframing for a dark-theme design system and icon library |
| **Interactive Tooling** | ipywidgets-based what-if risk prediction simulator |

---

## Data Source & Attribution

**Dataset:** Synthetic branch-level pharmaceutical retail transactions (13 branches × 50 drugs, 821,250 rows)

**Tools:** Python (Pandas, scikit-learn) · Power BI · Figma · Jupyter / Google Colab

**Initiative:** Digital Egypt Pioneers Initiative — Data Analytics Track · 2025 / 2026

---

## Summary

This **Pharmaceutical Waste & Profit Leakage Analysis** delivers:

**Complete 3-Layer Data Pipeline** — Bronze, Silver, Gold via Pandas notebooks
**7 Interactive Power BI Dashboard Pages** — Landing, Profitability, Waste & Loss, Branch Performance, Inventory, Executive Summary, Style Guide
**ML Transaction Risk Classifier** — Random Forest with 98.50% accuracy
**Transparent Feature Engineering** — 8 derived financial/waste KPIs, rule-based risk scoring
**Interactive What-If Tool** — ipywidgets-based risk prediction simulator
