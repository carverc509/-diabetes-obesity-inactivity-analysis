# Tableau Dashboard — Step-by-Step Build Guide
## Project: The Diabetes-Obesity-Inactivity Triangle

> This guide tells you exactly what files to import into Tableau, what connections to make,
> and what chart to build at every step of the project — from the opening EDA slide to the
> final summary page. Follow the notebooks first to produce the prepared CSV files, then use
> this guide to build the dashboard.

---

## Prerequisites Before Opening Tableau

Confirm the following files exist in `data/prepared/` after running notebook `02_data_cleaning.ipynb`:

| File | Contents | Used for |
|---|---|---|
| `cdi_overall_wide.csv` | One row per state per year; columns: `YearStart`, `LocationAbbr`, `LocationDesc`, `LocationID`, `latitude`, `longitude`, `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence` | Regression, clustering, time-series charts |
| `cdi_overall_long.csv` | Long format: one row per state-year-indicator; columns: `YearStart`, `LocationAbbr`, `LocationDesc`, `LocationID`, `latitude`, `longitude`, `indicator`, `prevalence` | Maps and multi-indicator line charts |
| `cdi_demographic.csv` | Demographic breakdowns: same columns plus `Stratification1` (Overall / Male / Female / race groups) | Bar charts and demographic comparisons |

You will also need the **U.S. state shapefile** for the geospatial map:
- Download from: https://www.census.gov/geographies/mapping-files/time-series/geo/tiger-line-file.html
- File: `tl_2020_us_state.shp` (or any recent vintage)
- Place in: `data/shapefiles/`

---

## Part 1 — Connect Data Sources in Tableau

### Connection 1: cdi_overall_wide.csv
1. Open Tableau Desktop or Tableau Public.
2. Click **"Connect to Data"** → **"Text File"**.
3. Navigate to `data/prepared/cdi_overall_wide.csv` and open it.
4. In the Data Source tab, verify Tableau has inferred the field types correctly:
   - `YearStart` → **Number (Whole)** — if it shows as Date, right-click and change to Number.
   - `LocationID` → **Number (Whole)**
   - `latitude` → **Number (Decimal)** — then right-click → **Geographic Role → Latitude**
   - `longitude` → **Number (Decimal)** — then right-click → **Geographic Role → Longitude**
   - `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence` → **Number (Decimal)**
   - `LocationAbbr`, `LocationDesc` → **String**
5. Right-click `LocationAbbr` → **Geographic Role → State/Province**.
6. Right-click `LocationDesc` → **Geographic Role → State/Province**.
7. Name this connection: **"CDI State-Year Panel"**

### Connection 2: cdi_demographic.csv
1. In the Data Source tab, click **"Add"** (top left) → **"Text File"** → select `cdi_demographic.csv`.
2. Verify types:
   - `Stratification1` → **String**
   - `prevalence` → **Number (Decimal)**
   - `indicator` → **String**
   - `YearStart` → **Number (Whole)**
3. Name this connection: **"CDI Demographics"**

### Connection 3: Shapefile (for geospatial map)
1. Click **"Add"** → **"Spatial File"** → navigate to `data/shapefiles/tl_2020_us_state.shp`.
2. Tableau will auto-detect geometry.
3. In the data canvas, drag `cdi_overall_wide.csv` and create a **Relationship** (or Join) to the shapefile on:
   - `cdi_overall_wide.LocationID` = `shapefile.STATEFP` (both are numeric FIPS codes)
   - Join type: **Inner Join** (so only states with data appear)
4. Name this connection: **"CDI + Shapefile"**

> **Tip:** Tableau Public does not support Spatial Files. If using Tableau Public, use the
> `latitude` and `longitude` columns from `cdi_overall_wide.csv` with a Symbol Map instead.
> For choropleth polygons on Tableau Public, use the built-in filled map by setting `LocationAbbr`
> Geographic Role to State/Province.

---

## Part 2 — Create Calculated Fields

Before building sheets, create these calculated fields in the **"CDI State-Year Panel"** data source:

### Calculated Field 1: Diabetes Risk Category
```
IF [diabetes_prevalence] >= 13 THEN "High Risk (≥13%)"
ELSEIF [diabetes_prevalence] >= 10 THEN "Moderate Risk (10–13%)"
ELSE "Lower Risk (<10%)"
END
```
*Used for color-coding states on scatter plots and maps.*

