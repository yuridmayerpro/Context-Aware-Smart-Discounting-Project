# Context-Aware Smart Discounting 📈

> **⚠️ Portfolio Disclaimer:** This repository contains a **sanitized and anonymized version** of a real-world project developed for the **largest B2B Pharmaceutical Distributor in Brazil**.
>
> **Business Context & Results:** This project delivered a **Strategic Pricing Engine** that empowers commercial teams to define optimal discount ranges adapted to specific business contexts.
>
> **Key Outcome:** The project **increased profit margins by up to 60 p.p. and reduced ineffective markdowns by designing a GLM engine that generates optimal discount ranges based on context variables (e.g., seasonality, region/UF).**

## 🎯 Project Overview

The **Smart Discount** project aims to develop a data science solution to find the optimal discount for Brand SKUs, optimizing promotional strategy actions. Currently, the company runs many campaigns with high discounts that do not necessarily generate value or demand increase. The solution seeks to understand product elasticity and identify the most effective discounts, ensuring better resource allocation.

The final deliverable is a data model that feeds an **Interactive Power BI Dashboard**. This tool allows business areas to receive recommendations for three discount ranges (minimum, ideal, and maximum) for an SKU or group of products, based on dynamic filters such as State (UF), Region, Channel, Brand, among others.

## 🧠 Theoretical Foundation & Methodology

The project's methodology is built upon microeconomics and econometrics foundations, ensuring that discount recommendations are accurate and aligned with business objectives.

### Elasticity Model

To estimate demand sensitivity to price variations, a robust statistical model is maintained in the processing stage (Python). The **coefficient estimation** is performed via a regression based on GLM (Generalized Linear Model).

The structure of the statistical model is:

$$
\log(E[Q]) = \beta_{0} \log(P) + \sum \beta_{k} X_{k} + \epsilon
$$

Where:

* **Q:** Total invoiced quantity (`BILLED_QTY`).

* **P:** Centered net unit price (`ln_price_c`).

* $X_k$**:** Vector of control variables and interactions (e.g., `TIER_BRAND`, `REGION_STATE`, `SUB_CHANNEL`).

* $\beta_0$**:** Base coefficient representing price elasticity.

To ensure estimation robustness, the implemented model is a **Negative Binomial Regression**, chosen for its suitability for count data with overdispersion (Variance > Mean).

### Discount Range Calculation (Power BI)

Although the statistical model is log-linear, **linear approximations** were chosen for calculating suggestion ranges in the dashboard (Scenarios). This decision aims to ensure result sobriety and avoid aggressive distortions in high discount ranges that exponential models might generate.

The formulas below reflect the DAX implementation:

#### a) Ideal Discount ($\delta^*$) - Gross Margin Maximization

Focuses on maximizing Gross Margin (LB). It utilizes a linearization of the profit curve to find a safe equilibrium point between current margin and elasticity.

$$
\delta^{*} = \frac{m - \frac{1}{|E|}}{2}
$$

Where:

* **m:** Standardized historical gross margin (`theoretical_unit_margin`).

* **|E|:** Absolute value of historical price elasticity (`final_elasticity`).

#### b) Uplift Discount ($\delta_{lift}$)

Calculates the discount necessary to achieve a volume growth target (*uplift*) defined by the user, assuming a linear demand response.

$$
\delta_{lift} = \frac{1 - V_{target}}{E}
$$

Where the Target Volume Index ($V_{target}$) is calculated as:

$$
V_{target} = V_{current} \times (1 + Lift\%)
$$

Where:

* $V_{current}$**:** Current indexed volume, calculated linearly as `1 + (E * -D_current)`.

* **Lift\_%:** Volume increment target inserted in the simulator.

* **E:** Price elasticity (negative value).

#### c) Maximum Discount ($\delta_{max}$)

Establishes the discount ceiling permitted to ensure operations respect the minimum margin defined by the business. The calculation reconstructs the price top-down.

