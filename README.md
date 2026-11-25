# Demographic Displacement & Gentrification Early-Warning System

## View full report at: https://sokar5555.github.io/hamilton-county-gentrification-early-warning

An end-to-end spatial machine learning pipeline that predicts neighborhood-level gentrification and displacement risk across Hamilton County, Ohio using longitudinal U.S. Census (ACS) data, spatial spillover engineering, and interpretable gradient-boosted trees (`LightGBM` + `SHAP`).

## Executive Summary

Urban redevelopment rarely happens in isolation—it tends to radiate outward from appreciating neighborhoods into adjacent lower- and moderate-income corridors with available housing stock. This project models **2014 baseline tract conditions and spatial neighbor spillovers** to predict which Hamilton County census tracts experienced gentrification pressure between **2014 and 2022**, without leaking any future evaluation data into the feature set.

* **Out-of-Fold Validation:** Evaluated via 10x5 Repeated Stratified K-Fold cross-validation, achieving an out-of-fold **ROC-AUC of `0.747`** and **PR-AUC of `0.485`** (more than **2.1x** the `0.226` baseline positive rate).
* **Top-10 Precision:** **70% (7 of 10)** highest-ranked tracts experienced realized gentrification between 2014 and 2022, led by Over-the-Rhine (`Tracts 10, 9, and 17`), East Walnut Hills/Evanston (`Tract 42`), and Madisonville (`Tract 277`).
* **Key Insight:** Beyond baseline median income, **neighboring vacancy rates**, **local vacancy rates**, and **neighboring home values** serve as the strongest upward drivers of displacement risk.

## Visual Results

### 1. Hamilton County Displacement Risk Map

![Hamilton County Gentrification & Displacement Early-Warning Risk](figures/hamilton_county_risk_map.png)

Out-of-fold predicted risk probabilities range from `0.055` to `0.918`. Highest-quintile risk (`0.561–0.918`) clusters heavily across Cincinnati's urban core (Over-the-Rhine, Pendleton, West End, Walnut Hills) and northwest along the Mill Creek / Northside corridor, while outer suburban tracts fall predominantly into the lowest risk band (`0.055–0.107`).

### 2. Out-of-Fold SHAP Feature Attribution

![SHAP Feature Attribution: Hamilton County Displacement Risk](figures/shap_summary.png)

Aggregated out-of-fold SHAP values highlight how baseline affordability and spatial context interact:

* **Median Income (2014):** Acts as the dominant downward/eligibility driver, pushing affluent suburban tracts sharply negative.
* **Vacancy as a Redevelopment Catalyst:** High **Neighbor Vacancy Rate (2014)** and **Housing Vacancy Rate (2014)** strongly elevate predicted risk, signaling underutilized housing stock near appreciating areas.
* **Spatial Spillover Pressure:** Higher **Neighbor Avg Home Value (2014)** consistently raises predicted risk even when controlling for local tract characteristics.

## Top 10 Highest-Risk Census Tracts

| GEOID | Census Tract | Predicted Risk | Realized Target (2014–2022) | 2014 Median Income ($) | 2014 Median Rent ($) | Spatial Rent Gap ($) | $\Delta$ College Share |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `39061001000` | Tract 10 (OTR) | 0.918 | 1 | $49,462 | $819 | -$54 | +25.0% |
| `39061000900` | Tract 9 (OTR) | 0.865 | 1 | $26,319 | $730 | -$12 | +9.0% |
| `39061001700` | Tract 17 (OTR) | 0.840 | 1 | $16,279 | $721 | +$49 | +21.7% |
| `39061006800` | Tract 68 | 0.810 | 0 | $30,986 | $743 | +$26 | 0.0% |
| `39061007100` | Tract 71 (Northside) | 0.791 | 0 | $51,992 | $854 | +$11 | +2.0% |
| `39061004200` | Tract 42 (Walnut Hills) | 0.790 | 1 | $56,610 | $815 | +$59 | +7.4% |
| `39061026400` | Tract 264 | 0.747 | 1 | $17,321 | $767 | -$38 | +16.0% |
| `39061007000` | Tract 70 | 0.744 | 1 | $44,030 | $920 | -$111 | +23.0% |
| `39061027700` | Tract 277 (Madisonville) | 0.738 | 1 | $47,390 | $849 | -$131 | +11.4% |
| `39061002200` | Tract 22 (CUF/Clifton) | 0.736 | 0 | $34,536 | $860 | -$103 | +4.1% |

### Interpreting False Positives