### Calculated Field 2: Obesity Quartile
```
IF [obesity_prevalence] >= PERCENTILE([obesity_prevalence], 0.75) THEN "Q4 (Highest)"
ELSEIF [obesity_prevalence] >= PERCENTILE([obesity_prevalence], 0.50) THEN "Q3"
ELSEIF [obesity_prevalence] >= PERCENTILE([obesity_prevalence], 0.25) THEN "Q2"
ELSE "Q1 (Lowest)"
END
```
*Note: Since PERCENTILE works in Table Calcs, alternatively use bins on `obesity_prevalence`.*

### Calculated Field 3: Region
```
IF [LocationAbbr] IN ("CT","ME","MA","NH","RI","VT","NJ","NY","PA") THEN "Northeast"
ELSEIF [LocationAbbr] IN ("IL","IN","MI","OH","WI","IA","KS","MN","MO","NE","ND","SD") THEN "Midwest"
ELSEIF [LocationAbbr] IN ("DE","FL","GA","MD","NC","SC","VA","WV","DC","AL","KY","MS","TN","AR","LA","OK","TX") THEN "South"
ELSE "West"
END
```
*Used for regional categorical breakdowns.*

### Calculated Field 4: Year (as Date)
```
MAKEDATE([YearStart], 1, 1)
```
*Converts the integer year to a proper Date field for time-series axes.*

---

## Part 3 — Dashboard Pages & Charts

Build each sheet below, then assemble them into dashboard pages. The dashboard should have **8 pages** matching the project brief requirements.

---

### PAGE 1 — Introduction
**Purpose:** Introduce the project, dataset, and hypothesis.

**Sheet: Text & Image Layout (use a Dashboard layout, not a Worksheet)**
- Add a text box with the project title, hypothesis statement, and dataset citation.
- Add a small static image (`visualizations/eda_pairplot.png`) as a teaser visual.
- Add a navigation button to Page 2.

**No Tableau chart required — this is a formatted text layout.**

---

### PAGE 2 — Exploratory Data Analysis: Scatter Plots

**Purpose:** Show the raw relationship between the predictor and outcome variables.

---

#### Sheet 2A: Obesity vs. Diabetes Scatter Plot (by State, Latest Year)

**Data source:** CDI State-Year Panel
**Filter:** `YearStart = 2020` (or the most recent complete year in the data)
**Filter:** `Stratification1 = Overall` (if column present; otherwise already filtered in the CSV)

**Steps:**
1. Drag `obesity_prevalence` to **Columns**.
2. Drag `diabetes_prevalence` to **Rows**.
3. Drag `LocationAbbr` to **Detail** (Marks card).
4. Drag `Region` (calculated field) to **Color**.
5. Drag `LocationAbbr` to **Label** (Marks card) — set to "Min/Max" so only extreme states label.
6. Change the mark type to **Circle**.
7. Right-click the x-axis → **Add Reference Line** → Line at **Average** of `obesity_prevalence`.
8. Right-click the y-axis → **Add Reference Line** → Line at **Average** of `diabetes_prevalence`.
9. Title: **"Obesity Rate vs. Diabetes Prevalence by State (2020)"**

**What this shows:** The positive linear relationship between obesity and diabetes at the state level. States in the top-right quadrant (high obesity, high diabetes) are visible. This chart motivates the regression hypothesis.

---

#### Sheet 2B: Physical Inactivity vs. Diabetes Scatter Plot

**Steps (same as 2A but swap x-axis):**
1. Drag `inactivity_prevalence` to **Columns**.
2. Drag `diabetes_prevalence` to **Rows**.
3. Drag `LocationAbbr` to **Detail**, `Region` to **Color**, `LocationAbbr` to **Label**.
4. Add average reference lines on both axes.
5. Title: **"Physical Inactivity Rate vs. Diabetes Prevalence by State (2020)"**

**What this shows:** The secondary predictor relationship. Compare visually with Sheet 2A to show which predictor has a tighter correlation.

---

#### Sheet 2C: Correlation Heatmap (via Crosstab)

*Note: Tableau does not natively render a correlation matrix heatmap. Use a workaround with a Crosstab and color encoding.*

