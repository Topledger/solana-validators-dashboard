# SQL Queries

Sample Trino SQL queries for dashboard implementation and ETL.

---

## Dashboard Queries

### Single Validator Load

Get all metrics for one validator at one epoch:

```sql
-- Get main metrics
SELECT * 
FROM validator_metrics_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch = 850;

-- Get tier breakdown
SELECT 
    tier_name,
    staker_count,
    pct_of_total_stakers,
    total_stake_in_tier,
    pct_of_total_stake,
    avg_stake_in_tier
FROM validator_stake_tiers_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch = 850
ORDER BY tier_min_stake;
```

---

### Time Series Data

Get historical data for trend charts:

```sql
SELECT 
    epoch,
    gini_coefficient,
    gini_delta,
    gini_pct_change,
    nakamoto_coeff_33,
    total_stakers,
    median_stake,
    network_median_gini
FROM validator_metrics_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch BETWEEN 831 AND 850  -- Last 20 epochs
ORDER BY epoch;
```

---

### Network Rankings

Get all validators ranked by concentration:

```sql
SELECT 
    vote_account,
    gini_coefficient,
    gini_percentile_rank,
    nakamoto_coeff_33,
    total_stakers,
    top_10pct_concentration
FROM validator_metrics_by_epoch
WHERE epoch = 850
ORDER BY gini_coefficient DESC
LIMIT 100;
```

---

### Lorenz Curve Generation

Calculate Lorenz curve coordinates on-the-fly:

```sql
WITH sorted_stakes AS (
    SELECT 
        active_stake_for_epoch as stake,
        ROW_NUMBER() OVER (ORDER BY active_stake_for_epoch ASC) as rank_num,
        COUNT(*) OVER () as total_stakers,
        SUM(active_stake_for_epoch) OVER () as total_stake
    FROM raw_stake_table
    WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
      AND epoch = 850
)
SELECT
    rank_num,
    CAST(rank_num AS DOUBLE) / total_stakers * 100 as cumulative_pct_stakers,
    SUM(stake) OVER (ORDER BY rank_num) / total_stake * 100 as cumulative_pct_stake
FROM sorted_stakes
ORDER BY rank_num;
```

---

## ETL Queries

### ETL: Calculate Validator Metrics

Complete ETL for calculating all validator metrics for one epoch:

