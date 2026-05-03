# Logistics Operations Performance Analysis





### Summary

This project analyzes a logistics operations database to evaluate service reliability, detention, route efficiency, driver performance, and facility bottlenecks. The analysis shows that the network moves a large volume of freight, but operational performance is inconsistent. On-time delivery performance is below what a mature logistics operation should target, and detention is high enough to indicate meaningful inefficiencies at pickup and delivery points.

The data suggests that the biggest opportunities are not random. Certain routes, facilities, and operating patterns consistently underperform, which means service quality can be improved through targeted operational changes rather than broad, expensive interventions. 


<img width="3224" height="664" alt="image" src="https://github.com/user-attachments/assets/c21f87ac-2347-4d02-894e-33f096a8e323" />
<img width="2770" height="1858" alt="image" src="https://github.com/user-attachments/assets/76f45781-8e0e-4c7b-b17f-159ac3c96276" />
<img width="2385" height="1345" alt="image" src="https://github.com/user-attachments/assets/a38ac7bf-4493-4f94-a796-4058a27d4ccf" />
<img width="2385" height="1345" alt="image" src="https://github.com/user-attachments/assets/21587b9c-d6b9-4ef3-a789-fdea9c4fd82c" />
<img width="2382" height="1345" alt="image" src="https://github.com/user-attachments/assets/d137c16b-3a51-45db-8a85-971a5a881cc3" />
<img width="2385" height="1345" alt="image" src="https://github.com/user-attachments/assets/10af98b2-63b9-45c3-b6a6-a418ddabf425" />




## Table of Contents