**Approach — use a text table with color:**
1. Create a **Measure Names / Measure Values** sheet.
2. Drag `Measure Names` to **Rows** and **Columns**.
3. Filter Measure Names to: `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence`.
4. Drag `Measure Values` to **Color** and **Text**.
5. Use a **Square** mark type.
6. Set the color palette to **Blue-Red Diverging** (negative correlations blue, positive red).
7. Title: **"Correlation Matrix: Diabetes, Obesity, Physical Inactivity"**

*Alternative: Export the correlation heatmap as `visualizations/eda_correlation_heatmap.png` from notebook 01 and embed the image in the dashboard.*

---

#### Sheet 2D: Pair Plot (embedded image)

- Export `visualizations/eda_pairplot.png` from notebook 01.
- In the dashboard, use **Image** object to embed it.
- Add a caption text box: *"Pair plot of diabetes prevalence, obesity prevalence, and physical inactivity — all three variables show positive pairwise correlations."*

---

### PAGE 3 — Exploratory Data Analysis: Categorical Breakdowns

**Purpose:** Examine how diabetes, obesity, and inactivity vary across demographic groups (sub-hypothesis H4).

---

#### Sheet 3A: Diabetes Prevalence by Race/Ethnicity (Bar Chart)

**Data source:** CDI Demographics
**Filter:** `indicator = 'diabetes_prevalence'`
**Filter:** `StratificationCategory1 = 'Race/Ethnicity'`
**Filter:** `YearStart = 2020`

**Steps:**
1. Drag `Stratification1` to **Rows**.
2. Drag `AVG(prevalence)` to **Columns**.
3. Sort bars descending by `AVG(prevalence)`.
4. Drag `Stratification1` to **Color**.
5. Add a reference line at the national average.
6. Title: **"Average Diabetes Prevalence by Race/Ethnicity (2020)"**

**What this shows:** Which racial/ethnic groups carry the highest diabetes burden. Confirms that non-Hispanic Black and American Indian/Alaska Native adults have disproportionately high prevalence.

---

#### Sheet 3B: Side-by-Side Bar Chart — All 3 Indicators by Gender

**Data source:** CDI Demographics
**Filter:** `StratificationCategory1 = 'Gender'`
**Filter:** `YearStart = 2020`

**Steps:**
1. Drag `indicator` to **Columns**.
2. Drag `AVG(prevalence)` to **Rows**.
3. Drag `Stratification1` to **Color**.
4. Change mark type to **Bar**.
5. Title: **"Diabetes, Obesity, and Inactivity Prevalence by Gender (2020)"**

**What this shows:** Gender differences in all three indicators. Men typically have higher obesity-related diabetes risk despite women having similar or higher obesity rates — useful for dashboard narrative.

---

#### Sheet 3C: Box Plot — Diabetes Prevalence by Region

**Data source:** CDI State-Year Panel
**Filter:** `YearStart = 2020`

**Steps:**
1. Drag `Region` (calculated field) to **Columns**.
2. Drag `diabetes_prevalence` to **Rows**.
3. Change mark type to **Circle** initially, then right-click → **Show Box Plot** (or use the Analytics pane → Box Plot).
4. Sort `Region` by median `diabetes_prevalence` descending.
5. Drag `LocationAbbr` to **Detail** so individual states show as points.
6. Title: **"Distribution of State Diabetes Prevalence by U.S. Census Region (2020)"**

**What this shows:** The South region has both the highest median and the widest spread of diabetes prevalence — supporting the "Diabetes Belt" geographic hypothesis.

---

### PAGE 4 — Geospatial Analysis

**Purpose:** Visualize the geographic distribution of diabetes, obesity, and inactivity across U.S. states.

---

#### Sheet 4A: Filled Map — Diabetes Prevalence by State

**Data source:** CDI State-Year Panel (or CDI + Shapefile connection)
**Filter:** `YearStart = 2020`

**Steps (using built-in geographic roles — works in Tableau Public):**
1. Double-click `LocationAbbr` — Tableau generates a map automatically.
2. Drag `AVG(diabetes_prevalence)` to **Color**.
3. Set color palette to **Sequential: Orange-Red** (light = low, dark = high).
4. Drag `LocationDesc` to **Tooltip**.
5. Drag `diabetes_prevalence` to **Tooltip**.
6. Drag `obesity_prevalence` to **Tooltip**.
7. Drag `inactivity_prevalence` to **Tooltip**.
8. Exclude Alaska and Hawaii for continental focus (optional — use a filter on `LocationAbbr`), or keep them and re-size.
9. Add a **Title:** "Age-Adjusted Diabetes Prevalence by State (2020)"
10. Add a **Caption** below: *"Source: CDC U.S. Chronic Disease Indicators, 2023 Release."*

