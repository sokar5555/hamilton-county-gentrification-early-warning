# Demographic Displacement & Gentrification Early-Warning System

## View full report at: https://sokar5555.github.io/hamilton-county-gentrification-early-warning

Neighborhood gentrification risk is estimated across all **226 populated Census tracts** in Hamilton County, Ohio. Tract-level probabilities are generated using a gradient-boosted decision tree model (`LGBMClassifier`) trained on baseline demographic, socioeconomic, and housing-supply indicators from the U.S. Census Bureau.

![Hamilton County Gentrification & Displacement Early-Warning Risk](figures/hamilton_county_risk_map.png)

## Methodology

* **Feature Set:** 18 tract-level predictors are engineered from ACS 5-Year Estimates, capturing baseline income, educational attainment, housing costs, renter concentration, housing density, and net housing-stock change.
* **Target Definition:** Tracts are labeled using a binary gentrification outcome (`51` observed positives; `22.6%` countywide base rate) reflecting sustained upward socioeconomic and housing-market shifts.
* **Validation:** Risk scores are generated strictly out-of-fold via stratified cross-validation so every tract is evaluated on a model that has not seen its label, limiting overfitting on a small county-level sample.

## Model Performance

| Metric | Value |
| :--- | :--- |
| **Tracts Scored** | `226` (`0` missing scores) |
| **Observed Gentrified Tracts** | `51` (`0.226` baseline rate) |
| **Out-of-Fold ROC-AUC** | `0.758` |
| **Out-of-Fold PR-AUC** | `0.481` (`2.13x` lift over baseline) |
| **Predicted Risk Range** | `0.039 – 0.896` |
| **Top-10 Hit Rate** | `5 / 10` (`Tracts 10, 9, 17, 42, 267`) |


## Key Insights

* **Core urban risk is driven by renter exposure and tight supply.** Over-the-Rhine holds the top three predicted risk scores (**Tracts 10, 9, and 17**), where heavy baseline renter shares (`78.1%` to `94.7%`) intersect with sharp educational shifts (`+21.7` to `+25.0` pp) and constrained or shrinking housing stock. **Tracts 42 and 267** account for the remaining confirmed hits in the top 10.
* **False positives are explained by distinct supply-side dynamics.** Pairing risk scores with net housing-unit changes surfaces two non-gentrifying patterns among high-scoring tracts:
  * *Student-housing expansion:* **Tract 22** (Clifton Heights / CUF) recorded the largest unit gain in the top 10 (`+149.1` units / `+10.8%`) alongside a `68.0%` renter share, reflecting purpose-built student housing rather than resident displacement.
  * *Housing-stock contraction:* **Tract 71** (`-244` units / `-11.8%`) and **Tract 254.02** (`-125` units / `-7.2%`) experienced steep unit losses, showing how baseline vulnerability can overlap with disinvestment or demolition rather than market heating.

![SHAP Feature Attribution: Hamilton County Displacement Risk](figures/shap_summary.png)

### Highlighted Top-10 Diagnostics

| Tract | Area | Risk | Renter Share | Unit Change | Signal | Target |
| :--- | :--- | :---: | :---: | :---: | :--- | :---: |
| **10** | Over-the-Rhine | `0.896` | `78.1%` | `+34` (`+2.8%`) | `+25.0` pp education jump | `1` |
| **9** | Over-the-Rhine | `0.859` | `78.1%` | `-76` (`-6.7%`) | Stock contraction in high-demand core | `1` |
| **17** | Over-the-Rhine | `0.831` | `94.7%` | `+96` (`+10.2%`) | `+21.7` pp education jump | `1` |
| **22** | Clifton Heights / CUF | `0.779` | `68.0%` | `+149.1` (`+10.8%`) | Student housing expansion | `0` |
| **71** | Hamilton County | `0.762` | — | `-244` (`-11.8%`) | Steep stock contraction | `0` |
| **254.02** | Hamilton County | `0.751` | — | `-125` (`-7.2%`) | Stock contraction | `0` |

*Tracts 42 and 267 also ranked in the top 10 (`target = 1`). The complete 10-tract table and interactive visuals are available in the knitted HTML report.*

## Limitations

* **Small sample sensitivity:** With 226 tracts and 51 positive cases, rankings at the top of the distribution remain sensitive to correlated urban-form features.
* **University confounding:** Dense, renter-heavy corridors near campuses (such as Tract 22) are frequently flagged even when turnover is driven by transient student populations.
* **Ambiguous unit loss:** Housing contraction cannot be separated by cause using Census stock counts alone; unit losses appear during both high-demand redevelopment (Tract 9, `-6.7%`) and disinvestment (Tracts 71 and 254.02).
* **Data lag and indirect measurement:** Multi-year ACS estimates lag current market conditions and carry tract-level margins of error. Aggregate demographic turnover is measured rather than direct eviction or involuntary mobility.
* **No spatial context:** Tracts are scored independently, omitting spillover pressure from adjacent corridors and block-level variation within tracts.

## Next Steps

* **Add campus controls:** Student population share, group-quarters share, and distance-to-campus indicators can be introduced to separate university growth from residential gentrification.
* **Integrate market and disinvestment data:** Hamilton County Auditor sales velocity, price-per-square-foot trends, building permits, demolitions, and vacancy rates should be added to distinguish redevelopment from abandonment.
* **Refine features and validation:** SHAP diagnostics, stronger regularization, spatial lag features, and spatially blocked cross-validation can be applied to improve top-decile precision.
* **Overlay displacement records:** Risk scores can be paired with eviction filings, tax delinquency, and expiring affordable-housing subsidies to support targeted anti-displacement work.

## Tech Stack

* **Data & Spatial Workflow (R):** R, R Markdown (`knitr`, `reticulate`), `tidycensus`, `sf`, `tidyverse`, `ggplot2`
* **Modeling & Validation (Python):** Python, `lightgbm` (`LGBMClassifier`), `scikit-learn`, `pandas`, `numpy`
* **Data Sources:** U.S. Census Bureau American Community Survey (ACS 5-Year Estimates) & TIGER/Line Census Tracts
* **Reporting:** Standalone knitted HTML report published via GitHub Pages

## Reproducibility

### Prerequisites
* **R (4.2+)** and **Python (3.9+)** configured for `reticulate`.
* A free **U.S. Census Bureau API key** (available at [api.census.gov](https://api.census.gov/data/key_signup.html)).

### Steps
1. **Clone the repository:**
   ```bash
   git clone https://github.com/<username>/<repo-name>.git
   cd <repo-name>
   ```
2. **Install R and Python dependencies:**
   ```r
   install.packages(c("rmarkdown", "knitr", "reticulate", "tidycensus", "sf", "tidyverse", "ggplot2"))
   ```
   ```bash
   pip install lightgbm scikit-learn pandas numpy
   ```
3. **Register the Census API key:**
   ```r
   tidycensus::census_api_key("YOUR_CENSUS_API_KEY", install = TRUE)
   ```
4. **Knit the report:**
   ```r
   rmarkdown::render("index.Rmd")
   ```
   *ACS data retrieval, feature engineering, out-of-fold LightGBM validation, and report figures are executed end-to-end when the document is knitted. If multiple Python environments are installed, the target interpreter can be set via `RETICULATE_PYTHON` prior to rendering.*
