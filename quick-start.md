# Quick Start Guide

## For Frontend Developers

### 1. Understand the Data

**Two main tables to query:**

```sql
-- Get all metrics for a validator
SELECT * FROM validator_metrics_by_epoch
WHERE vote_account = 'ABC123' AND epoch = 850;

-- Get tier breakdown
SELECT * FROM validator_stake_tiers_by_epoch
WHERE vote_account = 'ABC123' AND epoch = 850;
```

### 2. Implement a Simple KPI Card

**Example: Gini Coefficient Card**

```javascript
// API call
const data = await fetch('/api/validator/ABC123/metrics?epoch=850');

// Display
<div class="kpi-card">
  <h3>Gini Coefficient</h3>
  <div class="value">{data.gini_coefficient.toFixed(2)}</div>
  <div class="percentile">{data.gini_percentile_rank}th percentile</div>
  <div class="trend">
    {data.gini_pct_change > 0 ? '↑' : '↓'} {Math.abs(data.gini_pct_change)}%
  </div>
</div>
```

### 3. Implement a Simple Chart

**Example: Time Series**

```javascript
// Fetch data
const timeSeries = await fetch('/api/validator/ABC123/timeseries?start=830&end=850');

// Chart config
{
  x: timeSeries.map(d => d.epoch),
  y: timeSeries.map(d => d.gini_coefficient),
  type: 'line',
  name: 'Gini Coefficient'
}
```

### 4. Key Columns to Remember

```javascript
// Most important columns
const keyMetrics = {
  concentration: ['gini_coefficient', 'nakamoto_coeff_33', 'top_10pct_concentration'],
  counts: ['total_stakers', 'total_stake'],
  distribution: ['median_stake', 'mean_stake', 'p25', 'p75'],
  benchmarking: ['gini_percentile_rank', 'gini_z_score', 'network_median_gini']
};
```

## For Backend Developers

### 1. Set Up Tables

```sql
-- Create main metrics table
CREATE TABLE validator_metrics_by_epoch (
  vote_account STRING,
  epoch INTEGER,
  gini_coefficient DECIMAL,
  nakamoto_coeff_33 INTEGER,
  -- ... see full schema in table-schemas.md
) PARTITIONED BY (epoch);

-- Create tiers table
CREATE TABLE validator_stake_tiers_by_epoch (
  vote_account STRING,
  epoch INTEGER,
  tier_name STRING,
  -- ... see full schema in table-schemas.md
) PARTITIONED BY (epoch);
```

### 2. Implement Basic API Endpoint

```python
@app.get("/api/validator/{vote_account}/metrics")
def get_validator_metrics(vote_account: str, epoch: int):
    query = """
        SELECT * FROM validator_metrics_by_epoch
        WHERE vote_account = ? AND epoch = ?
    """
    result = trino.execute(query, [vote_account, epoch])
    return result
```

### 3. Run ETL

```sql
-- See full ETL in sql-queries.md
-- Basic flow:
1. Extract raw stake data for epoch
2. Calculate metrics per validator
3. Calculate network statistics
4. Join and insert into tables
```

## For Data Analysts

### 1. Query a Single Validator

```sql
SELECT 
    vote_account,
    epoch,
    gini_coefficient,
    nakamoto_coeff_33,
    total_stakers,
    gini_percentile_rank
FROM validator_metrics_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch = 850;
```

### 2. Compare Validators

```sql
SELECT 
    vote_account,
    gini_coefficient,
    gini_percentile_rank,
    nakamoto_coeff_33,
    total_stakers
FROM validator_metrics_by_epoch
WHERE epoch = 850
ORDER BY gini_coefficient DESC
LIMIT 10;
```

### 3. Analyze Trends

```sql
SELECT 
    epoch,
    gini_coefficient,
    network_median_gini,
    gini_percentile_rank
FROM validator_metrics_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch BETWEEN 840 AND 850
ORDER BY epoch;
```

## Testing the Setup

### 1. Verify Tables Exist

```sql
SHOW TABLES LIKE 'validator_%';
-- Should return:
--   validator_metrics_by_epoch
--   validator_stake_tiers_by_epoch
```

### 2. Check Data Quality

```sql
-- Verify data for latest epoch
SELECT COUNT(*) as validator_count
FROM validator_metrics_by_epoch
WHERE epoch = (SELECT MAX(epoch) FROM validator_metrics_by_epoch);
-- Should return ~1000 validators
```

### 3. Test API

```bash
curl http://localhost:8000/api/validator/ABC123/metrics?epoch=850
# Should return JSON with all metrics
```

## Next Steps

- **Frontend Team**: Start with [KPI Cards](kpi-cards.md), then move to [Visualizations](visualizations.md)
- **Backend Team**: Implement [API Endpoints](api-endpoints.md) and [ETL Pipeline](etl-pipeline.md)
- **Data Team**: Review [Table Schemas](table-schemas.md) and [SQL Queries](sql-queries.md)

---

**Ready to dive deeper?** Check out the full documentation in the sidebar →