**What this shows:** The "Diabetes Belt" — a cluster of dark-colored states in the Southeast (Mississippi, West Virginia, Alabama, Tennessee, Louisiana, Arkansas). This is the geographic confirmation of the hypothesis.

---

#### Sheet 4B: Filled Map — Obesity Prevalence by State

**Steps (identical to 4A but swap measure):**
1. Double-click `LocationAbbr`.
2. Drag `AVG(obesity_prevalence)` to **Color**.
3. Use **Sequential: Blue** palette.
4. Title: **"Age-Adjusted Obesity Prevalence by State (2020)"**

**What this shows:** Side-by-side with Sheet 4A, you can see that states with high obesity (dark blue) nearly mirror states with high diabetes (dark red), visually confirming the spatial correlation.

---

#### Sheet 4C: Filled Map — Physical Inactivity by State

**Steps (identical, swap measure):**
1. `AVG(inactivity_prevalence)` to Color.
2. **Sequential: Green** palette.
3. Title: **"Age-Adjusted Physical Inactivity Prevalence by State (2020)"**

---

#### Sheet 4D: Dual-Axis Symbol Map — Risk Overlay

**Purpose:** Overlay obesity rate (circle size) on top of the diabetes choropleth.

**Steps:**
1. Start with Sheet 4A (diabetes choropleth).
2. Drag `longitude` to **Columns** (creates a second map view).
3. Right-click → **Dual Axis**.
4. On the second axis's Marks card: change mark type to **Circle**.
5. Drag `AVG(obesity_prevalence)` to **Size**.
6. Drag `LocationAbbr` to **Detail**.
7. Set circle color to semi-transparent white.
8. Title: **"Diabetes Prevalence (color) with Obesity Rate (circle size) by State"**

**What this shows:** States where both measures are high (dark fill + large circle) are the most at-risk. This chart is the "hero" visualization for the geospatial page.

---

### PAGE 5 — Regression Analysis

**Purpose:** Quantify the relationship between lifestyle factors and diabetes prevalence.

---

#### Sheet 5A: Regression Trend Line — Obesity vs. Diabetes

**Data source:** CDI State-Year Panel
**Filter:** `YearStart = 2020`

**Steps:**
1. Recreate Sheet 2A (Obesity vs. Diabetes scatter plot).
2. Go to **Analytics** pane → drag **Trend Line** onto the view → select **Linear**.
3. Right-click the trend line → **Edit Trend Line** → check "Show confidence bands."
4. Right-click the trend line → **Describe Trend Line** to view the R², p-value, and slope in Tableau's output.
5. Add the R² and p-value as **Annotation → Point** on the chart.
6. Title: **"Linear Regression: Obesity Predicts Diabetes Prevalence (R² shown)"**

**What this shows:** The slope and fit of the obesity–diabetes regression. The R² value and confidence bands are visible on the chart. Quote the statsmodels output from notebook 04 in a text box alongside the chart for the full regression table (coefficients, standard errors, p-values).

---

#### Sheet 5B: Regression Trend Line — Inactivity vs. Diabetes

**Steps (identical to 5A, swap x-axis to `inactivity_prevalence`):**
1. Title: **"Linear Regression: Physical Inactivity Predicts Diabetes Prevalence"**

**What this shows:** Comparison with Sheet 5A. The slope and R² for inactivity will typically be slightly weaker than for obesity, confirming H1 (obesity is the stronger predictor).

---

#### Sheet 5C: Residual Plot (from exported image)

- Export the residual plot from notebook 04 as `visualizations/regression_residuals.png`.
- Embed as an image in the dashboard.
- Caption: *"Residuals of the OLS model: diabetes prevalence ~ obesity + inactivity. Residuals are approximately normally distributed with no strong pattern, confirming model validity."*

---

#### Sheet 5D: Regression Results Table (Text Table)

