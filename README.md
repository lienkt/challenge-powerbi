# 🚆 RailPulse Analytics: Transit Performance Dashboard

## 📌 Project Overview

RailPulse Analytics is an interactive Power BI dashboard built to evaluate the operational performance of the Belgian National Railway (SNCB/NMBS).

The report connects directly to an Azure SQL Database and transforms live railway operational records into executive-level performance indicators.

The dashboard focuses on four operational questions:

1. How punctual is the railway network?
2. During which hours does operational pressure increase?
3. Which train class contributes the most accumulated delay?
4. Which platforms experience the greatest average delay at a selected station?

---

# 🎯 Business Objectives

The dashboard enables railway stakeholders to quickly identify:

- Overall network punctuality
- Hourly train-volume and delay patterns
- Train classes responsible for accumulated delay
- Platform-level congestion hotspots

Interactive station filtering allows users to move from a network-level overview to station-specific operational analysis.

---

# 📊 Data Model

The Power BI semantic model follows a star-schema-style structure with one operational fact table and two supporting dimension tables.

## Fact Table — `liveboard_records`

Contains train operational events.

Important fields:

```text
record_id
station_id
vehicle_id
destination
scheduled_departure
delay_seconds
platform
canceled
```

## Dimension Table — `stations`

```text
station_id
station_name
standard_name
longitude
latitude
```

Relationship:

```text
stations[station_id]
        1
        |
        *
liveboard_records[station_id]
```

## Dimension Table — `vehicles`

```text
id
short_name
vehicle_type
vehicle_class_code
vehicle_number
```

Relationship:

```text
vehicles[id]
        1
        |
        *
liveboard_records[vehicle_id]
```

Train categories are provided directly by:

```text
vehicles[vehicle_class_code]
```

Therefore, train classes do not need to be parsed manually from `vehicle_id`.

---

# 🧮 Calculated Column

## Departure Hour

A calculated column extracts the hour from the scheduled departure datetime.

```DAX
Departure Hour =
HOUR(liveboard_records[scheduled_departure])
```

This produces hourly categories such as:

```text
8
9
10
```

and can support the Rush Hour Matrix.

---

# 📐 DAX Measures

## Total Trains

```DAX
Total Trains =
CALCULATE(
    COUNTROWS(liveboard_records),
    liveboard_records[canceled] = FALSE()
)
```

Counts operating train records within the current filter context.

---

## Average Delay Minutes

```DAX
Average Delay Minutes =
DIVIDE(
    CALCULATE(
        AVERAGE(liveboard_records[delay_seconds]),
        liveboard_records[canceled] = FALSE()
    ),
    60,
    0
)
```

Calculates average train delay in minutes.

---

## Total Delay Minutes

```DAX
Total Delay Minutes =
DIVIDE(
    CALCULATE(
        SUM(liveboard_records[delay_seconds]),
        liveboard_records[canceled] = FALSE()
    ),
    60,
    0
)
```

Calculates accumulated delay minutes within the current filter context.

---

## On-Time Rate %

```DAX
On-Time Rate % =
VAR OperatedTrains =
    CALCULATE(
        COUNTROWS(liveboard_records),
        liveboard_records[canceled] = FALSE()
    )

VAR OnTimeTrains =
    CALCULATE(
        COUNTROWS(liveboard_records),
        liveboard_records[canceled] = FALSE(),
        liveboard_records[delay_seconds] < 120
    )

RETURN
    DIVIDE(
        OnTimeTrains,
        OperatedTrains,
        0
    )
```

A train is considered on time when:

```text
delay_seconds < 120
```

or less than two minutes late.

Canceled services are excluded from the KPI.

---

# 📈 Dashboard Analysis

## 1. Punctuality Scorecard

### Business Question

> What percentage of operating trains run on time?

### Visualization

**KPI Card**

### Metric

```text
On-Time Rate %
```

### Current Dashboard Result

```text
92.79%
```

### Insight

The current dataset shows an overall **On-Time Rate of 92.79%**.

