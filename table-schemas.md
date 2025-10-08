# Table Schemas

Complete database schema for the validator dashboard.

## Overview

**Two-table architecture** for efficient storage and querying:

| Table | Rows | Columns | Granularity |
|-------|------|---------|-------------|
| `validator_metrics_by_epoch` | ~500K | 105 | 1 row per validator per epoch |
| `validator_stake_tiers_by_epoch` | ~2.5M | 15 | 5 rows per validator per epoch |

**Total storage**: ~3M rows (highly efficient for 1000 validators × 500 epochs)

---

## Table 1: validator_metrics_by_epoch

### Purpose
Stores all pre-calculated metrics for each validator at each epoch, including concentration metrics, network benchmarking, and epoch-over-epoch deltas.

### Schema

```sql
CREATE TABLE validator_metrics_by_epoch (
    -- IDENTITY
    vote_account                    STRING      -- Validator public key
    epoch                          INTEGER     -- Epoch number
    calculation_timestamp          TIMESTAMP   -- When metrics were calculated

    -- BASIC COUNTS
    total_stakers                  INTEGER     -- Number of unique stakers
    total_stake                    DECIMAL     -- Total SOL staked

    -- CENTRAL TENDENCY
    mean_stake                     DECIMAL     -- Average stake per staker
    median_stake                   DECIMAL     -- 50th percentile stake
    mean_median_ratio              DECIMAL     -- mean_stake / median_stake

    -- DISPERSION
    std_dev                        DECIMAL     -- Standard deviation
    coefficient_of_variation       DECIMAL     -- std_dev / mean_stake
    iqr                           DECIMAL     -- Q3 - Q1 (interquartile range)

    -- PERCENTILES
    p5                            DECIMAL     -- 5th percentile
    p10                           DECIMAL     -- 10th percentile
    p25                           DECIMAL     -- 25th percentile (Q1)
    p50                           DECIMAL     -- 50th percentile (median)
    p75                           DECIMAL     -- 75th percentile (Q3)
    p90                           DECIMAL     -- 90th percentile
    p95                           DECIMAL     -- 95th percentile
    p99                           DECIMAL     -- 99th percentile

    -- CONCENTRATION METRICS
    gini_coefficient              DECIMAL     -- 0-1 scale, inequality measure
    nakamoto_coeff_33             INTEGER     -- Stakers needed for 33% stake
    nakamoto_coeff_51             INTEGER     -- Stakers needed for 51% stake
    hhi_index                     DECIMAL     -- Herfindahl-Hirschman Index (0-10000)
    top_01pct_concentration       DECIMAL     -- % stake controlled by top 0.1%
    top_1pct_concentration        DECIMAL     -- % stake controlled by top 1%
    top_5pct_concentration        DECIMAL     -- % stake controlled by top 5%
    top_10pct_concentration       DECIMAL     -- % stake controlled by top 10%

    -- DISTRIBUTION SHAPE
    skewness                      DECIMAL     -- Asymmetry measure
    kurtosis                      DECIMAL     -- Tail heaviness measure

    -- NETWORK BENCHMARKING (duplicated across all validators for same epoch)
    network_total_validators      INTEGER     -- Total validators in network
    network_median_gini           DECIMAL     -- Median Gini across all validators
    network_mean_gini             DECIMAL     -- Mean Gini across all validators
    network_stddev_gini           DECIMAL     -- Std dev of Gini
    network_q1_gini              DECIMAL     -- Q1 of Gini distribution
    network_q3_gini              DECIMAL     -- Q3 of Gini distribution
    network_median_nakamoto       DECIMAL     -- Median Nakamoto across validators
    network_mean_nakamoto         DECIMAL     -- Mean Nakamoto across validators
    network_stddev_nakamoto       DECIMAL     -- Std dev of Nakamoto
    network_median_stakers        INTEGER     -- Median stakers per validator
    network_mean_stakers          INTEGER     -- Mean stakers per validator
    network_stddev_stakers        DECIMAL     -- Std dev of stakers

    -- RELATIVE METRICS (specific to this validator)
    gini_percentile_rank          DECIMAL     -- 0-100, where validator ranks
    gini_z_score                  DECIMAL     -- Standard deviations from mean
    nakamoto_percentile_rank      DECIMAL     -- 0-100, where validator ranks
    nakamoto_z_score              DECIMAL     -- Standard deviations from mean
    stakers_percentile_rank       DECIMAL     -- 0-100, where validator ranks
    stakers_z_score               DECIMAL     -- Standard deviations from mean

    -- EPOCH DELTAS (vs previous epoch)
    prev_epoch_gini               DECIMAL     -- Gini in epoch-1
    gini_delta                    DECIMAL     -- Change from prev epoch
    gini_pct_change               DECIMAL     -- % change from prev epoch
    prev_epoch_stakers            INTEGER     -- Stakers in epoch-1
    stakers_delta                 INTEGER     -- Change from prev epoch
    stakers_pct_change            DECIMAL     -- % change from prev epoch
    prev_epoch_stake              DECIMAL     -- Total stake in epoch-1
    stake_delta                   DECIMAL     -- Change from prev epoch
    stake_pct_change              DECIMAL     -- % change from prev epoch

    -- REWARD METRICS
    total_rewards_distributed     DECIMAL     -- Sum of all delegator rewards
    avg_reward_per_staker         DECIMAL     -- Mean reward
    median_reward_per_staker      DECIMAL     -- Median reward
    reward_gini_coefficient       DECIMAL     -- Gini of reward distribution
    reward_stake_gini_diff        DECIMAL     -- |reward_gini - stake_gini|
    avg_reward_rate_pct           DECIMAL     -- Mean APY %
    median_reward_rate_pct        DECIMAL     -- Median APY %
    min_reward_rate_pct           DECIMAL     -- Min APY %
    max_reward_rate_pct           DECIMAL     -- Max APY %
    total_commission_collected    DECIMAL     -- Sum of commission
    avg_commission_per_staker     DECIMAL     -- Mean commission
    validator_commission_pct      DECIMAL     -- Validator commission %

    -- REWARD STATISTICAL TESTS
    reward_stake_correlation      DECIMAL     -- Spearman ρ (stake vs reward rate)
    reward_stake_corr_pvalue      DECIMAL     -- P-value (optional, can calc on-fly)

    PRIMARY KEY (vote_account, epoch)
) PARTITIONED BY (epoch);
```