The three top-10 tracts labeled `0` (`Tracts 68, 71, and 22`) illustrate practical domain nuances rather than random noise:

* **Watch-List Leading Indicators:** Tracts like Northside (`Tract 71`) share nearly identical baseline affordability and neighbor-spillover profiles with gentrified tracts, pointing to early-stage or delayed displacement pressure.
* **University & Student Churn:** University-adjacent areas such as CUF (`Tract 22`) exhibit low baseline incomes and modest rents driven by transient student populations, which can mimic displacement risk profiles.
* **Hard-Threshold Boundary Effects:** Because the target uses strict percentile cutoffs, tracts experiencing meaningful market pressure just shy of one threshold are coded as `0`.

## Methodology & Pipeline Architecture

1. **Longitudinal Census Harmonization (R — `tidycensus`, `tidyverse`, `sf`):**
   * Pulls 5-year ACS estimates for **2010–2014** (baseline) and **2018–2022** (evaluation).
   * Apportions 2010-vintage tracts into **2020 Census tract geometries** using Ohio's official Census areal relationship crosswalk.
   * Adjusts 2014 income, rent, and home values to constant 2022 dollars (`1.258` CPI-U deflator) and filters out unpopulated industrial/riverfront tracts ($\le 50$ residents).
2. **Leakage-Free Spatial Feature Engineering (R — `spdep`):**
   * Constructs a row-standardized Queen-contiguity spatial weights matrix (`poly2nb` / `nb2listw`).
   * Engineers neighbor-averaged 2014 lags (`lag_*_14`) and local spatial pressure gaps (`rent_gap_14`, `inc_gap_14`, `home_val_gap_14`) strictly from 2014 predictors.
   * Defines `gentrified_target` using 2014–2022 shifts among lower- and moderate-income tracts ($<75\text{th}$ percentile baseline income, top-$30\%$ educational attainment jump, and above-median rent or home-value growth).
3. **Modeling & Interpretability (Python via `reticulate` — `LightGBM`, `SHAP`, `scikit-learn`):**
   * Trains a regularized, small-sample-tuned `LightGBM` classifier (`max_depth=3`, `num_leaves=7`, `class_weight="balanced"`) using **10x5 Repeated Stratified K-Fold CV**.
   * Computes out-of-fold probability estimates and out-of-fold `TreeExplainer` SHAP attributions to avoid in-sample overfitting bias.
4. **Spatial Diagnostics (R — `tmap`):**
   * Maps out-of-fold probabilities onto 2020 tract polygons and exports static publication-ready PNGs to `figures/`.

## Limitations & Roadmap (Generalizing to Any U.S. County)

* **Limitations:** Single-county sample size (~220 populated tracts) introduces higher variance than metro-wide pooling; ACS 5-year estimates proxy neighborhood change rather than direct household out-migration and carry sampling margins of error; areal interpolation assumes uniform population density within split tracts.
* **Next Steps:**
  1. **County-Agnostic Parameterization:** Convert hardcoded state/county/FIPS values into R Markdown YAML parameters (`params$state`, `params$county`) to dynamically fetch the matching state crosswalk and re-knit for any U.S. county.
  2. **Multi-County Spatial Group CV:** Pool comparable metros (e.g., Columbus, Cleveland, Pittsburgh, Indianapolis) and validate with Spatial Group K-Fold CV to test out-of-region generalization.
  3. **High-Frequency Leading Indicators:** Add parcel assessments, building permits, eviction filings, and university/transit flags to separate student churn from residential displacement.
  4. **Forward Watch-Lists:** Score 2018–2022 baselines forward with the frozen model to generate live 2022–2030 watch-lists validated against incoming ACS releases.

## Tech Stack & Reproduction

**Stack:** R (`tidycensus`, `tidyverse`, `sf`, `spdep`, `tmap`, `reticulate`, `knitr`) • Python (`lightgbm`, `shap`, `scikit-learn`, `pandas`, `numpy`, `matplotlib`)

### How to Run

1. Request a free U.S. Census API key at [api.census.gov](https://api.census.gov/data/key_signup.html).
2. Store the key in your local `~/.Renviron` (never commit keys to version control):
   ```r
   tidycensus::census_api_key("YOUR_CENSUS_API_KEY", install = TRUE)
   ```
3. Restart R, open `hamilton-county-gentrification-early-warning_final.Rmd` in RStudio, and click **Knit**. Both `figures/shap_summary.png` and `figures/hamilton_county_risk_map.png` will regenerate automatically alongside the HTML report.