$$
\delta_{max} = 1 - \frac{NP_{target}}{LP_{base}}
$$

Where $NP_{target}$ (Target Net Price) is derived from:

$$
NP_{target} = (m_{min} \times P_{dist}) + COGS
$$

Where:

* $m_{min}$**:** Target minimum margin (User input).

* $P_{dist}$**:** Distributor base price (List Price discounted by constant Distributor Discount).

* **COGS:** Cost of Goods Sold (`total_unit_cogs`).

### Hierarchical Fallback Mechanism

To ensure the dashboard always presents valid discount suggestions, even when the user applies very specific filters where sales history might be scarce, a **Hierarchical Fallback** algorithm was implemented in DAX.

This mechanism ensures critical parameters — Historical Margin, Elasticity, and Average Volume — are calculated with minimum statistical significance (configured for `_MinRows = 50` records).

#### Filter Relaxation Logic

The algorithm checks if the current context has sufficient data. If not, it progressively removes filters following a logical generalization order (from specific to broad), as per the structure below:

1. **Original Context:** Attempts calculation with all filters active.

2. **Product Hierarchy:** Removes *Product ID* → *Brand* → *Business Unit*.

3. **Channel Hierarchy:** Removes *Sales Team* → *Sub Channel*.

4. **Geographic Hierarchy:** Removes *State* → *Region*.

5. **Seasonal Hierarchy:** Removes *Week* → *Month*.

If at any stage the number of observations is sufficient ($\ge 50$), the calculation is performed with that data scope and the process stops. This avoids "division by zero" errors and ensures the suggested discount is always based on the closest possible historical reference to the user's selection.

# Smart Discount Pipeline

## 📂 Project Structure

This repository contains **8 sequential notebooks** representing the end-to-end Data Science lifecycle:

| File | Stage | Description | 
 | ----- | ----- | ----- | 
| `1_data_ingestion.ipynb` | **Ingestion** | Consolidates Sales and Financial data. Computes the **Unitary Cost (COGS)** from P&L statements (`REAL_UNIT_VALUE`). | 
| `2_data_processing.ipynb` | **Engineering** | Feature engineering pipeline. Handles outlier detection (Z-Score), creates lag features, and defines the target variable `BILLED_QTY` for the model. | 
| `3_eda.ipynb` | **Analysis** | Exploratory Data Analysis. Investigates demand curves, seasonality patterns, and price distributions across regions and channels. | 
| `4_feature_selection.ipynb` | **Selection** | Uses **FeatureWiz** (MRMR algorithm + Recursive XGBoost) to select the most predictive categorical features (e.g., `REGION_STATE`, `BRAND`), reducing dimensionality. | 
| `5_model_optimization.ipynb` | **Tuning** | Hyperparameter tuning loop for the GLM regularization (`alpha` and `L1_wt`) using **Optuna** to minimize AIC. | 
| `6_elasticity_model.ipynb` | **Modeling** | **Core Engine.** Trains the **Negative Binomial Regression (GLM)** to estimate the price elasticity coefficients for each context. | 
| `7_elasticity_analysis.ipynb` | **Evaluation** | Post-processing. Applies **Bühlmann-Straub Shrinkage** to adjust local elasticities towards the global mean (`final_elasticity`), ensuring statistical robustness. | 
| `8_discount_estimation.ipynb` | **Serving** | Calculates the final discount scenarios (Optimal, Min, Max) based on `final_elasticity` and exports to Power BI. | 

## ⚙️ Orchestration

* **Environment:** Azure Synapse Analytics / Databricks.

* **Frequency:** Monthly Execution.

* **Trigger:** Scheduled pipeline to update base margins and re-calculate elasticities based on the latest sales data.

## 🚀 Pipeline Flow (Detailed)

### 1. Cost Ingestion & Margins

* **Notebook:** `1_data_ingestion.ipynb`

