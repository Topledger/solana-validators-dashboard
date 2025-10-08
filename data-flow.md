# Data Flow

Understanding how data flows through the validator dashboard system.

---

## Overview

```
Raw Stake Data → ETL Pipeline → Metrics Tables → API → Dashboard
```

---

## 1. Data Sources

### Raw Stake Data

**Table**: `raw_stake_table` (source of truth)

**Columns**:
- `vote_account`: Validator public key
- `epoch`: Epoch number
- `staker`: Staker public key  
- `active_stake_for_epoch`: Stake amount in SOL

**Updated**: Once per epoch (~2-3 days)

**Volume**: ~50M rows (1000 validators × 50K avg stakers × recent epochs)

---

### Raw Reward Data

**Table**: `raw_staker_rewards_table`

**Columns**:
- `vote_account`: Validator public key
- `epoch`: Epoch number
- `staker`: Staker public key
- `delegator_staking_reward`: Reward amount
- `validator_commission`: Commission taken
- `active_stake_for_epoch`: Stake amount

---

## 2. ETL Pipeline

### Trigger

- **Schedule**: Runs once per epoch
- **Condition**: When new epoch data is available
- **Orchestration**: Airflow DAG

### Processing Steps

```
1. Detect new epoch completion
   ↓
2. Extract raw stake data for epoch
   ↓
3. Calculate validator-level metrics
   - Gini coefficient
   - Nakamoto coefficients
   - Percentiles
   - Concentration metrics
   ↓
4. Calculate network-wide statistics
   - Aggregates across all validators
   - Percentile ranks
   - Z-scores
   ↓
5. Calculate tier breakdowns
   - Group stakers by size
   - Aggregate per tier
   ↓
6. Calculate epoch deltas
   - Join with previous epoch
   - Calculate changes
   ↓
7. Insert into target tables
   - validator_metrics_by_epoch
   - validator_stake_tiers_by_epoch
   ↓
8. Invalidate API cache
   ↓
9. Send completion notification
```

### Runtime

- **Per validator**: ~2-5 seconds
- **1000 validators parallel**: 10-30 minutes total
- **Peak compute**: During network statistics aggregation

---

## 3. Storage Layer

### Metrics Tables

**Format**: Parquet (column-oriented)  
**Partitioning**: By epoch  
**Retention**: All historical epochs (cheap storage)

**Tables**:
- `validator_metrics_by_epoch`: 500K rows
- `validator_stake_tiers_by_epoch`: 2.5M rows

### Raw Data

**Format**: Parquet  
**Partitioning**: By epoch  
**Retention**: Last 30 epochs in hot storage, rest archived to S3

---

## 4. API Layer

### Caching Strategy

```
Browser → CDN Cache (1 hour) → API Server → Application Cache (1 hour) → Trino
```

**Cache Keys**:
- `/metrics/{vote_account}/{epoch}`: 1 hour TTL
- `/timeseries/{vote_account}`: 6 hours TTL
- `/lorenz/{vote_account}/{epoch}`: 6 hours TTL (computed on-demand)

### Query Patterns

**Fast queries** (<100ms):
- Single validator metrics (partition + index lookup)
- Time series (single partition scan)
- Network stats (pre-aggregated)

**Moderate queries** (100ms-1s):
- Lorenz curve generation (on-the-fly computation)
- Network rankings (full epoch scan with sort)

**Slow queries** (1s+):
- Complex analytics (avoided in real-time API)

---

## 5. Dashboard Layer

### Data Loading

```javascript
// Page load
1. Fetch validator metrics (cached)
2. Fetch tier breakdown (cached)
3. Render KPI cards + static charts

// User interaction
1. User changes epoch → Fetch new data
2. User requests Lorenz → Generate on-demand
3. User changes validator → Fetch new data
```

### Update Frequency

- **Dashboard**: Updates when user changes filters
- **Data**: Refreshes every epoch (~2-3 days)
- **Real-time**: Not required (analytical dashboard)

---

## 6. Data Lifecycle

### Creation

```
Epoch N completes → ETL triggered → Metrics calculated → Tables updated
```

### Access

```
User requests → API query → Cache check → DB query (if miss) → Response
```

### Archival

```
Epoch older than 90 days → Archive raw data to S3 → Keep metrics in hot storage
```

---

## Error Handling

### ETL Failures

1. **Single validator fails**: Continue with others, log error
2. **Network aggregation fails**: Retry 3 times, alert on failure
3. **Data quality issues**: Flag and skip, manual review

### API Failures

1. **Cache miss**: Query DB directly
2. **Query timeout**: Return partial data or cached version
3. **No data for epoch**: Return 404 with helpful message

---

## Monitoring

### Metrics to Track

- ETL job duration (alert if >60 min)
- API response times (p50, p95, p99)
- Cache hit rates (alert if <70%)
- Data freshness (alert if stale >2 epochs)
- Error rates (alert if >1%)

### Dashboards

- ETL pipeline status
- API performance metrics
- Data quality checks

---

## Scaling Considerations

### Current Scale

- 1000 validators
- ~50K stakers per validator (avg)
- ~50M raw stake records per epoch
- ~3M metrics records (historical)

### Future Scale (10x growth)

- 10,000 validators
- ~500M raw stake records per epoch
- ~30M metrics records

**Required changes**:
- Increased parallel processing (10x partitions)
- More aggressive caching
- Potential pre-aggregation of common queries

---

## Security

### Access Control

- API requires authentication
- Rate limiting per API key
- No direct DB access for external users

### Data Privacy

- All stake amounts are public blockchain data
- No PII stored
- Validator public keys only (no operator info)

---

## Next Steps

- **ETL Details**: See [ETL Pipeline](etl-pipeline.md)
- **Query Examples**: See [SQL Queries](sql-queries.md)
- **API Specs**: See [API Endpoints](api-endpoints.md)

