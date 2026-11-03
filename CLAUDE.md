# PCE Forecast Project: Context for Claude Code

## Goal
Predict the **headline PCE inflation rate for November 2026**: the year-over-year % change in BEA's PCE price index (FRED `PCEPI`), i.e. `(PCEPI[2026-11] / PCEPI[2025-11] - 1) * 100`. Also report the November month-over-month % change. This is a final project for a graduate Statistical Models course (UChicago ADSP 31014). The team presents in Week 10 (early December 2026), before BEA releases November PCE in late December, so this is a true out-of-sample forecast.

## Hard rules (course constraints)
- **Only regression models from the course:** linear regression (general linear model), GLMs (exponential family, any link), binary logistic, Poisson and gamma regression.
- **Not allowed:** time series models (ARIMA, VAR, exponential smoothing, state space), neural networks, LLMs, or ML ensembles (random forest, boosting).
- **Gray area:** lags of PCE itself (or of the PCE components) as predictors are close to autoregression. Don't make them the core of the model. Keep any such variant separate and labeled until the instructor confirms. CPI, PPI and market series are separate series, so lagging them is fine.
- Use only trustworthy public data: FRED, BLS, BEA, Census, ADP, Indeed Hiring Lab, Revelio Labs, Freddie Mac PMMS. Manheim, Zillow, EIA and USDA are also fine.
- Course methods to show (by week): Newton-Raphson/MLE (1), SWEEP algorithm for aliased columns (1), coefficient interpretation and Shapley-value predictor ranking (2), R²/adjusted R², forward and backward selection, Cook's distance and influence (3), GLM deviance, Pearson and deviance residuals (4), logistic odds, separation and interactions (5–6), gamma GLM with log link (9).

## Key insight
YoY for Nov 2026 is the chain of 12 monthly changes, Dec 2025 to Nov 2026. Through Aug 2026 those are known (YoY = 3.42% in Aug). September PCE is due around Oct 30 and October around late November, so by the presentation probably **only November's monthly change is unknown**. The core task is to predict 1–3 monthly changes well and chain them onto the released months:
`YoY_Nov = (PCEPI_latest_known * Π(1 + m̂_t/100) / PCEPI_2025-11 - 1) * 100`.