This means approximately 93 out of every 100 operating train records are delayed by less than two minutes.

The result indicates generally strong network punctuality, while the remaining delayed services should be investigated through the hourly, train-class, and platform-level visuals.

---

# 2. Rush Hour Matrix

### Business Question

> At what time does high train volume coincide with increased delay?

### Visualization

**Hourly comparison chart**

### X-Axis

```text
Departure Hour
```

### Metrics

Train volume:

```text
Total Trains
```

Delay:

```text
Average Delay Minutes
```

### Current Dashboard Pattern

The available data currently contains departures concentrated around:

```text
08:00
09:00
10:00
```

The chart shows very different operating conditions between those hours.

At approximately **08:00**, the average delay is extremely high while train volume is very low.

At approximately **10:00**, train volume reaches its highest level while average delay is close to zero.

The **09:00 interval** falls between these two extremes.

### Insight

The current data does **not show a simple relationship where more trains automatically produce more delay**.

The 08:00 period should be treated as a possible operational anomaly because the high average delay occurs with relatively few observations.

By contrast, the 10:00 period handles the highest observed train volume while maintaining very low average delay.

Therefore, the chart suggests that the largest delay spike may be caused by specific disrupted services rather than network capacity alone.

### Interpretation Note

High average delay should always be interpreted together with train volume.

A very high average calculated from only a few trains does not necessarily represent a network-wide rush-hour bottleneck.

---

# 3. Train Class Breakdown

### Business Question

> Which train class contributes the greatest amount of accumulated delay?

### Visualization

**Clustered Bar Chart**

### Category

```text
vehicles[vehicle_class_code]
```

The visual is restricted to the challenge categories:

```text
IC
S
P
```

where records are available.

### Value

```text
Total Delay Minutes
```

### Current Dashboard Result

The dashboard shows a clear difference between the available categories.

**IC trains account for the overwhelming majority of accumulated delay minutes.**

S-class trains contribute only a small proportion by comparison.

No P-class observations are currently visible in the source data.

### Insight

IC services are currently the largest contributor to total delay minutes and should therefore receive the highest priority for further operational investigation.

However, this does not automatically mean that an individual IC service has the worst average performance.

A class may generate a large cumulative delay because it operates more frequently than other categories.

For this reason, total delay should be interpreted together with train volume before making operational decisions.

---

# 4. Platform Congestion Analysis

### Business Question

> Which platform or track operates the furthest behind schedule at a selected station?

### Visualization

**Clustered Bar Chart**

### Axis

```text
liveboard_records[platform]
```

### Value

```text
Average Delay Minutes
```

### Station Slicer

```text
stations[standard_name]
```

The user can select a station such as:

```text
Brussel-Centraal/Bruxelles-Central
Brussel-Noord/Bruxelles-Nord
Gent-Sint-Pieters
Liège-Guillemins
Brugge
Mechelen
Namur
```

### Network-Level Pattern

Before applying a station filter, the current chart shows that platforms:

```text
2
12
9
```

have the highest average delay values in the displayed dataset.

Platform 2 currently has the highest average delay, followed by platforms 12 and 9.

### Important Interpretation

Platform numbers are station-specific.

Therefore, a network-wide comparison such as "Platform 2" may combine observations from platform 2 at multiple different stations.

For an operational recommendation, a station must first be selected using the station slicer.

### Station-Level Insight

After selecting **Brussel-Centraal/Bruxelles-Central**, the chart should be used to identify the platform with the highest average delay specifically at Brussels-Central.

This allows the operator to identify potential platform-level scheduling, dispatching, or capacity issues without mixing tracks from different stations.

---

# 🎨 Dashboard Design Decisions

The report uses a top-down executive structure.

## Executive KPI

The **On-Time Rate %** is positioned prominently because punctuality is the primary network-health indicator.

## Time-Based Analysis

The Rush Hour Matrix compares train activity and average delay on the same hourly timeline.

This makes it possible to distinguish:

```text
high traffic + high delay
```

from:

```text
low traffic + isolated high delay
```