```sql
WITH 
-- Step 1: Raw stake data
raw_data AS (
    SELECT 
        vote_account,
        850 as epoch,
        staker,
        active_stake_for_epoch as stake
    FROM raw_stake_table
    WHERE epoch = 850
),

-- Step 2: Sorted stakes for Gini & Nakamoto
sorted_stakes AS (
    SELECT 
        vote_account,
        epoch,
        stake,
        ROW_NUMBER() OVER (PARTITION BY vote_account ORDER BY stake ASC) as rank_asc,
        ROW_NUMBER() OVER (PARTITION BY vote_account ORDER BY stake DESC) as rank_desc,
        COUNT(*) OVER (PARTITION BY vote_account) as n,
        SUM(stake) OVER (PARTITION BY vote_account) as total_stake,
        SUM(stake) OVER (PARTITION BY vote_account ORDER BY stake DESC 
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) as cumsum_desc
    FROM raw_data
),

-- Step 3: Calculate Gini per validator
gini_calc AS (
    SELECT
        vote_account,
        epoch,
        MAX(n) as total_stakers,
        MAX(total_stake) as total_stake,
        (2.0 * SUM(rank_asc * stake)) / (MAX(n) * SUM(stake)) 
            - (MAX(n) + 1.0) / MAX(n) as gini_coefficient
    FROM sorted_stakes
    GROUP BY vote_account, epoch
),

-- Step 4: Calculate Nakamoto per validator
nakamoto_calc AS (
    SELECT
        vote_account,
        epoch,
        MIN(CASE WHEN cumsum_desc >= MAX(total_stake) * 0.33 THEN rank_desc END) as nakamoto_coeff_33,
        MIN(CASE WHEN cumsum_desc >= MAX(total_stake) * 0.51 THEN rank_desc END) as nakamoto_coeff_51
    FROM sorted_stakes
    GROUP BY vote_account, epoch
),

-- Step 5: Calculate percentiles per validator
percentiles_calc AS (
    SELECT
        vote_account,
        epoch,
        AVG(stake) as mean_stake,
        APPROX_PERCENTILE(stake, 0.50) as median_stake,
        STDDEV(stake) as std_dev,
        APPROX_PERCENTILE(stake, 0.05) as p5,
        APPROX_PERCENTILE(stake, 0.10) as p10,
        APPROX_PERCENTILE(stake, 0.25) as p25,
        APPROX_PERCENTILE(stake, 0.75) as p75,
        APPROX_PERCENTILE(stake, 0.90) as p90,
        APPROX_PERCENTILE(stake, 0.95) as p95,
        APPROX_PERCENTILE(stake, 0.99) as p99
    FROM sorted_stakes
    GROUP BY vote_account, epoch
),

-- Step 6: Calculate top concentration per validator
top_concentration AS (
    SELECT
        vote_account,
        epoch,
        SUM(CASE WHEN rank_desc <= GREATEST(1, CAST(MAX(n) * 0.001 AS INTEGER)) 
                 THEN stake ELSE 0 END) / MAX(total_stake) * 100 as top_01pct_concentration,
        SUM(CASE WHEN rank_desc <= GREATEST(1, CAST(MAX(n) * 0.01 AS INTEGER)) 
                 THEN stake ELSE 0 END) / MAX(total_stake) * 100 as top_1pct_concentration,
        SUM(CASE WHEN rank_desc <= GREATEST(1, CAST(MAX(n) * 0.05 AS INTEGER)) 
                 THEN stake ELSE 0 END) / MAX(total_stake) * 100 as top_5pct_concentration,
        SUM(CASE WHEN rank_desc <= GREATEST(1, CAST(MAX(n) * 0.10 AS INTEGER)) 
                 THEN stake ELSE 0 END) / MAX(total_stake) * 100 as top_10pct_concentration
    FROM sorted_stakes
    GROUP BY vote_account, epoch
),

-- Step 7: Calculate network-wide statistics
network_stats AS (
    SELECT
        epoch,
        COUNT(*) as network_total_validators,
        APPROX_PERCENTILE(gini_coefficient, 0.25) as network_q1_gini,
        APPROX_PERCENTILE(gini_coefficient, 0.50) as network_median_gini,
        APPROX_PERCENTILE(gini_coefficient, 0.75) as network_q3_gini,
        AVG(gini_coefficient) as network_mean_gini,
        STDDEV(gini_coefficient) as network_stddev_gini,
        
        APPROX_PERCENTILE(nakamoto_coeff_33, 0.50) as network_median_nakamoto,
        AVG(nakamoto_coeff_33) as network_mean_nakamoto,
        STDDEV(nakamoto_coeff_33) as network_stddev_nakamoto
    FROM gini_calc
    GROUP BY epoch
),

-- Step 8: Calculate percentile ranks
percentile_ranks AS (
    SELECT
        vote_account,
        epoch,
        PERCENT_RANK() OVER (PARTITION BY epoch ORDER BY gini_coefficient) * 100 as gini_percentile_rank,
        PERCENT_RANK() OVER (PARTITION BY epoch ORDER BY nakamoto_coeff_33) * 100 as nakamoto_percentile_rank,
        PERCENT_RANK() OVER (PARTITION BY epoch ORDER BY total_stakers) * 100 as stakers_percentile_rank
    FROM gini_calc
)

-- Step 9: Combine all metrics
SELECT
    g.vote_account,
    g.epoch,
    CURRENT_TIMESTAMP as calculation_timestamp,
    
    -- Basic counts
    g.total_stakers,
    g.total_stake,
    
    -- Central tendency
    p.mean_stake,
    p.median_stake,
    p.mean_stake / NULLIF(p.median_stake, 0) as mean_median_ratio,
    
    -- Dispersion
    p.std_dev,
    p.std_dev / NULLIF(p.mean_stake, 0) as coefficient_of_variation,
    p.p75 - p.p25 as iqr,
    
    -- Percentiles
    p.p5, p.p10, p.p25, p.median_stake as p50, p.p75, p.p90, p.p95, p.p99,
    
    -- Concentration metrics
    g.gini_coefficient,
    n.nakamoto_coeff_33,
    n.nakamoto_coeff_51,
    tc.top_01pct_concentration,
    tc.top_1pct_concentration,
    tc.top_5pct_concentration,
    tc.top_10pct_concentration,
    
    -- Network benchmarking (duplicated across all validators)
    ns.network_total_validators,
    ns.network_median_gini,
    ns.network_mean_gini,
    ns.network_stddev_gini,
    ns.network_q1_gini,
    ns.network_q3_gini,
    ns.network_median_nakamoto,
    ns.network_mean_nakamoto,
    ns.network_stddev_nakamoto,
    
    -- Relative metrics (specific to this validator)
    pr.gini_percentile_rank,
    (g.gini_coefficient - ns.network_mean_gini) / NULLIF(ns.network_stddev_gini, 0) as gini_z_score,
    pr.nakamoto_percentile_rank,
    (n.nakamoto_coeff_33 - ns.network_mean_nakamoto) / NULLIF(ns.network_stddev_nakamoto, 0) as nakamoto_z_score,
    pr.stakers_percentile_rank

FROM gini_calc g
JOIN nakamoto_calc n USING (vote_account, epoch)
JOIN percentiles_calc p USING (vote_account, epoch)
JOIN top_concentration tc USING (vote_account, epoch)
CROSS JOIN network_stats ns  -- Same for all validators
LEFT JOIN percentile_ranks pr USING (vote_account, epoch);
```

