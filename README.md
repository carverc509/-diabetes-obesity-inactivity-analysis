# 🩺 # The Diabetes-Obesity-Inactivity Triangle
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
11. [Tableau Dashboard Guide](#tableau-dashboard-guide)
12. [Limitations](#limitations)
13. [Demographic Data Corrections](#demographic-data-corrections)
14. [Next Steps](#next-steps)

---

## Project Overview

This project investigates the relationship between lifestyle risk factors — specifically **adult obesity rates** and **physical inactivity rates** — and **diagnosed diabetes prevalence** across all U.S. states from 2011 to 2021. Using the CDC's U.S. Chronic Disease Indicators (CDI) 2023 Release, the analysis moves from exploratory data visualization through geospatial mapping, regression modeling, cluster analysis, and time-series decomposition. Results are presented in an interactive Tableau dashboard designed to tell the full analytical story.

🔗 **[View the Interactive Dashboard on Tableau Public](https://public.tableau.com/views/MappingtheDiabetesBeltU_S_State-LevelRiskFactors20112021/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**

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
- `data/prepared/cdi_demographic.csv` — demographic stratification data (original, contains all race/ethnicity groups).
- `data/prepared/cdi_demographic_corrected.csv` — corrected demographic data with filtered race/ethnicity groups and proper prevalence values (used in Tableau and portfolio).

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
│       ├── cdi_demographic.csv
│       ├── cdi_demographic_corrected.csv   ← Corrected race/ethnicity data
│       ├── cdi_clusters.csv                ← K-Means cluster assignments
│       └── cdi_cluster_trends.csv          ← Cluster-level trends over time
│
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_data_cleaning.ipynb
│   ├── 03_geospatial_analysis.ipynb
│   ├── 04_regression_analysis.ipynb
│   ├── 05_cluster_analysis.ipynb
│   ├── 06_time_series_analysis.ipynb
│   └── Demographic_Stat_Fixes_for_Notebook_03.ipynb  ← Race/ethnicity data corrections
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

## Tableau Dashboard Guide

🔗 **[View the published dashboard: Mapping the Diabetes Belt](https://public.tableau.com/views/MappingtheDiabetesBeltU_S_State-LevelRiskFactors20112021/Story1?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**

> **Run notebooks 01 and 02 first.** The CSV files below do not exist until notebook 02 completes.

---

### Which File to Load Into Tableau

Load **`cdi_overall_wide.csv`** first — it powers most of the dashboard.

| Load Order | File | Used For |
|---|---|---|
| 1st (main) | `data/prepared/cdi_overall_wide.csv` | Maps, regression, clustering, time-series |
| 2nd | `data/prepared/cdi_demographic_corrected.csv` | Race/ethnicity bar charts |
| 3rd (optional) | `data/prepared/cdi_overall_long.csv` | Multi-indicator line charts |

You also need the **U.S. state shapefile** for the choropleth maps:
- Download from: https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html
- File: `tl_2020_us_state.shp` — place in `data/shapefiles/`

---

### Stage 1 — Connect Data Sources (3 connections)

**Connection 1: cdi_overall_wide.csv**
1. Open Tableau → **Connect to Data → Text File** → select `cdi_overall_wide.csv`
2. Set field types:
   - `YearStart` → Number (Whole)
   - `latitude` → Number (Decimal) → right-click → **Geographic Role → Latitude**
   - `longitude` → Number (Decimal) → right-click → **Geographic Role → Longitude**
   - `LocationAbbr` → right-click → **Geographic Role → State/Province**
   - `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence` → Number (Decimal)
3. Name this connection: **"CDI State-Year Panel"**

**Connection 2: cdi_demographic_corrected.csv**
1. Click **Add** → **Text File** → select `cdi_demographic_corrected.csv`
2. Verify: `Stratification1` = String, `prevalence` = Number (Decimal), `indicator` = String
3. Name: **"CDI Demographics"**
4. **Note:** This corrected file excludes American Indian/Alaska Native and Asian/Pacific Islander groups due to unreliable BRFSS sample sizes. See [Demographic Data Corrections](#demographic-data-corrections) for details.

**Connection 3: Shapefile**
1. Click **Add** → **Spatial File** → select `tl_2020_us_state.shp`
2. Join to `cdi_overall_wide` on: `LocationID = STATEFP` (Inner Join)
3. Name: **"CDI + Shapefile"**

> **Tableau Public users:** Skip the shapefile. Use the built-in filled map by setting `LocationAbbr` Geographic Role to State/Province.

---

### Stage 2 — Create 4 Calculated Fields

Create these in the **CDI State-Year Panel** data source before building any charts.

**Diabetes Risk Category**
```
IF [diabetes_prevalence] >= 13 THEN "High Risk (≥13%)"
ELSEIF [diabetes_prevalence] >= 10 THEN "Moderate Risk (10–13%)"
ELSE "Lower Risk (<10%)"
END
```

**Region**
```
IF [LocationAbbr] IN ("CT","ME","MA","NH","RI","VT","NJ","NY","PA") THEN "Northeast"
ELSEIF [LocationAbbr] IN ("IL","IN","MI","OH","WI","IA","KS","MN","MO","NE","ND","SD") THEN "Midwest"
ELSEIF [LocationAbbr] IN ("DE","FL","GA","MD","NC","SC","VA","WV","DC","AL","KY","MS","TN","AR","LA","OK","TX") THEN "South"
ELSE "West"
END
```

**Year as Date**
```
MAKEDATE([YearStart], 1, 1)
```

**YoY Change** — Table Calculation on `AVG(diabetes_prevalence)`: Difference → Along Table (Down)

---

### Stage 3 — Build 22 Sheets Across 8 Dashboard Pages

#### Page 1 — Introduction
Text box with project title, hypothesis, and dataset citation. Embed `visualizations/eda_pairplot.png` as an image object. No Tableau worksheet needed.

---

#### Page 2 — EDA: Scatter Plots
*(Filter all sheets to `YearStart = 2020`)*

**Sheet 2A — Obesity vs. Diabetes Scatter Plot**
1. `obesity_prevalence` → Columns
2. `diabetes_prevalence` → Rows
3. `LocationAbbr` → Detail, `Region` → Color, `LocationAbbr` → Label (Min/Max only)
4. Mark type: Circle
5. Right-click axes → Add Reference Line → Average on both axes
6. Title: *"Obesity Rate vs. Diabetes Prevalence by State (2020)"*

**Sheet 2B — Inactivity vs. Diabetes Scatter Plot**
- Same as 2A but swap x-axis to `inactivity_prevalence`
- Title: *"Physical Inactivity Rate vs. Diabetes Prevalence by State (2020)"*

**Sheet 2C — Correlation Heatmap**
1. Drag `Measure Names` to Rows and Columns
2. Filter to: `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence`
3. `Measure Values` → Color and Text. Mark type: Square
4. Color palette: Blue-Red Diverging
5. Title: *"Correlation Matrix: Diabetes, Obesity, Physical Inactivity"*

**Sheet 2D — Pair Plot**
- Embed `visualizations/eda_pairplot.png` as an Image object in the dashboard

---

#### Page 3 — EDA: Demographics
*(Use CDI Demographics data source. Filter all sheets to `YearStart = 2020`)*

**Sheet 3A — Diabetes by Race/Ethnicity (Bar Chart)**
1. Filter: `indicator = 'diabetes_prevalence'`, `StratificationCategory1 = 'Race/Ethnicity'`
2. `Stratification1` → Rows, `AVG(prevalence)` → Columns
3. Sort descending. `Stratification1` → Color. Add average reference line.
4. Title: *"Average Diabetes Prevalence by Race/Ethnicity (2020)"*

**Sheet 3B — All 3 Indicators by Gender (Grouped Bar)**
1. Filter: `StratificationCategory1 = 'Gender'`
2. `indicator` → Columns, `AVG(prevalence)` → Rows, `Stratification1` → Color
3. Mark type: Bar
4. Title: *"Diabetes, Obesity, and Inactivity Prevalence by Gender (2020)"*

**Sheet 3C — Diabetes by Region (Box Plot)**
1. `Region` → Columns, `diabetes_prevalence` → Rows
2. Mark type: Circle → Analytics pane → Box Plot
3. `LocationAbbr` → Detail. Sort by median descending.
4. Title: *"Distribution of State Diabetes Prevalence by U.S. Census Region (2020)"*

---

#### Page 4 — Geospatial Maps
*(Filter all sheets to `YearStart = 2020`)*

**Sheet 4A — Diabetes Choropleth**
1. Double-click `LocationAbbr` → auto-generates map
2. `AVG(diabetes_prevalence)` → Color. Palette: Sequential Orange-Red
3. Add `LocationDesc`, all 3 prevalences → Tooltip
4. Title: *"Age-Adjusted Diabetes Prevalence by State (2020)"*

**Sheet 4B — Obesity Choropleth**
- Same as 4A but `AVG(obesity_prevalence)` → Color. Palette: Sequential Blue

**Sheet 4C — Inactivity Choropleth**
- Same as 4A but `AVG(inactivity_prevalence)` → Color. Palette: Sequential Green

**Sheet 4D — Dual-Axis Risk Overlay**
1. Start with Sheet 4A (diabetes choropleth)
2. Drag `longitude` → Columns → right-click → Dual Axis
3. Second axis Marks card: mark type Circle, `AVG(obesity_prevalence)` → Size, color semi-transparent white
4. Title: *"Diabetes Prevalence (color) with Obesity Rate (circle size) by State"*

---

#### Page 5 — Regression Analysis
*(Filter all sheets to `YearStart = 2020`)*

**Sheet 5A — Obesity Regression Trend Line**
1. Recreate Sheet 2A scatter plot
2. Analytics pane → drag **Trend Line** → Linear → show confidence bands
3. Right-click trend line → Describe Trend Line (note R² and p-value)
4. Add R² and p-value as Annotation → Point on chart
5. Title: *"Linear Regression: Obesity Predicts Diabetes Prevalence"*

**Sheet 5B — Inactivity Regression Trend Line**
- Same as 5A but `inactivity_prevalence` on x-axis
- Title: *"Linear Regression: Physical Inactivity Predicts Diabetes Prevalence"*

**Sheet 5C — Residual Plot**
- Embed `visualizations/regression_residuals.png` from notebook 04 as image object

**Sheet 5D — Regression Results Table**
1. Connect `reports/regression_results.csv` as a data source
2. Text Table showing: `term`, `coefficient`, `std_error`, `p_value`, `95%_CI`
3. Title: *"OLS Regression Results: Diabetes Prevalence ~ Obesity + Inactivity"*

---

#### Page 6 — Cluster Analysis
*(After notebook 05: re-export `cdi_overall_wide.csv` with `cluster_label` and `cluster_name` columns added, then refresh the Tableau connection)*

**Sheet 6A — Cluster Scatter Plot**
1. `obesity_prevalence` → Columns, `diabetes_prevalence` → Rows
2. `cluster_name` → Color, `inactivity_prevalence` → Size, `LocationAbbr` → Label
3. Mark type: Circle
4. Title: *"K-Means Clusters: State Health Risk Profiles"*

**Sheet 6B — Cluster Geographic Map**
1. Double-click `LocationAbbr` → map
2. `cluster_name` → Color (Tableau Bold categorical palette)
3. Title: *"Geographic Distribution of K-Means State Health Clusters"*

**Sheet 6C — Cluster Profile Bar Chart**
1. `cluster_name` → Rows
2. Measure Names/Values → show all 3 prevalences as grouped bars, color by indicator
3. Sort by `AVG(diabetes_prevalence)` descending
4. Title: *"Mean Indicator Values by State Health Cluster"*

**Sheet 6D — Elbow Plot**
- Embed `visualizations/cluster_elbow_plot.png` from notebook 05 as image object

---

#### Page 7 — Time-Series Analysis

**Sheet 7A — National Trend Lines (All 3 Indicators)**
1. `Year (as Date)` → Columns (Year level)
2. Measure Values → Rows (filter to all 3 prevalences). Measure Names → Color
3. Mark type: Line. Set inactivity line to dashed style.
4. Title: *"National Average Prevalence Trends: Diabetes, Obesity, Physical Inactivity (2011–2021)"*

**Sheet 7B — State Diabetes Trends Over Time**
1. `Year (as Date)` → Columns, `AVG(diabetes_prevalence)` → Rows
2. `LocationAbbr` → Color. Mark type: Line
3. Highlight Mississippi (highest) and Colorado (lowest)
4. Title: *"State Diabetes Prevalence Trends Over Time (2011–2021)"*

**Sheet 7C — Decomposition Components**
- Embed `visualizations/ts_decomposition.png` from notebook 06 as image object

**Sheet 7D — Year-over-Year Change Bar Chart**
1. `Year (as Date)` → Columns
2. Table Calculation on `AVG(diabetes_prevalence)`: Difference → Along Table (Down) → Rows
3. Mark type: Bar. Color: positive change = red, negative = blue
4. Title: *"Year-over-Year Change in National Average Diabetes Prevalence"*

---

#### Page 8 — Results Summary

**Sheet 8A — State Rankings Table**
1. Filter: `YearStart = 2020`
2. `LocationDesc` → Rows. All 3 prevalences + `cluster_name` → Columns as Text Table
3. Sort by `diabetes_prevalence` descending. Add red gradient conditional formatting on diabetes column.
4. Title: *"2020 State Rankings: Diabetes, Obesity, and Physical Inactivity"*

**Sheet 8B — Hypothesis Scorecard**
Create 4 KPI text tiles in the dashboard layout:

| Hypothesis | Expected Result |
|---|---|
| H1: Obesity is strongest predictor | ✓ Confirmed — fill in R² from notebook 04 |
| H2: Southern states cluster highest | ✓ Confirmed — cluster map shows geographic coherence |
| H3: Diabetes rising nationally 2011–2021 | ✓ Confirmed — ADF test shows non-stationary upward trend |
| H4: Disparity stronger in Black adults | ✓ Partially confirmed — see demographic charts |

**Sheet 8C — Limitations (Text Box)**
```
• Ecological analysis: state-level correlations do not imply individual-level causation.
• Missing data: some state-year observations dropped due to suppressed BRFSS values.
• Self-reported data: BRFSS values may underestimate true prevalence.
• Unmeasured confounders: food environment, healthcare access, and genetics not modeled.
• BRFSS methodology change: data restricted to 2011+ for comparability.
• No county-level analysis: within-state variation is masked.
```

**Sheet 8D — Next Steps (Text Box)**
```
• Integrate county-level data from CDC PLACES for sub-state geographic resolution.
• Add socioeconomic predictors (poverty index, food desert score) to the regression model.
• Build a panel model with state fixed effects to control for unobserved state characteristics.
• Analyze lagged effects: does rising obesity in year t predict rising diabetes in year t+3?
• Extend clustering to include cardiovascular and COPD indicators for a broader health profile.
```

---

### Stage 4 — Dashboard Assembly

| Dashboard Tab | Sheets | Size |
|---|---|---|
| 1 — Introduction | Text + image | 1200 × 800 |
| 2 — EDA: Correlations | 2A, 2B, 2C, 2D | 1200 × 900 |
| 3 — EDA: Demographics | 3A, 3B, 3C | 1200 × 900 |
| 4 — Geospatial | 4A, 4B, 4C, 4D | 1200 × 900 |
| 5 — Regression | 5A, 5B, 5C, 5D | 1200 × 900 |
| 6 — Clustering | 6A, 6B, 6C, 6D | 1200 × 900 |
| 7 — Time-Series | 7A, 7B, 7C, 7D | 1200 × 900 |
| 8 — Summary | 8A, 8B, 8C, 8D | 1200 × 900 |

**Global Filters (apply to all pages):**
- `YearStart` → slider filter
- `Region` → multi-select checkbox filter

**Navigation:** Dashboard → Objects → Navigation → add forward/back buttons on every page.

**Color Scheme:**
- Diabetes → Orange-Red gradient
- Obesity → Blue gradient
- Inactivity → Green gradient
- Clusters → Tableau Bold (4 colors)
- Background → `#FFFFFF`, Font → Tableau Book 12pt body / 18pt titles

---

### Quick Reference: All 22 Charts

| Analysis Step | Chart | Sheet | Hypothesis Supported |
|---|---|---|---|
| EDA | Obesity vs. Diabetes scatter | 2A | H1 |
| EDA | Inactivity vs. Diabetes scatter | 2B | H1 |
| EDA | Correlation heatmap | 2C | H1 |
| EDA | Pair plot (image) | 2D | H1 |
| EDA Demographics | Bar chart by race/ethnicity | 3A | H4 |
| EDA Demographics | Grouped bar by gender | 3B | H4 |
| EDA Demographics | Box plot by region | 3C | H2 |
| Geospatial | Diabetes choropleth | 4A | H2 |
| Geospatial | Obesity choropleth | 4B | H1 geographic |
| Geospatial | Inactivity choropleth | 4C | H1 geographic |
| Geospatial | Dual-axis risk overlay | 4D | H1 + H2 |
| Regression | Obesity trend line + R² | 5A | H1 |
| Regression | Inactivity trend line + R² | 5B | H1 |
| Regression | Residual plot (image) | 5C | Model validity |
| Regression | Results table | 5D | H1 |
| Clustering | Cluster scatter plot | 6A | H2 |
| Clustering | Cluster geographic map | 6B | H2 |
| Clustering | Cluster profile bars | 6C | H2 |
| Clustering | Elbow plot (image) | 6D | K selection |
| Time-Series | National trend lines | 7A | H3 |
| Time-Series | State trend lines | 7B | H3 |
| Time-Series | YoY change bars | 7D | H3 |
| Summary | State rankings table | 8A | All |
| Summary | Hypothesis scorecard | 8B | All |

---

## Demographic Data Corrections

A supplementary notebook — **`Demographic_Stat_Fixes_for_Notebook_03.ipynb`** — documents corrections made to the race/ethnicity analysis in Notebook 03.

### What was changed

The original `cdi_demographic.csv` contained mixed data value types (percentages, crude rates per 100K, and counts) across different CDC questions. When averaged without filtering to a single question type, the race/ethnicity bar chart produced inflated values in the hundreds rather than actual prevalence percentages.

### How it was fixed

1. Filtered the raw CDC data to only the three target prevalence questions (diabetes, obesity, inactivity among adults ≥18)
2. Filtered to `Race/Ethnicity` stratification and excluded territories
3. Excluded **American Indian or Alaska Native** and **Asian or Pacific Islander** groups due to limited state-level sample sizes in BRFSS data, which produced statistically unreliable estimates
4. Recalculated averages and exported as `cdi_demographic_corrected.csv`

### Corrected values (Average Diabetes Prevalence by Race/Ethnicity, 2011–2021)

| Race/Ethnicity | Avg. Prevalence |
|---|---|
| Black, non-Hispanic | 14.3% |
| Multiracial, non-Hispanic | 12.4% |
| Hispanic | 11.7% |
| Other, non-Hispanic | 11.6% |
| White, non-Hispanic | 8.8% |

These corrected values are used in both the Tableau storyboard and the portfolio case study.

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

