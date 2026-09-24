# Italian Road Accidents 2001–2024: Municipal Risk Analysis

This is a brief analysis completed as the final assignment for the Boolean Master's in Data Analytics program. It is an analysis of road accidents across all Italian municipalities (2001–2024), combining **ISTAT** accident data with **SITUAS** territorial data. The project studies how accidents have evolved over time and builds a **composite risk score** to identify the municipalities where road-safety investment would matter most.

---

## Table of Contents

- [Business Question](#business-question)
- [Key Findings](#key-findings)
- [Data](#data)
- [Repository Structure](#repository-structure)
- [Methodology](#methodology)
- [Results](#results)
- [Limitations](#limitations)
- [How to Run](#how-to-run)

---

## Business Question

The scenario: a company working in **road traffic management and risk prevention** wants an updated overview of the best Italian municipalities in which to invest. That means finding where accidents are most common and how serious they are.

The whole analysis is structured around my two ideas:

- **Exposure (frequency):** how often do accidents occur?
- **Payout risk (severity):** how serious or fatal are they when they do occur?

Because a city can be high on one and low on the other, the project ranks municipalities under three different priorities: frequency-focused, severity-focused and balanced.

## Key Findings

1. **Accidents per capita and per km² have fallen over 24 years.** The trend holds for Rome, for the other large cities (Milan, Turin, Naples, Florence) and for the macro-regions. Injury and fatality rates do not show the same clean decline, especially in small regions such as Umbria and Valle d'Aosta.
2. **2020 is an outlier.** Every city and region shows a sharp drop caused by COVID-19 lockdowns, followed by a partial rebound. For this reason, the risk ranking uses only **2021–2024**.
3. **Raw per-capita rankings are misleading.** Small towns dominate them (e.g. Belforte Monferrato: 231.9 accidents per 10,000 residents per year vs. 60.4 for Rome) because a handful of events divides by a tiny population. Rome, unsurprisingly, has the most accidents in absolute terms.
4. **Large cities are high-volume but low-consequence.** 67.6% of large cities (DEGURBA = 1) fall in the "high frequency / low severity" quadrant, and none of them appears in any top-20 risk list.
5. **Rural municipalities skew toward severity.** 45.2% of rural municipalities (DEGURBA = 3) fall in "low frequency / high severity", consistent with a higher-speed, worse-outcome hypothesis. They also have the highest share of "high/high" cases (20.3%).
6. **The priority list depends on the policy goal.** Only 9/20 cities are shared between the equal-weight and frequency-heavy top-20 lists, and 14/20 between equal-weight and severity-heavy. **7 municipalities appear in all three lists** and are the most defensible candidates regardless of weighting: Paruzzaro, Altare, Origgio, Noventa di Piave, Campogalliano, San Cesareo and Surano.

## Data

| Source | Content |
|---|---|
| **ISTAT** | Road accidents per municipality and year: collisions, injuries, fatalities |
| **SITUAS** | Municipality metadata: residents, surface area (km²), region, province, DEGURBA 2021 classification |

The cleaned and joined dataset has **572,700 rows** in long format (one row per municipality × year × accident category) and covers roughly **7,900 municipalities** between 2001 and 2024.

**DEGURBA** (Degree of Urbanisation) classifies municipalities as `1` = cities / densely populated, `2` = towns and suburbs, `3` = rural areas. Two municipalities have no official level (`-1`) and are excluded from interpretation.

Data cleaning, fixing and joining of the two sources are documented in `data_preparation.ipynb`. This analysis starts from its output.

## Repository Structure

```
.
├── data_preparation.ipynb    # cleaning and joining of ISTAT + SITUAS data
├── analysis.ipynb            # metrics, EDA, risk score, final rankings
└── data/
    ├── istat_situas_traffic_accidents_data.csv                          # cleaned, joined input (long format)
    ├── istat_situa_traffic_accidents_2001_2024_metrics.csv              # yearly metrics, all municipalities
    ├── istat_situa_traffic_accidents_2021_2024_metrics_riskscore.csv    # 2021–2024 metrics + risk scores
    └── results/
        ├── top_20_frequency.csv
        ├── top_20_severity.csv
        └── top_20_equal.csv
```

## Methodology

### 1. Reshaping

The raw data has three rows per municipality-year (collisions, injuries, fatalities). The table is pivoted into **one row per municipality and year** with three columns: `collisions`, `injuries`, `fatalities`.

### 2. Metrics

| Metric | Definition | What it captures |
|---|---|---|
| Injury rate | injuries / collisions | Liability payout risk |
| Fatality rate | fatalities / collisions | Severe liability payout risk |
| Per-capita accident rate | collisions per 10,000 residents per year | Individual exposure |
| Collision density | collisions / km² | Frequency of urban accidents |
| Resident density | residents / km² | Context on urbanisation and population change |

Division by zero is handled explicitly. For multi-year windows, residents and area are averaged across years (municipal boundaries rarely change), and the per-capita rate is divided by the number of years recorded to obtain an annual, person-time rate.

### 3. Exploratory analysis

- Line plots of each metric over time for Rome vs. Belforte Monferrato, for the five largest cities, and for every region (grouped into North-West, North-East, Center, South and Islands).
- Municipality-level distributions, with attention to zero-inflation and right skew.

### 4. Handling small-sample instability

Two decisions keep the ranking from being dominated by noise:

- **Time window:** only 2021–2024, to exclude the 2020 lockdown distortion and reflect the post-pandemic situation.
- **Minimum exposure threshold:** only municipalities with **more than 30 collisions** in the window are scored (2,519 of 7,910 municipalities). With a tiny denominator, a single fatality can swing a rate from 0% to 50%; with 30+ collisions, one event moves it by about 3 percentage points. The threshold also removed a spurious correlation (0.63) between per-capita rate and injury rate that came from municipalities with zero accidents.

### 5. Normalisation

Two options were compared on the filtered data:

- `RobustScaler` (median/IQR on `log1p`-transformed metrics)
- **Percentile-rank normalisation** on the raw rates (selected)

Percentile rank was chosen because it gives all four metrics the same scale and spread by construction (mean ≈ 0.50, std ≈ 0.288). This matters for fixed-weight scores, since otherwise the widest-spread metric silently dominates. `RobustScaler` left `fatality_rate` with roughly double the spread of the others, and is vulnerable to IQR collapse with zero-inflated data. The trade-off is that percentile ranks discard magnitude information and must be recomputed against a reference distribution when new data arrives.

### 6. Risk score

Frequency and severity metrics are combined into two subscores, since the two dimensions are essentially uncorrelated once small municipalities are filtered out.

```
severity_subscore  = 0.6 × fatality_rate_pct + 0.4 × injury_rate_pct
frequency_subscore = mean(per_capita_accidents_rate_pct, collision_density_pct)

RiskScore = w_freq × frequency_subscore + w_sev × severity_subscore
```

Three weighting scenarios test how sensitive the ranking is to policy priorities:

| Scenario | w_freq | w_sev |
|---|---|---|
| Equal | 0.5 | 0.5 |
| Frequency-heavy | 0.7 | 0.3 |
| Severity-heavy | 0.3 | 0.7 |

### 7. Quadrant analysis

Municipalities are split at the median on both subscores into four quadrants (high/low frequency × high/low severity) and compared across DEGURBA levels.

## Results

### Quadrant distribution by degree of urbanisation

| DEGURBA | High freq / low sev | Low freq / high sev | High freq / high sev | Low / low |
|---|---|---|---|---|
| 1 (cities) | **67.6%** | 12.4% | 10.5% | 9.5% |
| 2 (towns/suburbs) | 32.0% | 29.2% | 17.4% | 21.3% |
| 3 (rural) | 11.6% | **45.2%** | 20.3% | 22.9% |

DEGURBA 2 is the most heterogeneous group: no dominant profile, so urbanisation alone explains little there.

### Top 10 municipalities (equal weighting, 2021–2024)

| # | Municipality | Prov. | Region | Residents (avg) | DEGURBA | Collisions | Fatalities | Injuries | Score |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Origgio | VA | Lombardia | 7,996 | 2 | 108 | 9 | 182 | 0.897 |
| 2 | Surano | LE | Puglia | 1,515 | 2 | 39 | 2 | 81 | 0.860 |
| 3 | Paruzzaro | NO | Piemonte | 2,149 | 2 | 47 | 2 | 75 | 0.857 |
| 4 | Santo Stefano al Mare | IM | Liguria | 2,007 | 2 | 41 | 4 | 57 | 0.857 |
| 5 | Noventa di Piave | VE | Veneto | 6,981 | 2 | 96 | 11 | 155 | 0.847 |
| 6 | Campogalliano | MO | Emilia-Romagna | 8,546 | 2 | 178 | 10 | 279 | 0.835 |
| 7 | Gambellara | VI | Veneto | 3,440 | 2 | 56 | 3 | 94 | 0.825 |
| 8 | Montebello della Battaglia | PV | Lombardia | 1,452 | 3 | 49 | 3 | 79 | 0.825 |
| 9 | Altare | SV | Liguria | 1,927 | 3 | 47 | 2 | 80 | 0.824 |
| 10 | Rivoli Veronese | VR | Veneto | 2,246 | 3 | 56 | 4 | 89 | 0.823 |

The full top-20 lists for all three scenarios are in [`data/results/`](data/results/).

### How the priority list changes with the goal

| Priority | Top-20 composition | Pattern |
|---|---|---|
| **Frequency-focused** | 18/20 DEGURBA 2, 1 DEGURBA 3, 1 DEGURBA 1 | Denser towns with high collision volume, often touristic or commuter locations (e.g. Taormina, Aci Castello, Porto Recanati, Rezzato) |
| **Severity-focused** | 10/20 DEGURBA 2, 9/20 DEGURBA 3 | Smaller municipalities (several under 3,000 residents) where individual accidents are disproportionately deadly, likely a speed and road-design issue rather than a volume issue |
| **Equal** | 14/20 DEGURBA 2, 6/20 DEGURBA 3 | A balanced default, with no large city in the top 20 |

The largest cities score in the middle of the distribution. Among the ten biggest, Bari (0.667) and Catania (0.653) rank highest, while Naples ranks lowest (0.546).

## Limitations

- **Small-town noise persists.** Even after the >30 collisions filter and percentile normalisation, small municipalities still tend to rank higher on severity because a few serious events weigh more in a small total.
- **The threshold is a judgement call.** 30 collisions is a reliability compromise, not a statistical test.
- **Percentile ranks discard magnitude** and are not directly reusable on new data (a `QuantileTransformer` fitted on a reference set would be the production equivalent).
- **Weights are assumptions.** The 0.6/0.4 severity split and the 0.3/0.5/0.7 scenarios are design choices; the sensitivity check shows the ranking does depend on them.
- **Explanations are hypotheses, not findings.** Ideas such as moped use in Naples, terrain, or road type and speed limits are plausible but not tested here; no road-level or vehicle-type data is used.
- **Short window.** The ranking uses four post-pandemic years, so it reflects recent conditions and may not capture longer-term trends.

## How to Run

```bash
git clone <your-repo-url>
cd <your-repo-name>
pip install pandas numpy seaborn matplotlib scikit-learn jupyter
jupyter notebook
```

Run `data_preparation.ipynb` first if you want to rebuild the cleaned dataset from the raw ISTAT and SITUAS files, then `analysis.ipynb`. The analysis notebook only needs `data/istat_situas_traffic_accidents_data.csv`.

**Stack:** Python, pandas, NumPy, seaborn, matplotlib, scikit-learn.

## Data Sources

- ISTAT – road accident data by municipality
- SITUAS – territorial and population data by municipality (including DEGURBA 2021)

## Author

Marco Boscato

- LinkedIn: [linkedin.com/in/marco-boscato](https://www.linkedin.com/in/marco-boscato-292351314/)
- Email: [boscatomarco@gmail.com](mailto:boscatomarco@gmail.com)
- GitHub: [@marcoboscato](https://github.com/marcoboscato)