**Note**: This is a simplified version covering ~70% of columns. Full ETL would include:
- Skewness and kurtosis calculations
- HHI index
- Epoch deltas (requires self-join with previous epoch)
- Reward metrics (requires joining with rewards data)

---

### ETL: Calculate Tier Metrics

Calculate stake tiers per validator:

```sql
WITH raw_data AS (
    SELECT 
        vote_account,
        850 as epoch,
        active_stake_for_epoch as stake
    FROM raw_stake_table
    WHERE epoch = 850
),

tiered_stakes AS (
    SELECT
        vote_account,
        epoch,
        stake,
        CASE 
            WHEN stake < 1000 THEN 'micro'
            WHEN stake < 10000 THEN 'small'
            WHEN stake < 100000 THEN 'medium'
            WHEN stake < 1000000 THEN 'large'
            ELSE 'whale'
        END as tier_name,
        CASE 
            WHEN stake < 1000 THEN 0
            WHEN stake < 10000 THEN 1000
            WHEN stake < 100000 THEN 10000
            WHEN stake < 1000000 THEN 100000
            ELSE 1000000
        END as tier_min_stake,
        CASE 
            WHEN stake < 1000 THEN 1000
            WHEN stake < 10000 THEN 10000
            WHEN stake < 100000 THEN 100000
            WHEN stake < 1000000 THEN 1000000
            ELSE NULL  -- Unbounded for whale
        END as tier_max_stake,
        SUM(stake) OVER (PARTITION BY vote_account) as total_stake,
        COUNT(*) OVER (PARTITION BY vote_account) as total_stakers
    FROM raw_data
)

SELECT
    vote_account,
    epoch,
    tier_name,
    tier_min_stake,
    tier_max_stake,
    
    -- Tier metrics
    COUNT(*) as staker_count,
    COUNT(*) * 100.0 / MAX(total_stakers) as pct_of_total_stakers,
    SUM(stake) as total_stake_in_tier,
    SUM(stake) * 100.0 / MAX(total_stake) as pct_of_total_stake,
    AVG(stake) as avg_stake_in_tier,
    APPROX_PERCENTILE(stake, 0.5) as median_stake_in_tier,
    MIN(stake) as min_stake_in_tier,
    MAX(stake) as max_stake_in_tier

FROM tiered_stakes
GROUP BY vote_account, epoch, tier_name, tier_min_stake, tier_max_stake
ORDER BY tier_min_stake;
```

---

## Analysis Queries

