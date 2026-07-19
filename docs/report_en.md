# Population Aging and Health Expenditure: An International Comparative Analysis

## About the Dataset

This project uses data from **World Bank Open Data**, one of the most authoritative sources of global development statistics. Unlike single-indicator projects, this analysis required combining **two separate indicators**:

1. **`Population ages 65 and above (% of total population)`** — official code `SP.POP.65UP.TO.ZS`.
2. **`Current health expenditure per capita (current US$)`** — official code `SH.XPD.CHEX.PC.CD`.

Both files were downloaded directly from the indicator pages on `data.worldbank.org` in the legacy Excel format (`.xls`). A notable technical detail: these files use the older binary Excel format (`CDFV2`), which requires the `xlrd` library to be read with pandas.

The raw data is in **wide** format: each row is a country, and each year (1960–2025) is a separate column. Alongside the main `Data` sheet, both files include two metadata sheets (`Metadata - Countries` and `Metadata - Indicators`) confirming the official code and description of each indicator.

## Project Objective

The goal was to answer the following question: **Is there a systematic relationship between the share of a country's elderly population and its health expenditure? And where does Japan — the world's most aged country — stand relative to that relationship?**

To answer this, both indicators were collected for 23 countries (including Japan, South Korea, Germany, and Italy per the project's explicit scope, plus a diverse sample from other continents) over 2000 to the most recent available year. A scatter plot with a trend line was then produced to reveal the overall relationship, followed by a quantitative assessment of Japan's specific position relative to that trend.

Below, each phase of the project is walked through together with the exact code used and the actual output obtained.

---

## Phase 1: Reshaping the Data from Wide to Long

The first technical challenge was that the raw data was in wide format — one column per year. For analysis and plotting, the **long** format (one row per country-year) is far more convenient. This transformation was done with `pd.melt`:

```python
def wide_to_long(df_wide, value_name):
    year_cols = [c for c in df_wide.columns if str(c).isdigit()]
    df_long = df_wide.melt(
        id_vars=['Country Name', 'Country Code'],
        value_vars=year_cols,
        var_name='Year',
        value_name=value_name
    )
    df_long['Year'] = df_long['Year'].astype(int)
    df_long = df_long[df_long['Country Code'].isin(countries)]
    return df_long
```

This function first identifies columns whose names are numeric (i.e., years), then uses `melt` to convert them from a horizontal layout (separate columns) into a vertical one (a single `Year` column and a single value column), and finally filters down to the 23 countries of interest.

The output of this phase showed that `pop65_wide` and `health_wide` each had shape `(265, 70)` — meaning the raw data covered all 265 countries/regions in the world across 70 columns (4 identifier columns + 66 years from 1960 to 2025). After filtering to 23 countries and converting to long format, both tables reached shape `(1518, 4)` — exactly 23 countries × 66 years. A final check confirmed that all 23 country codes, without exception, were present in both the population file and the health expenditure file.

---

## Phase 2: Final Cleaning and Merging the Two Indicators

Per the project scope, the analysis window was restricted to 2000 onward:

```python
YEAR_MIN = 2000
pop65_clean = pop65_long[pop65_long['Year'] >= YEAR_MIN].copy()
health_clean = health_long[health_long['Year'] >= YEAR_MIN].copy()
```

After this filter, both tables reached `(598, 4)` (23 countries × 26 years, 2000 to 2025).

Checking for missing values revealed something interesting: the `Pop65Plus` column had **no missing values at all**, while `HealthExpPerCapita` had **37 missing values**. The reason became clear after examining the "last year with data" for each country: population data (based on demographic estimates) was available for all countries through 2025, while health expenditure data (which requires actual financial reporting from each country) was only available through 2023 for most countries, and through 2024 for a subset (Canada, Germany, South Korea, Italy, the United Kingdom, France, Sweden, Portugal, Poland). This gap is expected: demographic statistics are estimated earlier and with greater confidence, whereas financial statistics require collection and official verification, which takes longer.

```python
merged = pop65_clean.merge(
    health_clean[['Country Code', 'Year', 'HealthExpPerCapita']],
    on=['Country Code', 'Year'], how='inner'
)
merged_final = merged.dropna(subset=['Pop65Plus', 'HealthExpPerCapita'])
```

This code merges the two tables on the combination of country and year, then drops any row missing a value in either indicator (since plotting a point on the scatter plot requires both values). The final result: **561 clean rows**, covering all 23 countries, spanning 2000 to 2024.

The final summary statistics showed a mean population aged 65+ of about 13% and a mean per-capita health expenditure of about $2,306, with a very wide range (from a minimum of $17 to a maximum of over $13,473) — reflecting the huge economic gap between countries in this sample.

---

## Phase 3: Scatter Plot and Trend Line

To answer the project's core question ("Is there a clear linear relationship?"), a simple linear regression was used:

```python
x = merged_final['Pop65Plus'].values
y = merged_final['HealthExpPerCapita'].values

slope, intercept = np.polyfit(x, y, 1)
correlation = np.corrcoef(x, y)[0, 1]
```

`np.polyfit(x, y, 1)` computes the best-fit straight line through all the points (slope and intercept), while `np.corrcoef` gives the Pearson correlation coefficient, which indicates how closely the relationship follows a straight line (ranging from -1 to 1; the closer to 1, the stronger the positive relationship).

The output showed a trend-line equation of `HealthExp = 240.32 × Pop65Plus − 812.51`, with a correlation coefficient of **0.650**. This indicates a positive and **fairly strong, though not perfect**, relationship between the two indicators — meaning that, on average, more aged countries do spend more per capita on health, but aging alone does not fully determine that spending (a perfect correlation would be close to 1).

To plot the chart, Japan was separated from the rest of the countries and highlighted with a distinct color and marker:

```python
plt.scatter(other_data['Pop65Plus'], other_data['HealthExpPerCapita'], alpha=0.35, color='gray', ...)
plt.plot(trend_x, trend_y, color='black', linestyle='--', ...)
plt.plot(japan_data['Pop65Plus'], japan_data['HealthExpPerCapita'], color='crimson', ...)
plt.scatter(last_japan['Pop65Plus'], last_japan['HealthExpPerCapita'], color='gold', marker='*', ...)
```

This code draws four layers on top of one another: light gray points for the other countries (to avoid clutter), a black dashed line for the trend, connected red points for Japan's historical trajectory (one point per year), and finally a gold star for Japan's most recent year to make it clearly identifiable.

The resulting chart revealed a striking pattern: as expected, Japan's points clustered on the right side of the chart (highest share of aging population), but contrary to the initial assumption, **Japan's trajectory sat below the trend line, not on it or above it**.

---

## Phase 4: Quantitative Analysis of Japan's Position

To turn this visual observation into a precise figure, the trend line's predicted value was computed for each year of Japan's data and compared against the actual value:

```python
japan_data['Predicted'] = slope * japan_data['Pop65Plus'] + intercept
japan_data['Residual'] = japan_data['HealthExpPerCapita'] - japan_data['Predicted']
japan_data['Residual_Pct'] = (japan_data['Residual'] / japan_data['Predicted']) * 100
```

`Predicted` is the value the trend line forecasts for that level of aging; `Residual` is the difference between the actual value and the prediction; and `Residual_Pct` converts that difference into a percentage to make comparison across years easier.

The full results table showed this pattern is not a one-off occurrence: in **22 of the 24 years** examined (2000–2023), Japan's `Residual_Pct` was **negative** — meaning actual expenditure was consistently below what the global trend line predicted. The only exceptions were 2011 and 2012 (with small positive deviations of 4.8% and 3.0%). From 2013 onward, this negative gap not only persisted but deepened, reaching its largest value in the most recent year (2023): Japan's actual expenditure was $3,638, while the trend line predicted $6,292 for its 29.6% aging level — that is, **42.2% below expectation**.

To assess whether this is unique to Japan or a broader group pattern, Japan's position was compared against two aged European countries (Germany and Italy) in the same year:

```python
peers = merged_final[
    (merged_final['Country Code'].isin(['DEU', 'ITA', 'JPN'])) &
    (merged_final['Year'] == last['Year'])
]
```

The result was highly revealing: **Germany**, with a lower aging level (22.8%), spent about **37.1% above** the trend line — the exact opposite of Japan. But **Italy**, like Japan, was **34.4% below** the trend line. This difference indicates the pattern is not simply about being "Japan-specific," but rather reflects a broader divide between health systems with stronger cost containment (Japan, Italy) versus higher-spending systems (Germany, and more broadly North America and Western Europe).

---

## Key Findings

Despite having the highest share of population aged 65+ among the 23 countries studied (29.6% in 2023), Japan's per-capita health expenditure is consistently **below** the global trend line — about 42% lower in the most recent year. This is not an isolated exception; the pattern held in 22 of the 24 years examined (2000–2023) and has deepened in recent years alongside intensifying aging. Peer comparison sharpens the picture: Germany, with a lower aging level (22.8%), spends about 37% **above** trend, while Italy, like Japan, is also below trend (-34.4%). This suggests that the relationship between aging and health spending is not a fixed natural law, but rather one in which national health-policy choices play a decisive role. In Japan's case, this pattern likely reflects a centrally managed public health system that has contained costs despite a heavy demographic burden — a finding that may be instructive for countries heading toward rapid aging, such as South Korea or, in the longer term, Iran.