**Steps:**
1. Create a new sheet with a **Text Table**.
2. Use a parameter or calculated field to display the regression output as a static formatted table.
3. Alternatively, export the regression results from notebook 04 as a CSV (`reports/regression_results.csv`) and connect it as a data source.
4. If using the CSV: connect `regression_results.csv` → display `term`, `coefficient`, `std_error`, `p_value`, `95%_CI` as a text table.
5. Title: **"OLS Regression Results: Diabetes Prevalence ~ Obesity + Inactivity"**

---

### PAGE 6 — Cluster Analysis

**Purpose:** Identify groups of states with similar profiles of diabetes, obesity, and inactivity.

---

#### Sheet 6A: Cluster Scatter Plot — K-Means Result

**Data source:** CDI State-Year Panel (with cluster labels added)

**Important setup step:** After running notebook 05, export the cluster assignments back to the wide CSV:
- Column: `cluster_label` (integer: 0, 1, 2, 3) — add to `cdi_overall_wide.csv`
- Also export `cluster_name` (descriptive: "High Risk", "Moderate Risk", "Lower Risk", "Lean & Inactive") — add to the CSV.

Then in Tableau, **refresh the data connection** so the new columns appear.

**Steps:**
1. Drag `obesity_prevalence` to **Columns**.
2. Drag `diabetes_prevalence` to **Rows**.
3. Drag `cluster_name` to **Color**.
4. Drag `inactivity_prevalence` to **Size**.
5. Drag `LocationAbbr` to **Label**.
6. Change mark type to **Circle**.
7. Title: **"K-Means Clusters: State Health Risk Profiles"**

**What this shows:** Visually distinct groups of states — the "High Risk" cluster (large circles, top-right) will be dominated by Southern states. The cluster colors will appear on the geographic map next (Sheet 6B), demonstrating the spatial coherence of the clusters.

---

#### Sheet 6B: Cluster Map — States Colored by Cluster

**Steps:**
1. Double-click `LocationAbbr` → map.
2. Drag `cluster_name` to **Color** — use a categorical palette (Bold, Tableau 10, or custom).
3. Drag `cluster_name` to **Tooltip**, `diabetes_prevalence` to Tooltip, `obesity_prevalence` to Tooltip.
4. Title: **"Geographic Distribution of K-Means State Health Clusters"**

**What this shows:** Whether the clusters are spatially coherent. The "High Risk" cluster should dominate the South and Appalachia, visually confirming H2.

---

#### Sheet 6C: Cluster Profile Bar Chart (Grouped)

**Steps:**
1. Drag `cluster_name` to **Rows**.
2. Create a **dual view** or use Measure Names/Values to show `AVG(diabetes_prevalence)`, `AVG(obesity_prevalence)`, `AVG(inactivity_prevalence)` as three grouped bars.
3. Color by indicator name.
4. Sort clusters by `AVG(diabetes_prevalence)` descending.
5. Title: **"Mean Indicator Values by State Health Cluster"**

**What this shows:** The "profile" of each cluster — confirms that the "High Risk" cluster has distinctly higher values on all three measures, not just diabetes.

---

#### Sheet 6D: Elbow Plot (embedded image)

- Export `visualizations/cluster_elbow_plot.png` from notebook 05.
- Embed in the dashboard.
- Caption: *"Elbow plot used to select K=4 clusters. Inertia decreases steeply from K=1 to K=4, then plateaus, indicating 4 as the optimal number of clusters."*

---

### PAGE 7 — Time-Series Analysis

**Purpose:** Show how national diabetes, obesity, and inactivity have trended over time.

---

#### Sheet 7A: Multi-Line Time-Series — National Trend (All 3 Indicators)

**Data source:** CDI State-Year Panel
**Filter:** `LocationAbbr = 'US'` — **Wait:** the US aggregate is excluded from the panel. Instead:

**Approach:** Aggregate the state-level data to a national average per year.
1. Drag `Year (as Date)` (calculated field: MAKEDATE) to **Columns** — set to **Year** granularity.
2. Drag `Measure Values` to **Rows** — filter to `diabetes_prevalence`, `obesity_prevalence`, `inactivity_prevalence`.
3. Drag `Measure Names` to **Color**.
4. Change mark type to **Line**.
5. Set the line for inactivity to a dashed style (right-click → Format → Line style).
6. Title: **"National Average Prevalence Trends: Diabetes, Obesity, Physical Inactivity (2011–2021)"**

