# 🔥 Greece Wildfire Analysis — 2020–2024

> A 5-year multi-source analysis of wildfire incidents across Greece, integrating Greek Fire Service operational data, hourly meteorological station readings, and prefecture-level geospatial coordinates into a unified Power BI data model.

[![Portfolio](https://img.shields.io/badge/Portfolio-Live-C05020?style=flat-square)](https://theofanisko.github.io/greece-wildfire-analysis)
[![Report](https://img.shields.io/badge/Full%20Report-PDF-C05020?style=flat-square)](https://drive.google.com/file/d/1npPH_aG50PyfU7GB8guyf3dbLT3plkR1/view?usp=sharing)
[![Power BI](https://img.shields.io/badge/Built%20with-Power%20BI-F2C811?style=flat-square&logo=powerbi&logoColor=black)](https://powerbi.microsoft.com)

---

## Overview

Greece experienced some of its most destructive wildfire seasons on record during 2020–2024, with over **3.87M acres burned** across **27,000+ incidents**. This project goes beyond aggregate statistics to investigate the meteorological, operational, and geographic factors that differentiate contained fires from catastrophic ones.

The core research questions driving this analysis:

- Do weather conditions at the precise moment of ignition predict final burned area?
- Is there a measurable pattern to which years become catastrophic, and why?
- Which prefectures exhibit structural resource gaps independent of annual variation?
- How does aerial vs. ground resource deployment correlate with fire outcomes?

---

## Data Sources

| Source | Description | Granularity | Period |
|---|---|---|---|
| **Greek Fire Service (ΠΥ)** | Incident logs — personnel, vehicles, aircraft, timestamps, burned area | Per incident | 2020–2024 |
| **Meteorological Stations** | Hourly readings per prefecture — temp, humidity, wind speed/gusts, precipitation, pressure, sunshine | Hourly · Per prefecture | 2020–2024 |
| **Geographic Coordinates** | Lat/Lon centroids per Greek prefecture | Prefecture level | Static |

> **Note on the meteorological join:** Weather data was matched to each fire incident at **hour-of-ignition precision** using DAX `LOOKUPVALUE` on a composite key of `[Prefecture, DateTime_Hour]` — not daily averages. This captures actual ignition-time conditions rather than smoothed daily estimates. ~9% of records had no matching weather observation (station gaps) and return `BLANK()`.

---

## Data Model

Star schema built in Power BI Desktop:

```
Φύλλο1 (Fact — Fire Incidents)
    ├── Bridge_Calendar  →  date-based time intelligence
    ├── Bridge_Νομοί     →  prefecture dimension (resolves M:M)
    ├── Καιρός           →  weather fact (joined via LOOKUPVALUE)
    └── Συντεταγμένες    →  lat/lon for map visuals
```

Bridge tables were necessary to resolve many-to-many relationships between the fire incident table and the weather/calendar dimensions, enabling correct filter propagation across all DAX measures.

---

## Feature Engineering

### Ignition-Time Weather Join

```dax
-- Calculated Column on Φύλλο1
Temp_Έναρξης =
LOOKUPVALUE(
    'Καιρός'[temp],
    'Καιρός'[Νομός], 'Φύλλο1'[Νομός],
    'Καιρός'[time],  'Φύλλο1'[Έναρξη_Hour]
)
```

Applied identically for `rhum`, `wspd`, `wpgt`, `prcp`, `dwpt`.

`Έναρξη_Hour` is a DAX calculated column that truncates the ignition timestamp to the nearest hour:

```dax
Έναρξη_Hour =
DATEVALUE(FORMAT([ΕΝΑΡΞΗ], "YYYY-MM-DD")) +
TIME(HOUR([ΕΝΑΡΞΗ]), 0, 0)
```

---

### Simplified Fire Weather Index (FWI)

A custom risk-scoring model inspired by the Canadian Forest Fire Weather Index System, adapted to available variables:

```dax
FWI_Έναρξης =
VAR _temp = VALUE('Φύλλο1'[Temp_Έναρξης])
VAR _rhum = VALUE('Φύλλο1'[Rhum_Έναρξης])
VAR _wspd = VALUE('Φύλλο1'[Wspd_Έναρξης])
VAR _prcp = VALUE('Φύλλο1'[Prcp_Έναρξης])

VAR _temp_score =
    SWITCH(TRUE(),
        _temp >= 40, 5, _temp >= 35, 4,
        _temp >= 30, 3, _temp >= 25, 2,
        _temp >= 20, 1, 0)

VAR _rhum_score =
    SWITCH(TRUE(),
        _rhum <= 10, 5, _rhum <= 20, 4,
        _rhum <= 30, 3, _rhum <= 40, 2,
        _rhum <= 50, 1, 0)

VAR _wspd_score =
    SWITCH(TRUE(),
        _wspd >= 60, 5, _wspd >= 45, 4,
        _wspd >= 30, 3, _wspd >= 15, 2,
        _wspd >= 5,  1, 0)

VAR _prcp_score =
    IF(ISBLANK(_prcp) || _prcp = 0, 2, 0)

RETURN
    IF(
        ISBLANK(_temp) || ISBLANK(_rhum) || ISBLANK(_wspd),
        BLANK(),
        _temp_score + _rhum_score + _wspd_score + _prcp_score
    )
```

**Scoring range:** 0–17, categorised as:

| Score | Category |
|---|---|
| 0–3 | Low |
| 4–6 | Moderate |
| 7–10 | High |
| 11–17 | Extreme |

---

### Key Measures

```dax
-- Year-over-Year burned area change
YoY_Area_Variation_Pct =
VAR _curr = SUM('Φύλλο1'[Γεωγρ_Έκταση])
VAR _prev =
    CALCULATE(
        SUM('Φύλλο1'[Γεωγρ_Έκταση]),
        DATEADD('Bridge_Calendar'[Date], -1, YEAR)
    )
RETURN DIVIDE(_curr - _prev, _prev)

-- Relative humidity YoY delta (climate indicator)
Hydration_YoY_Pct =
VAR _prev =
    CALCULATE(
        AVERAGE('Καιρός'[rhum]),
        DATEADD('Bridge_Calendar'[Date], -1, YEAR)
    )
RETURN DIVIDE(AVERAGE('Καιρός'[rhum]) - _prev, _prev)

-- Resource effectiveness ratios
Acres_Per_Aircraft =
DIVIDE(
    SUM('Φύλλο1'[Γεωγρ_Έκταση]),
    SUM('Φύλλο1'[Total_Εναέρια_μέσα])
)
```

---

## Key Findings

**1. Extreme FWI conditions (1.41% of incidents) produce ~450 acres/incident on average**, versus ~5 acres under Low conditions. The distribution is highly right-skewed — the majority of damage is concentrated in a small fraction of events.

**2. A biennial catastrophic cycle is evident:** 2021 (+525% YoY) and 2023 (+520% YoY) were devastating, while 2022 and 2024 contracted sharply (−72% each). This is consistent with a fuel load accumulation model — vegetation recovers in the calm year and provides fuel for the next peak.

**3. 2023 was the worst year on record** at 1.74M acres burned, with total fire-fighting completion hours peaking at 2M.

**4. July is the peak ignition month** (6K incidents), with the FWI time series closely tracking the burned area curve — confirming the index's predictive validity within this dataset.

**5. Evros and Euboea are structural outliers** — not just high-severity years. Both prefectures show persistently high burned area, long completion hours, and high aerial deployment simultaneously. High resource commitment, severe outcomes: a pattern inconsistent with temporary under-resourcing.

**6. Aerial assets are associated with 770 acres/unit** — vs. 38.8 acres/vehicle and 15.3 acres/person. This ratio likely reflects reactive deployment (aircraft dispatched to already-large fires) rather than inefficiency per se, but warrants investigation into pre-emptive deployment thresholds.

---

## Dashboard Pages

| Page | Focus |
|---|---|
| **Operational Overview** | KPIs, prefecture map, resource deployment by region, scatter of aerial effectiveness |
| **Seasonality & Climate** | Fire size by category, monthly incident counts, humidity/temperature scatter, FWI line chart |
| **Trends & Forecast** | YoY burned area time series, YoY% variation, hydration trend, Power BI auto-forecast |
| **FWI Risk Index** | FWI distribution donut, avg burned area by FWI category, FWI vs. area scatter, monthly FWI line |

---

## Limitations & Future Work

- **Cause of ignition** not available from ΠΥ — prevents analysis of arson vs. accidental vs. natural ignition
- **Response time** (report → first unit arrival) not in dataset — completion hours used as proxy
- **Land cover / vegetation type** not integrated — Corine Land Cover (CLMS) would improve fuel load modeling
- **Wind direction** (`wdir`) collected but not yet used — directional spread analysis is a natural extension
- The simplified FWI is not the canonical Canadian FWI system — it lacks the fuel moisture codes (FFMC, DMC, DC) that require multi-day accumulation data

---

## Tech Stack

- **Power BI Desktop** — data model, DAX measures, report pages
- **Power Query (M)** — source transformation, type casting, filtered rows
- **DAX** — calculated columns, measures, time intelligence, LOOKUPVALUE joins
- **Excel / CSV** — raw data sources (ΠΥ incident logs, weather station exports)

---

## Author

**Theofanis Kotsis**
[LinkedIn](https://www.linkedin.com/in/theofanis-kotsis) · [GitHub](https://github.com/theofanisko) · theofanisko@gmail.com