### Key Columns Explained

**Gini Coefficient** (`gini_coefficient`)
- Range: 0 (perfect equality) to 1 (total inequality)
- Measures stake concentration
- Lower is better (more decentralized)

**Nakamoto Coefficient** (`nakamoto_coeff_33`)
- Number of stakers needed to control 33% of stake
- Higher is better (more decentralized)
- Critical for Byzantine fault tolerance

**Z-Score** (`gini_z_score`)
- Standard deviations from network mean
- |z| < 1: Normal
- |z| > 2: Outlier

**Percentile Rank** (`gini_percentile_rank`)
- 0-100 scale showing position among all validators
- 0 = lowest Gini (best)
- 100 = highest Gini (worst)

---

## Table 2: validator_stake_tiers_by_epoch

### Purpose
Breaks down stake distribution into 5 tiers (micro, small, medium, large, whale) for each validator at each epoch.

### Schema

```sql
CREATE TABLE validator_stake_tiers_by_epoch (
    -- IDENTITY
    vote_account                  STRING      -- Validator public key
    epoch                        INTEGER     -- Epoch number
    tier_name                    STRING      -- 'micro', 'small', 'medium', 'large', 'whale'
    tier_min_stake               DECIMAL     -- Minimum stake for tier
    tier_max_stake               DECIMAL     -- Maximum stake for tier (NULL for whale)

    -- TIER METRICS
    staker_count                 INTEGER     -- Number of stakers in tier
    pct_of_total_stakers         DECIMAL     -- % of total validator stakers
    total_stake_in_tier          DECIMAL     -- Sum of stake in tier
    pct_of_total_stake           DECIMAL     -- % of validator's total stake
    avg_stake_in_tier            DECIMAL     -- Mean stake in tier
    median_stake_in_tier         DECIMAL     -- Median stake in tier
    min_stake_in_tier            DECIMAL     -- Min stake in tier
    max_stake_in_tier            DECIMAL     -- Max stake in tier

    PRIMARY KEY (vote_account, epoch, tier_name)
) PARTITIONED BY (epoch);
```