**What this shows:** The temporal dimension — whether all three indicators are rising, falling, or diverging over the study period. Typically you'll see diabetes and obesity rising while inactivity fluctuates. This supports H3.

---

#### Sheet 7B: State-Level Trend Lines — Diabetes Over Time (Small Multiples)

**Steps:**
1. Drag `Year (as Date)` to **Columns** — Year level.
2. Drag `AVG(diabetes_prevalence)` to **Rows**.
3. Drag `LocationAbbr` to **Color** (or Detail).
4. Change mark type to **Line**.
5. Use **Highlight** on a specific state of interest (e.g., Mississippi = highest; Colorado = lowest) by dragging `LocationAbbr` to Color and using **Highlight Selected**.
6. Title: **"State Diabetes Prevalence Trends Over Time (2011–2021)"**

**What this shows:** Whether some states are improving while others are worsening — tests whether the "Diabetes Belt" gap is widening or narrowing.

---

#### Sheet 7C: Decomposition Components (embedded images)

- Export `visualizations/ts_decomposition.png` from notebook 06 (trend, seasonal, residual components).
- Embed as image in the dashboard.
- Caption: *"Time-series decomposition of national diabetes prevalence 2011–2021 using STL decomposition. The trend component shows a consistent upward trajectory. The Augmented Dickey-Fuller test (p < 0.05) confirms the series is non-stationary — a significant linear time trend is present."*

---

#### Sheet 7D: Year-over-Year Change Bar Chart

**Data source:** CDI State-Year Panel
**Steps:**
1. Create a **Table Calculation** on `AVG(diabetes_prevalence)`: Difference → Along Table (Down).
2. Drag `Year (as Date)` to **Columns**.
3. Drag the calculated field (YoY change) to **Rows**.
4. Change mark type to **Bar**.
5. Color bars: positive change = red, negative change = blue. (Use calculated field: `IF [YoY Change] > 0 THEN "Increase" ELSE "Decrease" END` → Color.)
6. Title: **"Year-over-Year Change in National Average Diabetes Prevalence"**

**What this shows:** In which years was diabetes prevalence growing fastest? Were there any years of improvement?

---

### PAGE 8 — Results Summary & Conclusions

**Purpose:** Tie together all findings, state limitations, and propose next steps.

---

#### Sheet 8A: Final State Rankings Table

**Data source:** CDI State-Year Panel
**Filter:** `YearStart = 2020`

**Steps:**
1. Drag `LocationDesc` to **Rows**.
2. Drag `AVG(diabetes_prevalence)`, `AVG(obesity_prevalence)`, `AVG(inactivity_prevalence)` to **Columns** as a **Text Table**.
3. Add conditional formatting: color `diabetes_prevalence` with a red gradient (higher = darker red).
4. Sort by `diabetes_prevalence` descending.
5. Add `cluster_name` as an additional column for context.
6. Title: **"2020 State Rankings: Diabetes, Obesity, and Physical Inactivity"**

**What this shows:** A quick-reference summary of all states, fully supporting the dashboard narrative. Mississippi, West Virginia, and Alabama will rank highest; Colorado, Hawaii, and Utah will rank lowest.

---

#### Sheet 8B: Hypothesis Confirmation Scorecard (Text Table or KPI tiles)

Create 4 KPI-style tiles in the dashboard layout (using text boxes or BANs — Big Ass Numbers):

| Hypothesis | Result | Visual |
|---|---|---|
| H1: Obesity is strongest predictor | ✓ Confirmed — R² = [value from notebook] | Green badge |
| H2: Southern states cluster highest | ✓ Confirmed — cluster map shows geographic coherence | Green badge |
| H3: Diabetes rising nationally 2011–2021 | ✓ Confirmed — ADF test shows non-stationary upward trend | Green badge |
| H4: Disparity stronger in Black adults | ✓ Partially confirmed — see demographic charts | Yellow badge |

*Replace [value from notebook] with the actual R² from your regression output.*

---

#### Sheet 8C: Limitations Text Block

In the dashboard, add a **Text Box** with:

```
LIMITATIONS
-----------
• Ecological analysis: state-level correlations do not imply individual-level causation.
• Missing data: [N] state-year observations dropped due to suppressed BRFSS values.
• Self-reported data: BRFSS obesity and diabetes values may underestimate true prevalence.
• Unmeasured confounders: food environment, healthcare access, and genetics are not modeled.
• BRFSS methodology change: data restricted to 2011+ for comparability.
• No county-level analysis: within-state variation is masked.
```

