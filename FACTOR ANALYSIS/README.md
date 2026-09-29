# Factor Analysis Workflow (Python)

Reduces many correlated variables into a few hidden factors. Use the factors later in VAR, GARCH or regression. Works on any timeframe (1min to monthly) by changing one setting (`TF`).

## When to use
- You have many correlated variables (10+ macro series, 30+ stock returns)
- You want fewer inputs for VAR / GARCH / regression
- You want to find hidden drivers (market, value, momentum)

## When NOT to use
- Single series (ARMA, ARIMA, GARCH work directly)
- 3-5 variables (use VAR directly)
- Variables are not correlated (KMO will fail)

## PCA or Factor Analysis?
- **PCA:** forecasting and volatility pipelines (simple, orthogonal)
- **Factor Analysis:** when you want meaningful, nameable factors

## Where it fits
```
Many series -> Factor Analysis -> few factors -> VAR / GARCH
```
Or after ARMA/GARCH: run it on the residuals to find common shocks.

## Setup
```
pip install factor_analyzer seaborn pandas numpy matplotlib statsmodels
```

## Folder setup
Put all independent variables in **one folder**, one CSV per instrument. Paste the path in `FOLDER` (Cell 1).

```
my_factors/
  usdinr.csv
  gold.csv
  crude_oil.csv
  nifty.csv
  us10y_yield.csv
```

**Rules**
- First column = date/time, same format in all files
- Other columns = numeric only
- OHLC files: the code keeps only `close` (set by `KEEP_COL`)
- Same timezone in all files (no-timezone timestamps are treated as UTC)
- Same type of data in all files: don't mix daily files with intraday files
- Don't mix session-limited data (equities, indices) with 24h data (FX, crypto) at intraday TF
- Use prices (`PRICES = True`) or ready-made returns (`PRICES = False`)
- Yields/spreads: add the name to `DIFF_COLS` so they use difference, not log return
- No target/dependent variable, no other CSVs in the folder

## Settings (Cell 1)
| Setting | Meaning |
|---|---|
| `FOLDER` | folder path (use `r"..."` on Windows) |
| `TF` | `1min`, `5min`, `15min`, `1h`, `4h`, `1D`, `1W`, `ME` |
| `PRICES` | True = files are prices, False = already returns |
| `KEEP_COL` | column kept from multi-column files (default `close`) |
| `DIFF_COLS` | columns that use difference instead of log return |
| `MAX_MISSING` | max missing share per column (auto: 0.8 intraday, 0.2 daily+) |
| `DUP_CORR` | drop columns correlated above this (default 0.95) |

## Workflow (one cell each)
1. **Load folder:** merge on time, remove empty bins, fill short gaps, make returns, print missing %
2. **Clean and standardize:** clip outliers, z-score, drop constant and near-duplicate columns, check matrix health
3. **Correlation heatmap**
4. **Bartlett and KMO:** checks if data is suitable (Bartlett skipped when rows > 20,000)
5. **Number of factors:** Kaiser, scree, parallel analysis
6. **Run FA with rotation:** promax (correlated) or varimax (uncorrelated)
7. **Interpret:** loadings, communalities, variance explained
8. **Factor scores:** use in VAR/GARCH (with ADF check)
9. **Validate:** split-sample stability

## What the code does automatically
- Cleans column names, adds file name to avoid clashes
- Removes duplicate timestamps, sorts by time
- Removes time bins where most series have no price (nights, weekends, holidays)
- Fills price gaps of up to 2 bars
- Intraday only: removes the time-of-day volatility pattern
- Drops constant columns and near-duplicate columns (|r| > 0.95)
- Stops with a clear message if too few rows or columns remain

## How to interpret output

**Loader prints**
- Per-file shape and date range: check all files overlap. One short file cuts rows for everyone.
- Merged grid / After removing empty bins: big drop means files barely overlap in time
- Missing % table: a column near 100% is a problem file (wrong timezone, daily data, different dates). Remove it.

**Matrix health (Cell 2)**
- Rank = number of columns: good. Lower means duplicate/dependent columns.
- Condition number above 1e10: near-singular, remove similar series

**Bartlett**
- p < 0.05: good
- p > 0.05: stop, FA not useful
- With 20,000+ rows p is always about 0, so it is skipped. Use KMO.

**KMO**
- Above 0.6: acceptable, above 0.8: great
- Variable with KMO < 0.5: dropped automatically, then rechecked
- NaN: matrix is singular (duplicates or too few rows)

**Scree / parallel analysis**
- Number of factors = points above the random line (parallel analysis). More reliable than Kaiser.

**Loadings** (how strongly a variable belongs to a factor)
- Above 0.4: meaningful, above 0.7: strong
- Name each factor from its high-loading variables
- A variable loading high on two factors is messy. Try another rotation or fewer factors.

**Communality**
- Above 0.5: good
- Below 0.3: variable does not fit, consider dropping

**Variance explained**
- Cumulative 60%+ is a good target
- Intraday data usually gives lower values

**ADF p-value on factor scores**
- Below 0.05: stationary, safe for VAR/GARCH

**Split-sample congruence**
- Above 0.85: stable
- Below 0.85: factors change with time period, don't trust blindly

## Timeframe guide
| TF | Notes |
|---|---|
| 1min - 15min | Noisy, weaker correlations (Epps effect), factors weaker |
| 1h, 4h | Cleaner factors |
| 1D, 1W, ME | No time-of-day scaling. Weekly/monthly give few rows (need 100+) |

Run the same folder at 2-3 timeframes (e.g. 15min, 1h, 1D). Factors that stay similar across all are the reliable ones.

## Common problems
| Problem | Fix |
|---|---|
| 0 columns after loading | `MAX_MISSING` removed all. Check Missing % table, remove problem files, or use higher TF |
| KMO is NaN | Duplicate/dependent columns or too few rows. Check rank in Cell 2 |
| "Too few rows" | Files barely overlap. Check date ranges or use higher TF |
| KMO too low | Variables not correlated enough. Try another TF or different variables |
| No CSV found | Check `FOLDER` path |
| Dates not parsed | Date must be first column, format like YYYY-MM-DD HH:MM |
| Loadings change between periods | Unstable data. Use longer sample or fewer factors |
| Slow parallel analysis | Already sampled to 20k rows; reduce simulations if still slow |

## Pandas note
Use `"ME"` for month-end on new pandas (`"M"` on old). Use `"h"` and `"min"` on new pandas (`"H"` and `"T"` on old).
