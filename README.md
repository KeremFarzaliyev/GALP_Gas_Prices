# GALP Gasoline Price Analysis & Forecasting

This project analyzes daily GALP gasoline prices in Portugal (2007–2023) alongside major competitors (SHELL, REPSOL, BP, CEPSA, AVIA, Q8) and Brent crude oil. We explore competitive dynamics, seasonality, and employ four forecasting methods—ARIMAX, Monte Carlo simulation, XGBoost, and a few‐shot LLaMA-2 model—then present a simple ensemble that guards against large LLM errors.

---

## 1. Data Overview

- **Sources & Scope**  
  - Daily spot prices (€/L) for GALP, SHELL, REPSOL, BP, CEPSA, AVIA, Q8 (2007 – 2023).  
  - Daily Brent crude prices (USD/barrel) mapped to local dates.  
  - We focus on next‐day (“t+1”) GALP price forecasting using the most recent history and same‐day competitor/crude signals.

- **Key Observations**  
  1. **Correlations with Competitors**  
     - Over rolling 60‐day windows, GALP exhibits the strongest contemporaneous correlation with SHELL (≈ 0.47), REPSOL (≈ 0.38), and BP (≈ 0.34).  
     - CEPSA and AVIA correlations are moderate (≈ 0.38 and 0.28), while Q8 shows lower consistency (≈ 0.27).  
  2. **Seasonality & Trends**  
     - All brands (including GALP) show higher prices in late summer (driving season) and winter (heating demand), with troughs in late spring.  
     - Brent crude remains a leading driver: rising Brent often precedes a broad uptick in station prices by a few days.

---

## 2. Forecasting Methods & Performance

### 2.1 ARIMAX

- **Methodology**  
  - Fit an ARIMAX model on GALP returns with exogenous regressors: same‐day prices of SHELL, REPSOL, BP, CEPSA, AVIA, Q8, and Brent.  
  - Select AR/MA orders via AIC/BIC and verify residual whiteness.  
- **Outcomes (Test‐Set)**  
  - **MAE** = € 0.0535  
  - **RMSE** = € 0.0677  

### 2.2 Monte Carlo Simulation

- **Methodology**  
  1. Compute daily GALP returns over rolling 30-day windows.  
  2. For each day in the last 24 months, sample 1,000 next‐day returns from the empirical distribution.  
  3. Convert each simulated return back to price, then derive (a) the mean forecast price and (b) a 95 % confidence interval.  
- **Outcomes (Last 24 Months)**  
  - **Average Absolute Error (MAE)** = € 0.0041  
  - **95 % CI Coverage** = 91.85 % (actual prices fell inside the simulated CI 91.85 % of days)  

### 2.3 XGBoost

- **Methodology**  
  - Build a gradient‐boosted tree regressor using features such as:
    - Lagged GALP prices (up to t – 7)  
    - Same‐day competitor & Brent prices  
    - Rolling‐window (30/60-day) summary features (e.g., trend slopes, correlations).  
  - Tune hyperparameters via grid search and 5‐fold cross‐validation on the training set.  
- **Outcomes (Test‐Set)**  
  - **MAE** = € 0.0384  
  - **RMSE** = € 0.0524  

### 2.4 Few‐Shot LLaMA-2

- **Methodology**  
  - Craft a few‐shot “chat‐style” prompt showing 3 historical examples of (Date, GALP_yesterday, SHELL_yesterday, Brent_yesterday → Next‐day GALP).  
  - For each test date, supply that prompt to LLaMA-2 (via Hugging Face pipeline) instructing it to output exactly one number (no currency symbol).  
  - Parse the generated text to extract the float (NaN if parsing fails).  
- **Outcomes (Test‐Set)**  
  - **Valid Outputs**: ~316 of 391 test days (others returned NaN).  
  - **MAE** = € 0.4753  
  - **RMSE** = € 0.6864  

---

## 3. Key Insights & Takeaways

1. **Competitive Dynamics**  
   - GALP consistently underprices REPSOL and BP by ~€ 0.02–0.05, likely a strategic move to maintain market share.  
   - SHELL’s pricing tracks GALP most closely (correlation ≈ 0.47), suggesting similar supply‐chain exposure.  
   - AVIA and Q8 exhibit more volatile gaps, indicating aggressive promotional pricing during high‐demand windows.

2. **Seasonality**  
   - All station prices rise from June through August (driving season), dip briefly in early autumn, and climb again as winter approaches (heating demands).  
   - Brent oil spikes (e.g., mid‐2022) correlate with a ~2–3 day lag before station prices reflect the change.

3. **Model Comparisons**  
   - **Monte Carlo** delivers the tightest mean forecasts (MAE = € 0.0041) with ~92 % coverage inside its 95 % intervals—ideal for risk‐sensitive applications (e.g., hedging).  
   - **XGBoost** strikes a balance: MAE = € 0.0384, capturing nonlinearities and competitor signals.  
   - **ARIMAX** (MAE = € 0.0535) is more transparent but slightly less accurate.  
   - **LLaMA-2** demonstrates the potential of few‐shot prompting but suffers from “hallucinations” (MAE = € 0.4753), making it unreliable as a standalone forecaster.

4. **Ensemble Overview**  
   - While a full ensemble strategy was explored (e.g., 0.10 € safeguard on XGBoost vs. LLaMA-2 differences), this README focuses on individual model results.  
   - The final ensemble combined XGBoost + LLaMA-2 forecasts when they differed by < € 0.10, otherwise defaulting to XGBoost.  

5. **LLM “Chat‐Style” Capabilities**  
   - The same LLaMA-2 pipeline—fed with an appropriate system message and examples—can also function as a conversational assistant:  
     - Answer energy‐market questions, summarize price anomalies, and generate analytical bullet points.  
   - This dual utility underscores that, beyond numerical forecasting, LLMs can provide qualitative insights and code‐generation support.

---

## 4. Key Metrics Snapshot

| Method                   | MAE (€/L) | RMSE (€/L) | Additional Notes                              |
|--------------------------|-----------|------------|-----------------------------------------------|
| **ARIMAX**               | 0.0535    | 0.0677     | Exogenous regressors: SHELL → Q8 & Brent      |
| **Monte Carlo** (24 mo)  | 0.0041    | —          | 95 % CI coverage: 91.85 %                     |
| **XGBoost**              | 0.0384    | 0.0524     | Nonlinear features: lags, competitor/crude    |
| **Few‐Shot LLaMA-2**     | 0.4753    | 0.6864     | Only ~316/391 days returned valid float       |

---

## 5. Conclusion

This project delivers a comprehensive view of GALP gasoline pricing:  
- **Descriptive Analysis** reveals how GALP positions relative to major competitors and responds to Brent oil cycles.  
- **ARIMAX** and **XGBoost** offer reliable quantitative forecasts, with XGBoost achieving sub‐€0.04 daily error.  
- **Monte Carlo Simulation** demonstrates exceptional probabilistic accuracy (MAE ≈ € 0.0041, ~92 % coverage), ideal for risk management.  
- **Few‐Shot LLaMA-2** highlights LLM potential for price forecasting and qualitative analysis, albeit with notable error spikes—motivating a guarded ensemble approach.

Overall, the combined toolkit showcases both classical econometric/time‐series methods and modern machine‐learning/LLM techniques—providing a robust, full‐stack solution for energy‐market forecasting and analysis.