## Approach: bottom-up by driver group
Model PCE as five driver groups. Headline monthly change ≈ Σ (previous month's nominal share × group monthly change).

| Group | ~Share | Includes |
|---|---|---|
| Energy | 4% | Gasoline & other energy goods, electricity, utility gas |
| Food at home | 7.5% | Food & beverages for off-premises consumption |
| Core goods | 22% | Durables, plus nondurables excluding food and energy (clothing, Rx drugs, supplies) |
| Housing | 15% | Rent, owners' equivalent rent |
| Services ex-housing & energy | 51% | Health care, financial services & insurance, food services & accommodations, recreation, transportation services, other services, NPISH |

- **Targets:** each group's monthly % change from BEA NIPA Table 2.4.4U (PCE price indexes by type of product). **Weights:** nominal spending from Table 2.4.5U.
- **Option A (main model):** one regression per group on its own proxies, then aggregate with the shares.
- **Option B (comparison):** one regression of headline monthly change on about 5–10 group proxies.
- **Model families:** OLS as the baseline. Gamma GLM with log link on the gross ratio `P_t / P_{t-1}` (always positive). Logistic as a supporting model, e.g. P(Nov monthly change > 0.2%) or P(YoY ≥ 3%). Skip Poisson because inflation is not a count, but explain why in the write-up.
- Effort should follow volatility, not share. Energy (~4%) drives most monthly swings in headline, while housing is smooth. Compute each group's share² × variance contribution first to confirm this.

## Proxy series by category
The FRED IDs below were written from memory, so **verify each one when downloading**. Use seasonally adjusted series.

| Category | Group | Main proxy | Faster signal for November |
|---|---|---|---|
| Durable goods (~10%) | Core goods | CPI new vehicles `CUSR0000SETA01`, used cars `CUSR0000SETA02`, household furnishings `CUSR0000SAH3` | Manheim used-vehicle index (leads CPI used cars by ~2 months), import prices, dollar index, tariffs |
| Food at home (~7.5%) | Food | CPI food at home `CUSR0000SAF11`, PPI consumer foods | USDA Food Price Outlook, FAO food price index |
| Gasoline & energy goods (~2.5%) | Energy | CPI gasoline `CUSR0000SETB01` | EIA weekly retail gasoline `GASREGW`, WTI `DCOILWTICO` (all November weeks known by early December) |
| Other nondurables (~11%) | Core goods | CPI apparel `CPIAPPSL`, Rx drugs `CUSR0000SEMF01` | Import prices of consumer goods |
| Housing (~15%) | Housing | CPI rent `CUSR0000SEHA`, OER `CUSR0000SEHC` | Zillow and Apartment List market rents (lead by 6–12 months) |
| Utilities (~2%) | Energy | CPI electricity `CUSR0000SEHF01`, utility gas `CUSR0000SEHF02` | Henry Hub natural gas `DHHNGSP` |
| Health care (~16–17%) | Services | PPI physician care and hospitals, CPI medical care services `CUSR0000SAM2` | Smooth. Medicare rate changes hit in January. |
| Financial services & insurance (~7–8%) | Services | PPI portfolio management, CPI motor vehicle insurance `CUSR0000SETE` | S&P 500 `SP500`, lagged one month |
| Food services & accommodations (~7%) | Services | CPI food away from home `CUSR0000SEFV`, lodging `CUSR0000SEHB` | Leisure & hospitality hourly earnings `CES7000000003` |
| Recreation services (~4%) | Services | CPI recreation services `CUSR0000SARS` | None |
| Transportation services (~3–4%) | Services | CPI airline fares `CUSR0000SETG01`, maintenance & repair `CUSR0000SETD`, PPI airline passenger services | Jet fuel prices |
| Other services (~8%) | Services | CPI tuition `CUSR0000SEEB`, phone services, personal care | Average hourly earnings, Indeed posted wages |
| NPISH (~3%) | Services | No direct price; BEA builds it mostly from wages | Employment Cost Index (or fold into the rest of Services) |

## Information timing (critical)
Train with the same information set we'll have when forecasting in early December 2026:
- **Available:** October CPI and PPI (mid-November), probably October PCE (late November), and November real-time data (weekly gasoline, oil, natural gas, S&P 500, Manheim, jobless claims, breakevens).
- **Not available:** November CPI and PPI (mid-December).
- The existing `features_full.csv` lags all monthly releases by 2 months. For this forecast, a 1-month lag of CPI and PPI is legitimately available, so rebuild the lags to match.

## Selection and validation
1. Shortlist predictors by economics (the table above). Within each group, use forward/backward selection with AIC, BIC or F-tests, and rank predictors with Shapley values.
2. Check collinearity with VIF, and flag aliased columns with SWEEP.
3. Find influential periods (COVID 2020–22, 2008 oil collapse, 2021 used-car spike) with Cook's distance. Compare three choices: keep them, add a dummy (`covid_2020_2022` already exists), or start the sample later.
4. **Backtest with an expanding window only:** fit through t−1, predict t, for every month from 2015 to 2026. Never use random CV. Score by MAE in percentage points against two benchmarks: last month's change and the 3-month average.
5. Backtest the full pipeline on past Novembers using only data that was available at the time.
6. BEA revises earlier months with every release. Train on the latest vintage, and compare the final forecast to the first-release figure.

## Existing files
- `features_full.csv`: 335 monthly rows (1999-01 to 2026-11), targets `y_pce_mom` and `y_pce_yoy`, ~70 lagged macro predictors (L1–L3), `month`, `covid_2020_2022`. Useful as a starting point, but its predictors are broad macro series rather than component proxies. Verify its sources and lag logic.
- `exploring_PCE.ipynb`: empty notebook.
- Course code to reuse for style and methods: `../SMDS Code/Regression.py` and the weekly example scripts.

## Deliverables
1. Data pipeline: download the component targets, weights and proxies, then build lags that match the information timing.
2. Models: Option A (per-group OLS, plus gamma GLM checks), Option B (single regression), and a logistic supporting model.
3. Backtest table comparing the models and benchmarks.
4. Final forecast: Nov 2026 monthly and YoY change, with a prediction interval and a breakdown by group contribution.
5. Presentation-ready charts and a short write-up explaining model choices in course terms.

## Open questions for the instructor
- Is the target YoY (assumed) or month-over-month?
- Are lags of PCE itself allowed as predictors?
- How will predictions be scored?