### Find Validators with High Whale Concentration

```sql
SELECT 
    v.vote_account,
    v.gini_coefficient,
    t.pct_of_total_stake as whale_pct
FROM validator_metrics_by_epoch v
JOIN validator_stake_tiers_by_epoch t 
    ON v.vote_account = t.vote_account 
    AND v.epoch = t.epoch
WHERE v.epoch = 850
  AND t.tier_name = 'whale'
  AND t.pct_of_total_stake > 50  -- Whales control >50%
ORDER BY t.pct_of_total_stake DESC;
```

---

### Compare Validator to Peers

```sql
WITH peer_group AS (
    -- Define peers as validators with similar total stake (±20%)
    SELECT v2.vote_account as peer_vote_account
    FROM validator_metrics_by_epoch v1
    CROSS JOIN validator_metrics_by_epoch v2
    WHERE v1.vote_account = 'TARGET_VALIDATOR'
      AND v1.epoch = 850
      AND v2.epoch = 850
      AND v2.total_stake BETWEEN v1.total_stake * 0.8 AND v1.total_stake * 1.2
      AND v2.vote_account != v1.vote_account
),
peer_stats AS (
    SELECT
        APPROX_PERCENTILE(gini_coefficient, 0.50) as peer_median_gini,
        AVG(gini_coefficient) as peer_mean_gini
    FROM validator_metrics_by_epoch
    WHERE vote_account IN (SELECT peer_vote_account FROM peer_group)
      AND epoch = 850
)
SELECT 
    v.vote_account,
    v.gini_coefficient as validator_gini,
    ps.peer_median_gini,
    v.gini_coefficient - ps.peer_median_gini as gini_vs_peers
FROM validator_metrics_by_epoch v
CROSS JOIN peer_stats ps
WHERE v.vote_account = 'TARGET_VALIDATOR'
  AND v.epoch = 850;
```

---

### Track Metric Changes Over Time

```sql
SELECT 
    epoch,
    gini_coefficient,
    gini_delta,
    CASE 
        WHEN gini_delta < 0 THEN 'Improving'
        WHEN gini_delta > 0 THEN 'Worsening'
        ELSE 'Stable'
    END as trend,
    AVG(gini_coefficient) OVER (
        ORDER BY epoch 
        ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
    ) as gini_ma5  -- 5-epoch moving average
FROM validator_metrics_by_epoch
WHERE vote_account = 'xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk'
  AND epoch BETWEEN 840 AND 850
ORDER BY epoch;
```

---

## Testing Queries

### Verify Data Quality

```sql
-- Check for latest epoch
SELECT MAX(epoch) as latest_epoch
FROM validator_metrics_by_epoch;

-- Count validators in latest epoch
SELECT epoch, COUNT(*) as validator_count
FROM validator_metrics_by_epoch
WHERE epoch = (SELECT MAX(epoch) FROM validator_metrics_by_epoch)
GROUP BY epoch;

-- Verify all validators have 5 tiers
SELECT vote_account, COUNT(DISTINCT tier_name) as tier_count
FROM validator_stake_tiers_by_epoch
WHERE epoch = 850
GROUP BY vote_account
HAVING COUNT(DISTINCT tier_name) != 5;  -- Should return 0 rows
```

---

## Performance Tips

### Use Partitioning

```sql
-- Good: Filters by epoch (uses partition pruning)
SELECT * FROM validator_metrics_by_epoch WHERE epoch = 850;

-- Bad: Full table scan
SELECT * FROM validator_metrics_by_epoch WHERE gini_coefficient > 0.7;
```

### Leverage Indexes

```sql
-- Create indexes for common queries
CREATE INDEX idx_vote_epoch ON validator_metrics_by_epoch(vote_account, epoch);
CREATE INDEX idx_epoch_gini ON validator_metrics_by_epoch(epoch, gini_coefficient);
```

---

## Next Steps

- **API Integration**: See [API Endpoints](api-endpoints.md) for wrapping these queries
- **Visualizations**: See [Visualizations](visualizations.md) for using query results
- **Schema Reference**: See [Table Schemas](table-schemas.md) for column details