---

#### Sheet 8D: Next Steps Text Block

```
NEXT STEPS
----------
• Integrate county-level data from CDC PLACES for sub-state geographic resolution.
• Add socioeconomic predictors (poverty index, food desert score) to the regression model.
• Build a panel model with state fixed effects to control for unobserved state characteristics.
• Analyze lagged effects: does rising obesity in year t predict rising diabetes in year t+3?
• Extend clustering to include cardiovascular and COPD indicators for a broader health profile.
```

---

## Part 4 — Dashboard Assembly Instructions

### Dashboard Layout (8 Pages)

| Dashboard Tab | Sheets Included | Width × Height |
|---|---|---|
| 1 — Introduction | Text boxes, project image | 1200 × 800 |
| 2 — EDA: Correlations | 2A, 2B, 2C, 2D | 1200 × 900 |
| 3 — EDA: Demographics | 3A, 3B, 3C | 1200 × 900 |
| 4 — Geospatial | 4A, 4B, 4C, 4D | 1200 × 900 |
| 5 — Regression | 5A, 5B, 5C, 5D | 1200 × 900 |
| 6 — Clustering | 6A, 6B, 6C, 6D | 1200 × 900 |
| 7 — Time-Series | 7A, 7B, 7C, 7D | 1200 × 900 |
| 8 — Summary | 8A, 8B, 8C, 8D | 1200 × 900 |

### Global Filters (add to all relevant pages)
1. **Year filter:** `YearStart` slider — set to "All Values" default, allow user to change.
2. **Region filter:** `Region` calculated field — multi-select checkboxes.
3. **Indicator filter** (on demographic pages): `indicator` field — multi-select.

### Navigation Buttons
1. On each dashboard, add **Navigation** button objects (Dashboard → Objects → Navigation).
2. Point each button to the next/previous dashboard tab.
3. Style buttons with arrows and page labels (e.g., "→ Geospatial Analysis").

### Color Scheme Consistency
- **Diabetes:** Red / Orange-Red gradient
- **Obesity:** Blue / Blue-Green gradient
- **Inactivity:** Green / Teal gradient
- **Clusters:** Tableau Bold categorical palette (4 colors)
- **Background:** White `#FFFFFF`, grid lines `#E8E8E8`
- **Font:** Tableau Book, size 12 for body, size 18 for titles

---

## Part 5 — Quick Reference: Charts by Analysis Step

| Analysis Step | Chart Type | Sheet | Supports Hypothesis |
|---|---|---|---|
| EDA — Correlation | Scatter + trend line | 2A, 2B | H1: Obesity strongest predictor |
| EDA — Correlation matrix | Heatmap (color table) | 2C | H1 |
| EDA — Demographic | Bar chart by race | 3A | H4: Racial disparities |
| EDA — Demographic | Grouped bar by gender | 3B | H4 |
| EDA — Regional | Box plot by region | 3C | H2: South highest |
| Geospatial | Filled choropleth (diabetes) | 4A | H2: Diabetes Belt |
| Geospatial | Filled choropleth (obesity) | 4B | H1 geographic |
| Geospatial | Risk overlay dual-axis map | 4D | H1 + H2 combined |
| Regression | Scatter + OLS trend line | 5A, 5B | H1: Confirms coefficients |
| Regression | Residual plot (image) | 5C | Model validity |
| Regression | Results table | 5D | H1: p-values, R² |
| Clustering | Cluster scatter plot | 6A | H2: Cluster profiles |
| Clustering | Cluster geographic map | 6B | H2: Spatial coherence |
| Clustering | Cluster profile bars | 6C | H2: Cluster interpretation |
| Time-Series | Multi-line national trend | 7A | H3: Rising diabetes |
| Time-Series | State-level trend lines | 7B | H3: Which states improving? |
| Time-Series | YoY change bars | 7D | H3: Growth rate by year |
| Summary | State rankings table | 8A | All hypotheses |
| Summary | Hypothesis scorecard | 8B | All hypotheses |

---

*Guide prepared for CareerFoundry Data Immersion Achievement 6 — February 2026.*
*Data: CDC U.S. Chronic Disease Indicators, 2023 Release.*
