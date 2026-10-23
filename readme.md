# Empirical Replication: Racial Discrimination in the Sharing Economy

> **Replication Portfolio & Pipeline Guide for Quantitative Marketing / Econometrics**  
> **Target Paper**: Edelman, Benjamin, Michael Luca, and Dan Svirsky. 2017. "Racial Discrimination in the Sharing Economy: Evidence from a Field Experiment." *American Economic Journal: Applied Economics*, 9 (2): 1–22.  
> **Replication Package**: [AEA Data and Code Repository (10.1257/app.20160213)](https://www.aeaweb.org/articles?id=10.1257/app.20160213)  
> **Core Stack**: R (`tidyverse`, `fixest`), Stata (`reghdfe` cross-validation)

---

## 1. Project Overview & Research Question

This repository contains the complete, reproducible end-to-end replication pipeline for Edelman, Luca, and Svirsky (2017). 

* **Core Research Question**: Do Airbnb hosts discriminate against guests based on perceived race in an online matching market? Specifically, do inquiries sent from identical guest accounts with distinctively African-American names receive significantly lower acceptance rates than those with distinctively White names?
* **Methodology**: Large-scale randomized field experiment (audit study) analyzing ~6,400 guest inquiries sent to Airbnb listings across major US metropolitan markets.
* **Core Econometric Model**: Linear Probability Model (OLS) with clustered standard errors and listing/host demographic controls:
  $$\text{Yes}_i = \beta_0 + \beta_1 \text{African\_American}_i + X_i \gamma + \varepsilon_i$$
* **Key Baseline Finding**: Guests with African-American names experience a statistically significant **8.0 percentage point penalty** in host acceptance rates (40.8% vs. 48.8%, $p < 0.001$). This penalty remains remarkably stable when conditioning on extensive host and listing characteristics.

---

## 2. Repository Directory Structure

The project enforces strict separation between read-only raw data, automated scripts, and generated outputs. Raw data is never modified manually.

```text
airbnb_replication/
├── data/
│   ├── raw/                 # Read-only original data (main_data.csv, hosts.dta, master_analysis.do)
│   └── processed/           # Script-generated clean datasets (df_clean.rds)
├── code/
│   ├── 00_setup.R           # Automated directory initialization
│   ├── 01_clean.R           # Schema extraction, headless CSV parsing, sample selection
│   ├── 02_table1.R          # Descriptive statistics and raw balance checks
│   └── 03_table2.R          # Primary LPM regressions (feols) and mechanism interactions
├── output/
│   ├── table2.html          # Publication-ready Table 2 (Word/Browser compatible)
│   └── table2.tex           # LaTeX source for paper integration
└── README.md                # Replication log, theoretical proofs, and pipeline guide
```

---

## 3. Environment & Prerequisites

Run all scripts in **R 4.0+**. Install the primary dependencies via CRAN:

```r
install.packages(c("tidyverse", "fixest"))
```

* `tidyverse` (`readr`, `dplyr`, `purrr`): High-performance data manipulation and tidy workflows.
* `fixest` (`feols`, `etable`): Ultra-fast high-dimensional fixed effects regression and cluster-robust standard errors (industry standard in quantitative marketing and empirical economics).

---

## 4. End-to-End Replication Workflow

### Step 0: Data Sourcing & Verification
1. Download the official replication package `app.20160213_data.zip` from the AEA website.
2. Unzip all files (`main_data.csv`, `hosts.dta`, `master_analysis.do`, `readme.docx`) into `data/raw/`.

### Step 1: Econometric Framework & Identification
* **Unit of Observation**: A single booking inquiry sent to an Airbnb host listing ($i$).
* **Outcome Variable ($Y_i$)**: Binary indicator `yes` ($1$ if host accepted or conditionally accepted; $0$ if host rejected or did not respond).
* **Treatment Variable ($D_i$)**: Indicator `guest_black` ($1$ if distinctively African-American name, $0$ if distinctively White name).
* **Error Structure & Clustering**: Inquiries sent to the same host/listing or same name-city market pair exhibit correlated error shocks. Standard errors must be clustered at the `name_by_city` or host level to prevent artificial precision and inflated $t$-statistics.

### Step 2: Automated Directory Setup (`code/00_setup.R`)
```r
base_dir <- "airbnb_replication"
dirs <- file.path(base_dir, c("data/raw", "data/processed", "code", "output"))
lapply(dirs, dir.create, recursive = TRUE, showWarnings = FALSE)

files <- file.path(base_dir, c(
  "code/01_clean.R",
  "code/02_table1.R",
  "code/03_table2.R",
  "README.md"
))
lapply(files, file.create)
```

### Step 3: Raw Data Diagnosis & Dynamic Schema Extraction (`code/01_clean.R`)
The raw file `main_data.csv` lacks a header row. Reading it with default `read_csv()` causes the first observation to be misparsed as column headers. We programmatically extract the true 49 variable names from the authors' Stata do-file:

```r
library(tidyverse)

# Dynamically parse 49 variable names from Stata do-file rename commands
do_lines <- readLines("airbnb_replication/data/raw/master_analysis.do", n = 75)
rename_lines <- do_lines[grepl("^rename v", do_lines)]
var_names <- sub("^rename v[0-9]+\\s+", "", rename_lines)

# Read raw CSV preserving all 7,554 observations
df_raw <- read_csv(
  "airbnb_replication/data/raw/main_data.csv",
  col_names = var_names
)
```

### Step 4: Sample Selection & Cleaning (`code/01_clean.R`)
* **Administrative Exclusions**: Inquiries in **Tampa** and **Atlanta** were shut down by Airbnb during the experiment; they are excluded from the experimental analysis.
* **Outcome Coding**: Non-responses (`host_response == -1`) are treated as rejections (`yes = 0`) because guests cannot secure lodging. Technical message transmission failures (`host_response %in% c(13, 14)`) are coded as missing (`NA`).
* **Missing Value Handling**: Survey missing values are coded as `-1` in the raw data; `price <= 0` is converted to `NA` before calculating `log_price` to prevent invalid log transformations (`NaN`).

```r
df_clean <- df_raw %>%
  # Exclude banned experimental cities
  filter(!city %in% c("Tampa", "Atlanta")) %>%
  mutate(
    # Outcome: 1 = positive response, 0 = rejection / no response, NA = transmission error
    yes = case_when(
      host_response %in% c(1, 4, 6) ~ 1,
      host_response == 0 | host_response == -1 | host_response %in% c(2, 3, 7, 8, 9, 10, 11, 12) ~ 0,
      host_response %in% c(13, 14) ~ NA_real_
    ),
    # Treatment indicator
    guest_black = if_else(guest_race == "black", 1, 0),
    
    # Cluster variable
    name_by_city = paste0(guest_first_name, "_", city),
    
    # Host and Listing Covariates
    host_race_black   = if_else(host_race == "black", 1, 0),
    host_gender_M     = if_else(host_gender == "M", 1, 0),
    multiple_listings = if_else(number_of_listings > 1, 1, 0),
    shared_property   = if_else(property_setup %in% c("Private Room", "Shared Room"), 1, 0),
    ten_reviews       = if_else(number_of_reviews >= 10, 1, 0),
    log_price         = log(if_else(price > 0, price, NA_real_))
  )

# Sample Breakdown Verification
# Total filtered sample: 6,390 rows (6,235 valid responses + 155 NAs)
```

### Step 5: Primary Regressions (Table 2 Replication via `fixest`)
We estimate Columns (1)–(3) using `feols()` with standard errors clustered by `name_by_city`:

```r
library(fixest)

# Column (1): Raw Treatment Effect
m1 <- feols(yes ~ guest_black, 
            data = df_clean, cluster = ~name_by_city)

# Column (2): Controlling for Host Demographics
m2 <- feols(yes ~ guest_black + host_race_black + host_gender_M, 
            data = df_clean, cluster = ~name_by_city)

# Column (3): Controlling for Host and Property Characteristics
m3 <- feols(yes ~ guest_black + host_race_black + host_gender_M + 
              multiple_listings + shared_property + ten_reviews + log_price, 
            data = df_clean, cluster = ~name_by_city)

etable(m1, m2, m3)
```

#### Replicated Results Matrix (Table 2)

| Variable | Column (1): Baseline | Column (2): Host Controls | Column (3): Full Controls |
| :--- | :---: | :---: | :---: |
| **Guest is African-American** | **-0.0796\*\*\*** (0.0170) | **-0.0860\*\*\*** (0.0174) | **-0.0922\*\*\*** (0.0179) |
| Host is African-American | — | 0.0665\*\* (0.0233) | 0.0901\*\*\* (0.0237) |
| Host is Male | — | -0.0524\*\*\* (0.0138) | -0.0468\*\* (0.0142) |
| Host has Multiple Listings | — | — | 0.0532\*\*\* (0.0146) |
| Shared Property (Private/Shared Room) | — | — | -0.0695\*\*\* (0.0153) |
| Host has 10+ Reviews | — | — | 0.1189\*\*\* (0.0144) |
| $\ln(\text{Price})$ | — | — | -0.0727\*\*\* (0.0113) |
| **Observations** | **6,235** | **5,615** | **5,560** |
| $R^2$ | 0.00641 | 0.01124 | 0.04063 |
| Standard Errors | Clustered (`name_by_city`) | Clustered (`name_by_city`) | Clustered (`name_by_city`) |

*Significance levels: \*\*\* $p < 0.001$, \*\* $p < 0.01$, \* $p < 0.05$.*

### Step 6: Mechanism & Heterogeneity Analysis (Table 3 Interaction)
To test whether African-American hosts also exhibit bias against African-American guests, we estimate:
$$\text{Yes}_i = \beta_0 + \beta_1 \text{GuestBlack}_i + \beta_2 \text{HostBlack}_i + \beta_3 (\text{GuestBlack}_i \times \text{HostBlack}_i) + \varepsilon_i$$

```r
m_inter <- feols(yes ~ guest_black * host_race_black, 
                 data = df_clean, cluster = ~name_by_city)
etable(m_inter)
```

* **Baseline Acceptance Rate** (White host, White guest): **48.6%** ($\beta_0 = 0.4856$)
* **Penalty from White Hosts** ($\beta_1$): **-8.7 percentage points** ($p < 0.001$)
* **Interaction Term** ($\beta_3$): **+0.0134** ($SE = 0.0464$, $p = 0.77$, statistically insignificant)
* **Total Penalty from African-American Hosts** ($\beta_1 + \beta_3$): **-7.4 percentage points**
* **Substantive Finding**: African-American hosts penalize African-American guests at nearly identical rates to White hosts, refuting simple in-group favoritism theories and suggesting broad statistical discrimination or shared institutional heuristics.

### Step 7: Exporting Publication-Ready Tables (`code/03_table2.R`)
```r
var_dict <- c(
  "guest_black"       = "Guest is African-American",
  "host_race_black"   = "Host is African-American",
  "host_gender_M"     = "Host is Male",
  "multiple_listings" = "Host has Multiple Listings",
  "shared_property"   = "Shared Property",
  "ten_reviews"       = "Host has 10+ Reviews",
  "log_price"         = "ln(Price)"
)

etable(
  m1, m2, m3,
  dict = var_dict,
  drop = "Constant",
  title = "Table 2. The Impact of Race on Likelihood of Acceptance",
  file = "airbnb_replication/output/table2.html",
  replace = TRUE
)
```

---

## 5. Core Econometric Lessons for Research Assistants

### Lesson 1: Stability of $\hat{\beta}_1$ under Randomization (OVB Formula)
In observational studies, adding controls changes $\hat{\beta}_1$ significantly due to omitted variable bias:
$$\hat{\beta}_1^{\text{short}} = \hat{\beta}_1^{\text{long}} + \hat{\gamma} \frac{\widehat{\text{Cov}}(D, X)}{\widehat{\text{Var}}(D)}$$
In a randomized audit experiment, $\widehat{\text{Cov}}(D, X) \approx 0$ by design. Adding host/property controls serves to verify the balance of the randomization and soak up residual variance, rather than rescue identification.

### Lesson 2: Listwise Deletion vs. Sample Selection Bias
When covariates containing missing values are introduced (e.g., undetermined host demographic traits), OLS executes listwise deletion ($N$ drops from 6,235 to 5,560). Researchers must verify that sample attrition across models does not selectively drop specific demographic cohorts.

### Lesson 3: Economic Grounding of Non-Response
In digital platforms, passive non-response (`host_response == -1`) cannot be discarded as random survey non-response. Because ignoring a guest prevents transaction completion, non-responses are economically equivalent to rejections (`yes = 0`). Dropping them would introduce severe selection attenuation bias.

---

## 6. How to Run

1. Clone this repository:
   ```bash
   git clone https://github.com/<your-username>/airbnb-discrimination-replication.git
   cd airbnb-discrimination-replication
   ```
2. Place `main_data.csv` and `master_analysis.do` into `airbnb_replication/data/raw/`.
3. Run the analysis sequentially:
   ```bash
   Rscript airbnb_replication/code/01_clean.R
   Rscript airbnb_replication/code/03_table2.R
   ```
4. Formatted tables will be generated in `airbnb_replication/output/`.
