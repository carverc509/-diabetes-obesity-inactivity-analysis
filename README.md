# The Diabetes-Obesity-Inactivity Triangle
### Predicting U.S. State-Level Diabetes Prevalence from Lifestyle Risk Factors

> **CareerFoundry Data Immersion — Achievement 6: Advanced Analytics & Dashboard Design**

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Hypothesis](#hypothesis)
3. [Dataset](#dataset)
4. [Design Decisions](#design-decisions)
5. [Data Cleaning Process](#data-cleaning-process)
6. [Reasons for Modifying the Data](#reasons-for-modifying-the-data)
7. [Analysis Pipeline](#analysis-pipeline)
8. [Repository Structure](#repository-structure)
9. [How to Run](#how-to-run)
10. [Tools & Libraries](#tools--libraries)
11. [Limitations](#limitations)
12. [Next Steps](#next-steps)

---

## Project Overview

This project investigates the relationship between lifestyle risk factors — specifically **adult obesity rates** and **physical inactivity rates** — and **diagnosed diabetes prevalence** across all U.S. states from 2011 to 2021. Using the CDC's U.S. Chronic Disease Indicators (CDI) 2023 Release, the analysis moves from exploratory data visualization through geospatial mapping, regression modeling, cluster analysis, and time-series decomposition. Results are presented in an interactive Tableau dashboard designed to tell the full analytical story.

---

## Hypothesis

> **States with higher rates of obesity and physical inactivity among adults will have significantly higher age-adjusted prevalence of diagnosed diabetes. Southern states will form a distinct high-risk cluster (the "Diabetes Belt"), and this relationship will hold across demographic subgroups defined by gender and race/ethnicity.**

### Sub-hypotheses
- **H1 (Regression):** Obesity prevalence is the strongest single predictor of diabetes prevalence at the state level.
- **H2 (Geographic):** Southern states consistently show the highest diabetes prevalence, forming a spatially coherent cluster.
- **H3 (Temporal):** National diabetes prevalence has increased monotonically from 2011–2021 despite declining obesity rates in some states.
- **H4 (Demographic):** The obesity–diabetes relationship is stronger among non-Hispanic Black adults than among non-Hispanic White adults.

---

## Dataset

| Attribute | Details |
|---|---|
| **Source** | CDC — U.S. Chronic Disease Indicators (CDI), 2023 Release |
| **URL** | https://data.cdc.gov/Chronic-Disease-Indicators/U-S-Chronic-Disease-Indicators/g4ie-h725 |
| **License** | Public Domain (U.S. Government open data) |
| **Original file** | `U.S._Chronic_Disease_Indicators__CDI___2023_Release.csv` |
| **Rows (original)** | ~1,100,000+ |
| **Columns (original)** | 34 |
| **Year range** | 2001–2021 (analysis uses 2011–2021) |
| **Geographic coverage** | 50 U.S. states + DC + territories + national aggregate |

### Key columns used

| Column | Role |
|---|---|
| `YearStart` | Temporal dimension |
| `LocationAbbr` / `LocationDesc` | State identifier |
| `Topic` | Filter: Diabetes, Nutrition/Physical Activity/Weight Status |
| `Question` | Filter: specific indicators (see below) |
| `DataValueType` | Filter: Age-adjusted Prevalence |
| `DataValue` | Primary numeric outcome/predictor |
| `LowConfidenceLimit` / `HighConfidenceLimit` | Uncertainty bands |
| `Stratification1` | Demographic subgroup (Overall, Male, Female, race/ethnicity) |
| `GeoLocation` | POINT(lon lat) — parsed to lat/lon for geospatial analysis |
| `LocationID` | FIPS code — joined to U.S. state shapefile |

### Indicators analyzed

| Indicator | TopicID | QuestionID | DataValueType |
|---|---|---|---|
| Diabetes prevalence (adults ≥18) | `DIA` | `DIA4_1` | Age-adjusted Prevalence |
| Obesity prevalence (adults ≥18) | `NPAW` | `NPAW7_2` | Age-adjusted Prevalence |
| Physical inactivity (adults ≥18) | `NPAW` | `NPAW5_1` | Age-adjusted Prevalence |

---

## Design Decisions

### 1. Scope: State-level analysis only
**Decision:** Exclude territory-level rows (Puerto Rico, Guam, U.S. Virgin Islands) and the national aggregate (`LocationAbbr == 'US'`).

**Reason:** The project requires a geographic component with at least 2 values; 50 states + DC provides rich geographic variation. Territories have incomplete data across many years and indicators, and the national aggregate would create a pseudo-observation that skews spatial statistics. The analysis is conducted at the state level to enable choropleth mapping against a standard U.S. state shapefile.

### 2. One DataValueType: Age-adjusted Prevalence
**Decision:** Filter all three indicators (diabetes, obesity, inactivity) to `DataValueType == 'Age-adjusted Prevalence'` only.

**Reason:** The dataset contains multiple representations of each indicator (Number, Crude Prevalence, Crude Rate, Age-adjusted Prevalence). Mixing types in a regression would conflate absolute counts with rates and introduce serious confounding. Age-adjusted prevalence is the most appropriate for comparing across states with different demographic compositions — older states would otherwise appear to have more disease simply due to their age structure. Using the same `DataValueType` for all three indicators ensures apples-to-apples comparisons.

### 3. Year range: 2011–2021
**Decision:** Restrict to years 2011–2021 for the primary analysis.

**Reason:** BRFSS (Behavioral Risk Factor Surveillance System) — the source for all three indicators — underwent a methodological redesign in 2011 (new weighting methodology). Data from before 2011 is not directly comparable to post-2011 data. Restricting to 2011–2021 gives 11 years of comparable annual data and covers the most recent decade, satisfying the "no more than 3 years old" data requirement.

### 4. Separate "Overall" stratification for regression/clustering
**Decision:** For regression and cluster analysis, use only rows where `Stratification1 == 'Overall'`.

**Reason:** Including gender and race/ethnicity rows alongside "Overall" rows would triple-count states in the dataset (each state would appear multiple times per year with different DataValues). This would violate regression assumptions of independence and distort cluster centroids. Demographic breakdowns are analyzed separately in the exploratory and geospatial sections to answer sub-hypothesis H4.

### 5. Pivot structure
**Decision:** Transform from long format (one row per indicator per state per year) to wide format (one row per state per year, with diabetes, obesity, and inactivity as separate columns).

**Reason:** The original CDI dataset is in a long/tidy format optimized for database storage. Regression analysis requires each observation (state-year) to have predictor and outcome values in separate columns. The pivot also makes it immediately apparent when a state-year observation is missing any of the three required values, simplifying completeness checks.

### 6. Geospatial: FIPS-code join to shapefile
**Decision:** Use `LocationID` (numeric FIPS code) to join to a U.S. Census Bureau TIGER/Line state shapefile rather than relying on the `GeoLocation` POINT column.

**Reason:** The `GeoLocation` column contains centroid points only, which cannot draw state polygon boundaries needed for a choropleth. A FIPS-based join to the TIGER/Line shapefile produces accurate state polygons. FIPS codes are stable, standardized identifiers that merge cleanly without risk of name-matching errors (e.g., "New Mexico" vs. "New mexico").

---

## Data Cleaning Process

The cleaning pipeline is implemented in **`notebooks/02_data_cleaning.ipynb`**. Below is a summary of each step.

### Step 1 — Load and initial inspection
- Load the raw CSV with `pd.read_csv()` using `low_memory=False` to avoid mixed-type inference warnings.
- Log the shape, dtypes, and null counts for all 34 columns.
- Identify columns with >90% missing values (`StratificationCategory2/3`, `StratificationID2/3`, `Response`, `ResponseID`) — flagged for potential removal.

### Step 2 — Filter to relevant topics and indicators
- Retain only rows where `TopicID` is in `['DIA', 'NPAW']` (Diabetes and Nutrition/Physical Activity/Weight Status).
- Within those topics, filter `QuestionID` to the three target indicators: `DIA4_1`, `NPAW7_2`, `NPAW5_1`.
- **Rows retained:** ~3–5% of original dataset; all others dropped.

### Step 3 — Filter DataValueType
- Retain only `DataValueType == 'Age-adjusted Prevalence'`.
- Drop all rows with other DataValueTypes (Number, Crude Prevalence, Crude Rate, etc.).

### Step 4 — Geographic filtering
- Drop rows where `LocationAbbr == 'US'` (national aggregate).
- Drop rows where `LocationAbbr` is in `['PR', 'GU', 'VI']` (territories).
- Retain all 50 states + DC (`LocationID` in 1–56, excluding territories).

### Step 5 — Year filtering
- Retain only rows where `YearStart >= 2011`.
- For indicators with multi-year windows (e.g., `YearStart != YearEnd`), flag and handle separately. For this dataset, all three BRFSS-sourced indicators are single-year; no multi-year averaging is needed.

### Step 6 — Handle missing DataValues
- Rows where `DataValue` is NaN are dropped. These correspond to `DatavalueFootnote` values of "No data available" or "Data not shown because of too few respondents or cases."
- **Reason for dropping rather than imputing:** The analysis requires actual measured values; imputing missing state-year prevalence data would introduce fabricated observations into a public health dataset. Missing values are documented in the limitations section of the dashboard.

### Step 7 — Remove non-numeric DataValue entries
- A small subset of rows contains text strings in `DataValue` (e.g., alcohol policy categories). After filtering to the three target indicators, none should remain — this step is a safety check with an assertion.

### Step 8 — Cast DataValue to float
- Convert `DataValue` and `DataValueAlt` from object/string to `float64` to ensure correct numerical operations.

### Step 9 — Parse GeoLocation
- Extract latitude and longitude from the `GeoLocation` column (`"POINT (lon lat)"` format) using a regex.
- Store as separate `longitude` and `latitude` columns (float64).
- Rows where `GeoLocation` is null (typically the national aggregate, already removed) are dropped.

### Step 10 — Stratification split
- Split the cleaned dataset into two subsets:
  - `df_overall`: rows where `Stratification1 == 'Overall'` — used for regression, clustering, and time-series.
  - `df_demo`: rows where `StratificationCategory1` is in `['Gender', 'Race/Ethnicity']` — used for demographic breakdowns in EDA.

### Step 11 — Pivot to wide format (for df_overall)
- Pivot `df_overall` so each row = one state + one year.
- Columns: `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence`, plus `LocationAbbr`, `LocationID`, `latitude`, `longitude`, `YearStart`.
- Rows with any NaN in the three prevalence columns after pivoting are dropped (listwise deletion for complete case analysis).

### Step 12 — Export prepared datasets
- `data/prepared/cdi_overall_wide.csv` — wide-format state-year panel (regression, clustering, time-series).
- `data/prepared/cdi_overall_long.csv` — long-format state-year-indicator (alternative uses, Tableau import).
- `data/prepared/cdi_demographic.csv` — demographic stratification data (EDA, Tableau demographic charts).

---

## Reasons for Modifying the Data

| Modification | Reason |
|---|---|
| Dropped 95%+ of rows (topic/indicator filtering) | The CDI dataset covers ~19 chronic disease topics; only 3 indicators across 2 topics are relevant to this hypothesis. Retaining irrelevant rows wastes memory and would contaminate analysis. |
| Dropped non-Age-adjusted DataValueTypes | Ensures all numeric comparisons use the same scale; prevents confounding by population age structure differences across states. |
| Dropped national aggregate rows | National-level observations are not independent of state-level rows and cannot be mapped in a state-level geospatial analysis. |
| Dropped territory rows | Territories have sparse data and are not part of standard U.S. state shapefiles; including them would break the geospatial join. |
| Dropped pre-2011 rows | BRFSS methodology changed in 2011; pre-2011 values are not comparable to post-2011 values on the same scale. |
| Dropped rows with missing DataValue | Cannot impute public health surveillance data without fabricating observations; missing values are documented as a project limitation. |
| Parsed GeoLocation to lat/lon | The original POINT string format is not usable by geospatial libraries (geopandas, folium) or Tableau's geographic roles without parsing. |
| Pivoted long→wide | Regression and clustering require one observation per state-year with all variables as columns; long format would produce incorrect results in scikit-learn and statsmodels. |
| Exported separate demographic subset | Keeping demographic rows in the main analytical dataset would corrupt regression (multiple rows per state-year) while still being needed for EDA demographic charts. |

---

## Analysis Pipeline

| Notebook | Analysis | Key Output |
|---|---|---|
| `01_data_exploration.ipynb` | Scatterplots, correlation heatmaps, pair plots, categorical bar charts | `visualizations/eda_*.png` |
| `02_data_cleaning.ipynb` | Full cleaning pipeline (see above) | `data/prepared/*.csv` |
| `03_geospatial_analysis.ipynb` | Choropleth maps using geopandas + U.S. state shapefile | `visualizations/map_*.png` |
| `04_regression_analysis.ipynb` | OLS regression: diabetes ~ obesity + inactivity | `reports/regression_results.txt` |
| `05_cluster_analysis.ipynb` | K-means clustering of states by all 3 indicators | `visualizations/cluster_*.png` |
| `06_time_series_analysis.ipynb` | National trend, decomposition, stationarity test | `visualizations/ts_*.png` |

---

## Repository Structure

```
Project folder 6/
├── README.md                          ← You are here
├── .gitignore
│
├── data/
│   ├── original/                      ← Raw CDC CSV (not tracked by git — too large)
│   └── prepared/                      ← Cleaned datasets output by notebook 02
│       ├── cdi_overall_wide.csv
│       ├── cdi_overall_long.csv
│       └── cdi_demographic.csv
│
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_data_cleaning.ipynb
│   ├── 03_geospatial_analysis.ipynb
│   ├── 04_regression_analysis.ipynb
│   ├── 05_cluster_analysis.ipynb
│   └── 06_time_series_analysis.ipynb
│
├── visualizations/                    ← All saved plot images
│   ├── eda_scatterplot_obesity_vs_diabetes.png
│   ├── eda_correlation_heatmap.png
│   ├── eda_pairplot.png
│   ├── eda_barplot_by_race.png
│   ├── map_diabetes_choropleth.png
│   ├── map_obesity_choropleth.png
│   ├── cluster_kmeans_scatter.png
│   ├── cluster_elbow_plot.png
│   └── ts_national_trend.png
│
├── reports/
│   └── regression_results.txt
│
└── tableau_guide/
    └── tableau_instructions.md        ← Step-by-step Tableau dashboard guide
```

---

## How to Run

### Prerequisites
```bash
pip install pandas numpy matplotlib seaborn scikit-learn statsmodels geopandas folium jupyter
```

### Steps
1. Place the raw CSV in `data/original/U.S._Chronic_Disease_Indicators__CDI___2023_Release.csv`
2. Open JupyterLab or Jupyter Notebook in this folder
3. Run notebooks **in order** (01 → 06); notebook 02 must complete before any later notebook
4. After notebook 02, the prepared CSVs will be in `data/prepared/` and ready for Tableau import

---

## Tools & Libraries

| Tool | Purpose |
|---|---|
| Python 3.10+ | Primary analysis language |
| pandas | Data loading, cleaning, reshaping |
| numpy | Numerical operations |
| matplotlib / seaborn | Static visualizations |
| scikit-learn | K-means clustering, preprocessing |
| statsmodels | OLS regression, time-series decomposition |
| geopandas | Geospatial joins and choropleth maps |
| folium | Interactive geographic maps |
| Tableau Public / Desktop | Final interactive dashboard |
| Jupyter Notebook | Reproducible analytical scripts |
| Git | Version control |

---

## Limitations

- **Ecological fallacy:** All analysis is at the state level. A high state-level correlation between obesity and diabetes does not prove the relationship holds at the individual level.
- **Missing data:** Some state-year-indicator combinations have suppressed values (too few respondents). These are dropped via listwise deletion and documented in the dashboard.
- **Confounders not modeled:** Socioeconomic status, healthcare access, food environment, and genetic factors are not included in the regression model.
- **BRFSS self-reported data:** Obesity and diabetes prevalence are based on self-reported survey responses, which may underestimate true prevalence.
- **No county-level analysis:** State-level data masks substantial within-state variation; county-level CDI data would provide finer geographic resolution.
- **Correlation ≠ causation:** Regression findings show associations; causal inference would require longitudinal individual-level data.

---

## Next Steps

- Obtain county-level data from CDC PLACES to drill down to sub-state geographic variation.
- Add socioeconomic covariates (poverty rate, food desert index, healthcare access) as additional regression predictors.
- Build a panel regression model with state fixed effects to control for unobserved time-invariant state characteristics.
- Analyze the lag relationship: does a change in obesity in year *t* predict a change in diabetes in year *t+k*?
- Extend the clustering to include additional chronic disease indicators (cardiovascular, COPD) for a broader state health profile.

---

*Data source: Centers for Disease Control and Prevention (CDC), U.S. Chronic Disease Indicators (CDI), 2023 Release. Data accessed February 2026.*