- [Objective](#objective)
- [Data Source](#data-source)
- [Tools](#tools)
- [Data Preparation](#data-Preparation)
- [Data Quality Checks](#data-quality-checks)
- [Exploratory Data Analysis](#exploratory-data-analysis)
- [KPI Summary](#kpi-summary)
- [Key Findings](#key-findings)
- [Recommendations](#recommendations)
- [SQL Code Used](#SQL-Code-Used)
- [Conclusion](#conclusion)

## Objective

The objective of this project is to analyze the logistics operating database to answer one core business question:

**How can the business improve delivery reliability, reduce detention, and increase transportation efficiency without increasing operating cost?**

To answer this, the project examines shipment performance, trip efficiency, route-level variation, driver-level behavior, and facility-level bottlenecks. The goal is to identify the operational drivers behind late deliveries and detention, then translate those findings into concrete actions that improve service quality and network efficiency.

## Data Source

This is the [dataset](https://www.kaggle.com/datasets/yogape/logistics-operations-database?utm_source=chatgpt.com&select=customers.csv). It contains a realistic logistics operations schema.

Core tables used in the analysis include:

`Loads`
`Trips`
`Delivery Events`
`Routes`
`Drivers`
`Customers`
`Facilities`
`Trucks`

Supporting tables available for extension include:

`Fuel Purchases`
`Maintenance Records`
`Safety Incidents`
`Driver Monthly Metrics`
`Truck Utilization Metrics`
`Trailers`

## Tools

- Python
- Pandas
- NumPy
- Matplotlib
- Seaborn
- SQL
- Excel

## Data Preparation

A master analysis table was created by joining the operational tables.

```sql
SELECT
    l.load_id,
    l.customer_id,
    l.route_id,
    l.load_date,
    l.load_type,
    l.weight_lbs,
    l.pieces,
    l.revenue,
    l.fuel_surcharge,
    l.accessorial_charges,
    l.load_status,
    l.booking_type,
    t.trip_id,
    t.driver_id,
    t.truck_id,
    t.trailer_id,
    t.dispatch_date,
    t.actual_distance_miles,
    t.actual_duration_hours,
    t.fuel_gallons_used,
    t.average_mpg,
    t.idle_time_hours,
    t.trip_status,
    r.origin_city,
    r.origin_state,
    r.destination_city,
    r.destination_state,
    r.typical_distance_miles,
    r.base_rate_per_mile,
    r.fuel_surcharge_rate,
    r.typical_transit_days,
    d.employment_status,
    d.years_experience,
    d.home_terminal,
    c.customer_type,
    c.primary_freight_type,
    e.on_time_rate,
    e.avg_detention_minutes,
    e.delivery_events
FROM Loads l
LEFT JOIN Trips t
    ON l.load_id = t.load_id
LEFT JOIN Routes r
    ON l.route_id = r.route_id
LEFT JOIN Drivers d
    ON t.driver_id = d.driver_id
LEFT JOIN Customers c
    ON l.customer_id = c.customer_id
LEFT JOIN (
    SELECT
        trip_id,
        AVG(CASE WHEN on_time_flag = 1 THEN 1.0 ELSE 0.0 END) AS on_time_rate,
        AVG(detention_minutes) AS avg_detention_minutes,
        COUNT(*) AS delivery_events
    FROM Delivery Events
    GROUP BY trip_id
) e
    ON t.trip_id = e.trip_id;
```


## Data Quality Checks

Before analysis, the following checks were performed:

Duplicate check

```sql
SELECT load_id, COUNT(*)
FROM Loads
GROUP BY load_id
HAVING COUNT(*) > 1;
```

Null check

```sql
SELECT
    SUM(CASE WHEN load_id IS NULL THEN 1 ELSE 0 END) AS missing_load_id,
    SUM(CASE WHEN route_id IS NULL THEN 1 ELSE 0 END) AS missing_route_id,
    SUM(CASE WHEN revenue IS NULL THEN 1 ELSE 0 END) AS missing_revenue
FROM Loads;
```

Date range check

```sql
SELECT
    MIN(load_date) AS min_load_date,
    MAX(load_date) AS max_load_date
FROM Loads;
```

Range check for key metrics

```sql
SELECT
    MIN(revenue) AS min_revenue,
    MAX(revenue) AS max_revenue,
    MIN(actual_distance_miles) AS min_distance,
    MAX(actual_distance_miles) AS max_distance,
    MIN(average_mpg) AS min_mpg,
    MAX(average_mpg) AS max_mpg
FROM Loads l
LEFT JOIN Trips t
    ON l.load_id = t.load_id;
```


## Exploratory Data Analysis

### 1. Revenue Trend Analysis

```python
monthly = analysis_df.groupby('load_month').agg(
    loads=('load_id', 'count'),
    revenue=('revenue', 'sum'),
    avg_revenue=('revenue', 'mean')
).reset_index().sort_values('load_month')
```

```python
plt.figure(figsize=(12, 5))
sns.lineplot(data=monthly, x='load_month', y='avg_revenue', marker='o', color='#0f766e')
plt.title('Average Revenue per Load by Month')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

### 2. On-Time Performance Trend

```python
monthly_service = analysis_df.groupby('load_month').agg(
    on_time=('on_time_rate', 'mean'),
    detention=('avg_detention_minutes', 'mean')
).reset_index().sort_values('load_month')
```

```python
plt.figure(figsize=(12, 5))
sns.lineplot(data=monthly_service, x='load_month', y='on_time', marker='o', color='#b45309')
plt.title('On-Time Rate by Month')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

### 3. Route Performance Analysis

```python
route_perf = analysis_df.groupby(['route_id', 'origin_city', 'destination_city']).agg(
    loads=('load_id', 'count'),
    revenue=('revenue', 'sum'),
    avg_on_time=('on_time_rate', 'mean'),
    avg_detention=('avg_detention_minutes', 'mean')
).reset_index().sort_values(['avg_on_time', 'avg_detention'])
```

```python
route_perf.head(10)
```

### 4. Driver Performance Analysis

```python
driver_perf = analysis_df.groupby(['driver_id', 'employment_status']).agg(
    loads=('load_id', 'count'),
    revenue=('revenue', 'sum'),
    avg_on_time=('on_time_rate', 'mean'),
    avg_detention=('avg_detention_minutes', 'mean'),
    avg_mpg=('average_mpg', 'mean'),
    years_experience=('years_experience', 'max')
).reset_index().sort_values(['avg_on_time', 'avg_detention'])
```

```python
driver_perf.head(10)
```

### 5. Facility Bottleneck Analysis

```python
facility_perf = deliv.groupby('facility_id').agg(
    events=('event_id', 'count'),
    avg_detention=('detention_minutes', 'mean'),
    on_time_rate=('on_time_flag', 'mean')
).reset_index().merge(
    facilities[['facility_id', 'facility_name', 'city', 'state', 'facility_type', 'dock_doors']],
    on='facility_id',
    how='left'
).sort_values(['avg_detention', 'on_time_rate'], ascending=[False, True])
```

```python
facility_perf.head(10)
```

## KPI Summary

| KPI | Value |
|---|---:|
| Loads | 85,410 |
| Total Revenue | $257,264,551 |
| Average Revenue per Load | $3,073.71 |
| Median Revenue per Load | $2,827.98 |
| On-Time Rate | 44.6% |
| Average Detention Minutes | 91.5 |
| Average Distance Miles | 1,430.3 |
| Average Duration Hours | 24.0 |
| Average MPG | 6.5 |
| Average Idle Hours | 7.01 |

## Key Findings

### Service performance is the biggest issue

The on-time rate is **44.6%**, which is a clear signal that the network is underperforming on customer service.

### Detention is significant

Average detention is **91.5 minutes**, suggesting meaningful inefficiency at terminals, docks, or delivery sites.

### Route performance is uneven

Some routes are much weaker than others, which means the problem is concentrated and can be attacked lane by lane.

### Facility operations matter

Certain facilities consistently show higher detention, pointing toward dock scheduling or warehouse process issues.

### Driver and truck efficiency vary

Differences in MPG and idle time show that vehicle utilization and operating behavior are not uniform across the fleet.

## Recommendations

### 1. Target the worst facilities first

The biggest service gains are likely to come from fixing bottlenecks at the locations with the highest detention.

### 2. Review weak lanes and appointment scheduling

Routes with low on-time rates should be reviewed for transit planning, dwell time, and dock coordination issues.

### 3. Use scorecards to manage operations weekly

Create route, driver, and facility scorecards so exceptions are visible before they become customer problems.

### 4. Improve driver and truck assignment logic

Use historical performance to assign better-performing resources to difficult or high-value lanes.

### 5. Track detention as a core KPI

Detention should be managed as a first-class operating metric, not just a byproduct of delivery delay.

## SQL Code Used

### KPI query

```sql
SELECT
    COUNT(*) AS loads,
    COUNT(DISTINCT trip_id) AS trips,
    SUM(revenue) AS total_revenue,
    AVG(revenue) AS avg_revenue_per_load,
    AVG(actual_distance_miles) AS avg_distance_miles,
    AVG(actual_duration_hours) AS avg_duration_hours,
    AVG(average_mpg) AS avg_mpg,
    AVG(idle_time_hours) AS avg_idle_hours
FROM analysis_dataset;
```

### Monthly trend query

```sql
SELECT
    DATE_TRUNC('month', load_date) AS month,
    COUNT(*) AS loads,
    SUM(revenue) AS revenue,
    AVG(revenue) AS avg_revenue,
    AVG(on_time_rate) AS on_time_rate,
    AVG(avg_detention_minutes) AS avg_detention_minutes
FROM analysis_dataset
GROUP BY 1
ORDER BY 1;
```

### Route performance query

```sql
SELECT
    route_id,
    origin_city,
    destination_city,
    COUNT(*) AS loads,
    SUM(revenue) AS revenue,
    AVG(on_time_rate) AS avg_on_time_rate,
    AVG(avg_detention_minutes) AS avg_detention_minutes
FROM analysis_dataset
GROUP BY 1, 2, 3
ORDER BY avg_on_time_rate ASC, avg_detention_minutes DESC
LIMIT 10;
```

### Driver performance query

```sql
SELECT
    driver_id,
    employment_status,
    COUNT(*) AS loads,
    SUM(revenue) AS revenue,
    AVG(on_time_rate) AS avg_on_time_rate,
    AVG(avg_detention_minutes) AS avg_detention_minutes,
    AVG(average_mpg) AS avg_mpg,
    MAX(years_experience) AS years_experience
FROM analysis_dataset
GROUP BY 1, 2
ORDER BY avg_on_time_rate ASC, avg_detention_minutes DESC
LIMIT 10;
```

### Facility performance query

```sql
SELECT
    d.facility_id,
    f.facility_name,
    f.city,
    f.state,
    COUNT(*) AS events,
    AVG(d.detention_minutes) AS avg_detention_minutes,
    AVG(CASE WHEN d.on_time_flag THEN 1.0 ELSE 0.0 END) AS on_time_rate
FROM delivery_events d
LEFT JOIN facilities f
    ON d.facility_id = f.facility_id
GROUP BY 1, 2, 3, 4
ORDER BY avg_detention_minutes DESC
LIMIT 10;
```

The final story is simple:

**the business is moving freight at scale, but service reliability and detention control need improvement**.

