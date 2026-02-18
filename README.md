# 🩺 The Diabetes Belt: A Data-Driven Public Health Investigation

> Exploring how obesity and physical inactivity predict diabetes prevalence across U.S. states (2011–2021)

**[ View Tableau Storyboard](#)** · **[📄 Portfolio Case Study](#)**

---

## Project Overview

This project investigates geographic and behavioral drivers of diabetes prevalence across all 50 U.S. states and DC using 11 years of CDC surveillance data. The analysis moves from exploratory correlation analysis through regression modeling, K-Means clustering, geospatial visualization, and time series — building toward a full public health narrative about why some states suffer from dramatically higher diabetes rates than others.

The analysis was completed as the capstone Achievement 6 project for the **CareerFoundry Data Analytics Program**.

---

## Research Questions & Hypotheses

| | Question |
|---|---|
| **Q1** | Are obesity and physical inactivity both significant predictors of diabetes prevalence? |
| **Q2** | Which is the stronger predictor — obesity or physical inactivity? |
| **Q3** | Do Southern states form a distinct high-risk geographic cluster? |

| Hypothesis | Statement | Result |
|---|---|---|
| **H1** | Physical inactivity is a stronger predictor of diabetes prevalence than obesity | ✅ Confirmed |
| **H2** | Southern states form a distinct high-risk geographic cluster | ✅ Confirmed |

---

## Key Findings

- **Physical inactivity (R² = 0.69)** explains ~10 additional percentage points of state-level diabetes variation compared to obesity (R² = 0.59). Both are highly significant (p < 0.001).
- **K-Means clustering (k=4)** automatically grouped Mississippi, Alabama, Arkansas, Louisiana, Kentucky, Oklahoma, and South Carolina into a single High Risk cluster — *without geographic information as input*. The algorithm found the regional pattern purely from health data.
- The **High Risk vs. Low Risk gap (~5.5 percentage points)** has remained stable from 2011–2021. These are structural, not temporary, disparities.
- **Obesity is rising consistently** with no plateau in sight through 2021, predicting continued upward pressure on diabetes rates.

---

## Data Sources

### Primary Dataset
**CDC Chronic Disease Indicators (CDI) — 2023 Release**
- Source: [data.cdc.gov](https://data.cdc.gov/Chronic-Disease-Indicators)
- Coverage: All 50 U.S. states + DC, 2011–2021
- Raw records: 22,703 rows × 34 columns
- Three indicators used:
  - `Prevalence of diagnosed diabetes among adults aged >= 18 years`
  - `Obesity among adults aged >= 18 years`
  - `No leisure-time physical activity among adults aged >= 18 years`
- Stratification: Overall (no demographic subgroups) for cross-state comparability
- Collected via: Behavioral Risk Factor Surveillance System (BRFSS)

### Geospatial Data
- U.S. state boundaries GeoJSON used for choropleth maps in Notebook 03
- Source: [PublicaMundi / MappingAPI](https://github.com/PublicaMundi/MappingAPI/blob/master/data/geojson/us-states.json)

---

## Repository Structure

```
diabetes-belt-analysis/
│
├── data/
│   ├── raw/
│   │   └── U.S._Chronic_Disease_Indicators__CDI___2023_Release.csv
│   └── prepared/
│       ├── cdi_overall_wide.csv          # Pivoted wide format (state × year)
│       ├── cdi_demographic.csv           # Race/ethnicity breakdown
│       ├── cdi_clusters.csv              # K-Means cluster assignments
│       ├── cdi_time_series.csv           # National average trends
│       ├── cdi_cluster_trends.csv        # Cluster-level trends over time
│       └── regression_results.csv        # R², slopes, p-values
│
├── notebooks/
│   ├── Notebook_01_EDA.ipynb             # Exploratory data analysis & correlations
│   ├── Notebook_02_Data_Cleaning.ipynb   # Cleaning, filtering, data prep
│   ├── Notebook_03_Geospatial.ipynb      # Choropleth maps (folium)
│   ├── Notebook_04_Regression.ipynb      # Linear regression (H1)
│   ├── Notebook_05_Cluster.ipynb         # K-Means clustering (H2)
│   └── Notebook_06_Time_Series.ipynb     # Trend analysis 2011–2021
│
├── visualizations/
│   ├── correlation_heatmap.png
│   ├── regression_scatter.png
│   ├── regression_residuals.png
│   ├── cluster_elbow_plot.png
│   ├── cluster_scatter.png
│   ├── cluster_profiles.png
│   ├── ts_national_trends.png
│   ├── ts_cluster_diabetes_trends.png
│   ├── ts_state_comparison.png
│   └── ts_yoy_change.png
│
└── README.md
```

---

## Methodology

### 1. Exploratory Data Analysis (Notebook 01)
- Correlation heatmap across all three indicators
- Scatterplots and pair plots
- Regional box plots (Northeast, Midwest, South, West)
- Key finding: All pairwise correlations exceed r = 0.72

### 2. Data Cleaning (Notebook 02)
- Filtered to the correct BRFSS diabetes question (critical fix — raw data contains 20 different diabetes questions)
- Removed U.S. territories (GU, PR, VI, AS, MP) and national aggregate rows
- Pivoted to wide format with one row per state-year
- Final dataset: 450 complete rows across 11 years × 41 states

### 3. Geospatial Analysis (Notebook 03)
- Three choropleth maps (folium) with rich tooltips
- Demographic breakdown by race/ethnicity and gender
- Key finding: All three maps independently reveal a "Diabetes Belt" in the Southeast

### 4. Regression Analysis (Notebook 04)
- Simple linear regression using `scipy.stats.linregress`
- Cross-sectional analysis on 2020 data (n = 41 states)
- Obesity → Diabetes: R² = 0.5859, slope = 0.31, p < 0.001
- Inactivity → Diabetes: R² = 0.6858, slope = 0.40, p < 0.001
- Residual plots confirm linear assumptions hold

### 5. Cluster Analysis (Notebook 05)
- K-Means with StandardScaler normalization
- Elbow method selected k = 4
- 3-year average (2019–2021) used to smooth noise
- Cluster assignments: Low Risk (8), Moderate-Active (12), Moderate Risk (15), High Risk (7)

### 6. Time Series Analysis (Notebook 06)
- National average trends 2011–2021
- Cluster-level trajectories — gap between High Risk and Low Risk stable over 11 years
- Mississippi vs. Colorado comparison across all three indicators
- Year-over-year change analysis

---

## Tools & Libraries

| Tool | Purpose |
|---|---|
| Python 3.13 | Primary analysis language |
| pandas | Data manipulation and cleaning |
| numpy | Numerical operations |
| matplotlib / seaborn | Static visualizations |
| scipy.stats | Linear regression |
| scikit-learn | K-Means clustering, StandardScaler |
| folium | Interactive choropleth maps |
| Tableau Public | Final storyboard and interactive dashboard |
| Jupyter Notebook | Analysis environment |

---

## How to Run

1. Clone this repository
2. Install dependencies:
   ```bash
   pip install pandas numpy matplotlib seaborn scipy scikit-learn folium jupyter
   ```
3. Download the raw CDI dataset from [data.cdc.gov](https://data.cdc.gov/Chronic-Disease-Indicators) and place it in `data/raw/`
4. Run notebooks in order: 01 → 02 → 03 → 04 → 05 → 06

---

## Tableau Storyboard

The Tableau storyboard presents the final results in an interactive format. It covers: introduction, data overview, EDA correlations, geospatial maps, regression results (H1), cluster results (H2), time series trends, and limitations.

**[View the full storyboard on Tableau Public →](#)**

---

## Limitations

- **Ecological fallacy:** State-level associations cannot be directly applied to individual-level conclusions
- **Self-report bias:** BRFSS data relies on self-reported height/weight and activity
- **COVID-19 disruption:** 2020–2021 physical inactivity data reflects pandemic-era behavior changes
- **Unmeasured confounders:** Poverty, food access, and healthcare coverage explain additional variance
- **Cross-sectional regression:** The 2020 snapshot cannot establish causality

---

## Author

**Cam** — Health Science Specialist → Data Analyst  
Master of Health Sciences, Lindenwood University  
CareerFoundry Data Analytics Program — Achievement 6

[LinkedIn](#) · [Portfolio](#) · [Tableau Public](#)

---

*Data source: CDC Chronic Disease Indicators (CDI), 2023 Release. Retrieved from data.cdc.gov.*