## Train-Class Comparison

A horizontal bar chart is used because it provides a clear ranking of accumulated delay across train categories.

## Platform Analysis

A horizontal bar chart ranks platforms by average delay.

A station slicer prevents platform numbers from different stations from being interpreted as the same physical track.

---

# 🔎 Current Dashboard Findings

Based on the current dashboard snapshot:

### 1. Network punctuality is relatively strong

The overall On-Time Rate is:

```text
92.79%
```

This indicates that the large majority of observed operating trains remain within the two-minute punctuality threshold.

### 2. The largest hourly delay spike is not associated with the highest train volume

Around 08:00, average delay is very high while traffic volume is relatively low.

Around 10:00, traffic volume reaches its maximum while average delay is minimal.

This suggests that isolated disruptions may currently have a greater impact on delay than traffic volume itself.

### 3. IC trains dominate cumulative delay

IC services contribute substantially more total delayed minutes than S services in the current data.

This makes IC operations the highest-priority category for deeper investigation.

### 4. Platform delays are concentrated in a small number of tracks

At network level, platforms 2, 12, and 9 currently appear at the top of the average-delay ranking.

However, operational conclusions should only be made after applying a station filter because platform numbers repeat across stations.

---

# 💡 Tactical Recommendations

## 1. Investigate the 08:00 Delay Spike

The largest average-delay spike currently occurs during a period with relatively low train volume.

Rather than assuming that rush-hour capacity is responsible, RailPulse recommends examining the individual delayed services operating during this interval.

Possible causes include:

- isolated train failure
- infrastructure disruption
- late incoming rolling stock
- operational dependency from another route

---

## 2. Prioritize IC Service Reliability

IC trains currently account for the largest share of accumulated delay minutes.

Operational teams should investigate:

- frequently delayed IC routes
- rolling-stock reliability
- timetable dependencies
- major hub connections
- recurring infrastructure conflicts

Further analysis should compare **IC train volume with average IC delay** before concluding that the train class itself is inherently less reliable.

---

## 3. Investigate High-Delay Platforms at Major Hubs

Platform-level analysis should be performed after selecting a specific station such as Brussels-Central.

Platforms showing both:

```text
high average delay
+
significant train volume
```

should receive priority.

Potential interventions include:

- timetable adjustments
- alternative platform allocation
- improved dispatch coordination
- infrastructure capacity review

---

# ⚠️ Data Interpretation Notes

## Canceled Services

Canceled train records are excluded from punctuality and delay measures.

## On-Time Definition

The challenge defines an on-time service as:

```text
delay_seconds < 120
```

A service delayed exactly two minutes is therefore classified as delayed.

## Train Classes

Train categories are taken directly from:

```text
vehicles[vehicle_class_code]
```

No artificial operational observations are created for categories absent from the source database.

## Platform Numbers

Platform numbers must be interpreted within a station context.

Platform 2 at Brussels-Central is not the same operational location as Platform 2 at another station.

---

# 📦 Deliverables

This repository contains:

- Power BI workbook file: `reports/Railpulse-dashboard.pbix`
- Public Power BI Service dashboard link
- Documented DAX calculations
- Azure SQL data model
- Dashboard preview image in README (recommended)
- Project README
- SQL schema / database scripts

---

# 🛠️ Technologies

- Power BI Service
- DAX
- Azure SQL Database
- SQL
- Relational Data Modeling
- Data Visualization
- Business Intelligence

---

# 📷 Dashboard Preview

Dashboard preview file:

[Open the dashboard preview PDF](reports/Railpulse-dashboard.pdf)

Note: a PDF can be linked from the README, but it usually will not render inline like an image on GitHub. If you want the preview to appear directly in the README, export one page as PNG or JPG and embed that image instead.

---

# 🔗 Live Dashboard

Power BI Service:

```text
https://app.powerbi.com/links/fkLHWqMKUl?ctid=33ac9060-aa07-4cb9-8fbc-4dcf3a48b1d7&pbi_source=linkShare
```