* **Core Logic:**

  * Consumes P&L data (`FACT_FINANCIAL`) and Material mapping to build the Unitary Cost (COGS) foundation.

  * Calculates `REAL_UNIT_VALUE` by dividing total cost `ACTUAL_VALUE` by invoiced quantity.

  * **Imputation:** Handles nulls or inconsistencies using `historical_mean` by EAN and accounting category.

### 2. Data Engineering & Target Definition

* **Notebook:** `2_data_processing.ipynb`

* **Core Logic:**

  * Unifies Sales, S&OP, and Cost data into a single analytical view.

  * **Outlier Removal:** Applies Z-Score filters on `BILLED_QTY`, `LIST_PRICE_UNIT`, and `REAL_UNIT_MARGIN`.

  * **Target Variable:** Defines `BILLED_QTY` (Invoiced Quantity) as the target variable for the model.

  * **Price Centering:** Creates `NET_PRICE_ROUND` and calculates `PRICE_GAP_VS_MARKET`.

### 3. Feature Selection

* **Notebook:** `4_feature_selection.ipynb`

* **Core Logic:**

  * **Dimensionality Reduction:** Groups rare categories into 'Others' based on volume contribution (Pareto).

  * **Selection:** Uses **FeatureWiz** library (MRMR algorithm) to define a "Whitelist" of relevant categorical features (e.g., `BRAND_proc`, `REGION_STATE_proc`).

### 4. Elasticity Modeling

* **Notebook:** `6_elasticity_model.ipynb`

* **Core Logic:**

  * **Clustering:** K-Means groups brands into Price Tiers (`TIER_BRAND`) to avoid Simpson's Paradox.

  * **GLM (Generalized Linear Model):** Uses **Negative Binomial** family with Log link (suitable for overdispersed data).

  * **Formula:** `Q ~ ln_price_c + Categoricals + ln_price_c:Categoricals`.

  * **Evaluation:** Logs metrics (RMSE, MAE, Pseudo-R2) to MLflow.

### 5. Shrinkage & Robustness

* **Notebook:** `6_elasticity_model.ipynb` (Post-processing step) & `7_elasticity_analysis.ipynb`

* **Core Logic:**

  * Calculates `elasticity_local` (base coefficient + interactions).

  * Applies **Bühlmann-Straub Shrinkage**: Weights observed local elasticity with the global group mean (`elasticity_group_mean`). This corrects estimates for contexts with few observations or extreme variance.

  * **Final Output:** `final_elasticity` (or `elasticity_final`), capped to avoid positive values (Giffen goods anomalies).

### 6. Serving Layer

* **Notebook:** `8_discount_estimation.ipynb`

* **Core Logic:**

  * Prepares the final table for Power BI consumption.

  * Simulates Discount Scenarios:

    * `delta_star` (Ideal)

    * `delta_min` (Min Uplift)

    * `delta_max` (Max Margin Safety)

  * Integrates with `theoretical_unit_margin` to calculate profit impacts.

## 🛠 Maintenance Guide & Tips

1. **Model Monitoring (MLflow):**
   The model saves full execution logs, parameters, and artifacts to MLflow every run. Check the "Experiments" tab for diagnostic plots (Residuals, Variance Check).

2. **Hyperparameter Tuning:**

   * **Notebook:** `5_model_optimization.ipynb`

   * Regularization (ElasticNet) uses `alpha` and `L1_wt`. If the model is excessively "zeroing" important coefficients, adjust these values in the tuning loop.

3. **Performance:**

   * **Notebook:** `2_data_processing.ipynb`

   * The lag feature calculation step can be computationally expensive. Adjusting the lookahead window parameters can improve performance at the cost of capturing long-term effects.

4. **Elasticity Interpretation:**

   * **Notebook:** `7_elasticity_analysis.ipynb`

   * The final column `final_elasticity` undergoes *capping* (limiting extreme values) and positive value treatment (inverting illogical positive demand curves) before serving.