### Tier Definitions

| Tier | Range | Typical Staker Profile |
|------|-------|------------------------|
| **micro** | 0 - 1,000 SOL | Retail/small investors |
| **small** | 1,000 - 10,000 SOL | Regular retail investors |
| **medium** | 10,000 - 100,000 SOL | Serious individual investors |
| **large** | 100,000 - 1,000,000 SOL | Institutions/wealthy individuals |
| **whale** | 1,000,000+ SOL | Major institutions/exchanges |

### Example Data

For validator `ABC123` at epoch `850`:

| tier_name | staker_count | pct_of_total_stakers | pct_of_total_stake |
|-----------|--------------|----------------------|--------------------|
| micro | 42,000 | 87.1% | 4.8% |
| small | 5,000 | 10.4% | 15.4% |
| medium | 1,200 | 2.5% | 21.2% |
| large | 30 | 0.06% | 17.3% |
| whale | 4 | 0.008% | 41.3% |

**Key Insight**: 87% of stakers are micro, but 4 whales control 41% of stake.

---

## Partitioning Strategy

Both tables are **partitioned by epoch** for optimal query performance:

```sql
PARTITIONED BY (epoch)
```

### Benefits:
- Queries filtering by epoch only scan relevant partitions
- Easy to drop old epochs if needed
- Efficient for time-series queries

### Example Query Performance:

```sql
-- Scans only 1 partition (fast)
SELECT * FROM validator_metrics_by_epoch 
WHERE epoch = 850;

-- Scans 20 partitions (still fast)
SELECT * FROM validator_metrics_by_epoch 
WHERE epoch BETWEEN 831 AND 850;
```

---

## Indexes

Recommended indexes for optimal performance:

```sql
-- Primary lookup: validator + epoch
CREATE INDEX idx_vote_epoch 
ON validator_metrics_by_epoch(vote_account, epoch);

-- Network rankings
CREATE INDEX idx_epoch_gini 
ON validator_metrics_by_epoch(epoch, gini_coefficient);

-- Percentile lookups
CREATE INDEX idx_epoch_rank 
ON validator_metrics_by_epoch(epoch, gini_percentile_rank);
```

---

## Data Types

| Type | Example Values | Precision |
|------|----------------|-----------|
| `STRING` | Vote account addresses | Variable |
| `INTEGER` | Epoch numbers, counts | 64-bit |
| `DECIMAL` | SOL amounts, percentages | 18,6 precision |
| `TIMESTAMP` | Calculation timestamps | Millisecond precision |

---

## Storage Estimates

### validator_metrics_by_epoch
- Rows: 1,000 validators × 500 epochs = 500,000 rows
- Columns: 105
- Size: ~50-100 MB (Parquet compressed)

### validator_stake_tiers_by_epoch
- Rows: 1,000 validators × 500 epochs × 5 tiers = 2,500,000 rows
- Columns: 15
- Size: ~100-200 MB (Parquet compressed)

**Total**: ~150-300 MB for complete historical data

---

## Next Steps

- **Implement ETL**: See [ETL Pipeline](etl-pipeline.md)
- **Query Examples**: See [SQL Queries](sql-queries.md)
- **API Design**: See [API Endpoints](api-endpoints.md)

