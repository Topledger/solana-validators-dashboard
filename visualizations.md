# Visualizations

Complete specifications for all dashboard charts.

> **Note**: All visualizations reference columns from `validator_metrics_by_epoch` and `validator_stake_tiers_by_epoch` tables. See [Table Schemas](table-schemas.md) for details.

---

## KPI Cards (Top Row)

### Card 1: Gini Coefficient

**Metric**: `gini_coefficient`

**Display**:
- Value: `0.72` (2 decimals)
- Subtitle: `"65th percentile"` (from `gini_percentile_rank`)
- Trend: `↑ 2.7%` or `↓ 2.7%` (from `gini_pct_change`)
- Context: `"vs Network: 0.68"` (from `network_median_gini`)

**Color Coding**:
- Green: `< 0.5`
- Yellow: `0.5 - 0.7`
- Red: `> 0.7`

---

### Card 2: Nakamoto Coefficient

**Metric**: `nakamoto_coeff_33`

**Display**:
- Value: `18 stakers`
- Subtitle: `"35th percentile"` (from `nakamoto_percentile_rank`)
- Trend: Arrow based on `nakamoto_delta`
- Context: `"Network median: 22"` (from `network_median_nakamoto`)

---

### Card 3: Total Stakers

**Metric**: `total_stakers`

**Display**:
- Value: `48,234` (formatted with commas)
- Subtitle: `"92nd percentile"` (from `stakers_percentile_rank`)
- Trend: `stakers_pct_change`
- Context: `"Network median: 12.5K"` (from `network_median_stakers`)

---

### Card 4: Median Stake

**Metric**: `median_stake`

**Display**:
- Value: `1,234 SOL`
- Subtitle: `"58th percentile"`
- Interpretation: **Lower = more accessible**

---

### Card 5: Top 10% Concentration

**Metric**: `top_10pct_concentration`

**Display**:
- Value: `67.5%`
- Subtitle: Dynamic based on value

**Color Coding**:
- Green: `< 50%` → "Good distribution"
- Yellow: `50-70%` → "Moderate concentration"
- Red: `> 70%` → "High concentration"

---

## Lorenz Curve

**Chart Type**: Line chart

**Data Source**: Calculate on-the-fly from raw stake data

### Axes

**X-Axis**:
- Column: `cumulative_pct_stakers`
- Label: "Cumulative % of Stakers"
- Range: 0-100%

**Y-Axis**:
- Column: `cumulative_pct_stake`
- Label: "Cumulative % of Stake"
- Range: 0-100%

### Lines to Plot

1. **Perfect Equality**: `y = x` (45° diagonal)
   - Color: Light gray, dashed

2. **This Validator** (current epoch)
   - Color: Blue, bold
   - Data: From SQL query below

3. **Comparison Line** (based on filter):
   - If "vs Network": Gray dashed (average Lorenz)
   - If "vs Previous Epoch": Blue dotted

### Annotations

- Gini coefficient value in text box
- Shaded area between equality line and validator curve

### SQL Query

```sql
WITH sorted_stakes AS (
    SELECT 
        active_stake_for_epoch as stake,
        ROW_NUMBER() OVER (ORDER BY active_stake_for_epoch ASC) as rank_num,
        COUNT(*) OVER () as total_stakers,
        SUM(active_stake_for_epoch) OVER () as total_stake
    FROM raw_stake_table
    WHERE vote_account = :vote_account AND epoch = :epoch
)
SELECT
    rank_num,
    CAST(rank_num AS DOUBLE) / total_stakers * 100 as cumulative_pct_stakers,
    SUM(stake) OVER (ORDER BY rank_num) / total_stake * 100 as cumulative_pct_stake
FROM sorted_stakes
ORDER BY rank_num
```

---

## Stake Tier Distribution

**Chart Type**: Treemap (primary) or Dual Bar Chart (alternative)

**Data Source**: `validator_stake_tiers_by_epoch` table

### Query

```sql
SELECT 
    tier_name,
    staker_count,
    pct_of_total_stakers,
    total_stake_in_tier,
    pct_of_total_stake,
    avg_stake_in_tier
FROM validator_stake_tiers_by_epoch
WHERE vote_account = :vote_account AND epoch = :epoch
ORDER BY tier_min_stake
```

### Treemap Specification

**Rectangle Size**: `pct_of_total_stake`  
**Rectangle Label**: `tier_name + pct_of_total_stake`  
**Rectangle Color**: Gradient  
- Micro → Light blue
- Small → Blue
- Medium → Orange
- Large → Red
- Whale → Dark red

**Tooltip**:
```
Tier: {tier_name}
Stakers: {staker_count} ({pct_of_total_stakers}%)
Stake: {total_stake_in_tier} SOL ({pct_of_total_stake}%)
Avg: {avg_stake_in_tier} SOL
```

### Alternative: Dual Bar Chart

**X-Axis**: `tier_name` (categorical)  
**Y-Axis Left**: `pct_of_total_stakers` (blue bars)  
**Y-Axis Right**: `pct_of_total_stake` (orange bars)

Shows the mismatch between staker count and stake control.

---

## Time Series Chart

**Chart Type**: Multi-line chart

**Data Source**: `validator_metrics_by_epoch` table

### Query

```sql
SELECT 
    epoch,
    gini_coefficient,
    gini_delta,
    nakamoto_coeff_33,
    total_stakers,
    network_median_gini
FROM validator_metrics_by_epoch
WHERE vote_account = :vote_account 
  AND epoch BETWEEN :start_epoch AND :end_epoch
ORDER BY epoch
```

### Axes

**X-Axis**: `epoch`  
- Label: "Epoch"
- Range: Last 20 epochs (default)

**Y-Axis** (dual):
- Left: `gini_coefficient` (0-1 scale)
- Right: `nakamoto_coeff_33`, `total_stakers` (normalized)

### Lines

1. **Gini Coefficient** (blue, left axis)
2. **Network Median Gini** (gray dashed, left axis)
3. *(Optional)* Nakamoto Coefficient (orange, right axis)

### Features

- Highlight current epoch with vertical line
- Tooltip shows all values + deltas
- Annotations for significant changes

---

## Concentration Bar Chart

**Chart Type**: Horizontal bar chart

**Data Source**: `validator_metrics_by_epoch` table

### Metrics

- `top_1pct_concentration` → "Top 1%"
- `top_5pct_concentration` → "Top 5%"
- `top_10pct_concentration` → "Top 10%"

### Axes

**X-Axis**: Percentage (0-100%)  
- Label: "% of Total Stake Controlled"

**Y-Axis**: Categorical labels
- "Top 1% of stakers"
- "Top 5% of stakers"  
- "Top 10% of stakers"

### Features

- Bars colored based on value (green/yellow/red)
- Vertical reference line at network median
- Value labels at bar ends

**Color Coding**:
- Green: `< 40%`
- Yellow: `40-60%`
- Red: `> 60%`

---

## Network Position Radar Chart

**Chart Type**: Radar/spider chart with 6 dimensions

**Data Source**: `validator_metrics_by_epoch` table

### Dimensions (all 0-100 scale)

1. **Decentralization**: `100 - gini_percentile_rank`  
   *(Inverted: lower Gini = better)*

2. **Nakamoto**: `nakamoto_percentile_rank`  
   *(Higher = better)*

3. **Popularity**: `stakers_percentile_rank`  
   *(More stakers = better)*

4. **Accessibility**: `100 - percentile_rank(median_stake)`  
   *(Lower median = more accessible)*

5. **Reward Rate**: `percentile_rank(avg_reward_rate_pct)`  
   *(Higher reward = better)*

6. **Fairness**: `100 - percentile_rank(reward_stake_gini_diff)`  
   *(Lower diff = fairer)*

### Visualization

- Filled polygon area = this validator's profile
- Reference circle at 50th percentile (dotted)
- Each axis labeled with dimension name + actual value
- Larger area = better overall performance

---

## Z-Score Deviation Chart

**Chart Type**: Diverging horizontal bar chart

**Data Source**: `validator_metrics_by_epoch` table

### Metrics

- `gini_z_score` → "Gini Coefficient"
- `nakamoto_z_score` → "Nakamoto Coefficient"
- `stakers_z_score` → "Total Stakers"

### Axes

**X-Axis**: Z-score (-3 to +3)  
- Center line at 0 (network mean)
- Label: "Standard Deviations from Network Mean"

**Y-Axis**: Metric names (categorical)

### Features

**Bars**:
- Extend left for negative z (below average)
- Extend right for positive z (above average)

**Color Coding**:
- Green: `|z| < 1` (normal, within 1σ)
- Yellow: `1 < |z| < 2` (unusual)
- Red: `|z| > 2` (outlier)

**Annotations**:
- Z-score value at bar end
- Reference lines at -2σ, -1σ, 0, +1σ, +2σ

---

## Box Plot Comparison

**Chart Type**: Box-and-whisker plot (3 boxes side-by-side)

**Data Source**: Percentiles from `validator_metrics_by_epoch`

### Boxes

1. **This Validator** (current epoch)
2. **Previous Epoch**
3. **Network Median**

### Box Components

- **Min**: `p5`
- **Q1**: `p25`
- **Median**: `p50` (thick line)
- **Q3**: `p75`
- **Max**: `p95`

### Axes

**Y-Axis**: Stake amount (SOL)  
- Scale: Logarithmic (recommended)

**X-Axis**: Categorical (box labels)

---

## Reward Distribution Scatter Plot

**Chart Type**: Scatter plot with regression line

**Data Source**: Raw staker data

### Query

```sql
SELECT 
    active_stake_for_epoch as stake,
    delegator_staking_reward / active_stake_for_epoch * 100 as reward_rate_pct
FROM raw_staker_rewards_table
WHERE vote_account = :vote_account AND epoch = :epoch
```

### Axes

**X-Axis**: `stake` (log scale)  
- Label: "Stake Amount (SOL)"

**Y-Axis**: `reward_rate_pct`  
- Label: "Reward Rate (%)"

### Features

- Each staker as a point
- Alpha transparency (many overlapping points)
- Color by tier (optional)
- Regression line (should be flat if fair)
- Reference line at `median_reward_rate_pct`

### Annotation

Display: `reward_stake_correlation`  
Interpretation: "No correlation = fair distribution"

---

## Epoch Delta Table

**Chart Type**: Data table with sparklines

**Data Source**: `validator_metrics_by_epoch` (last 10 epochs)

### Columns

1. **Epoch**: `epoch`
2. **Gini**: `gini_coefficient` (2 decimals)
3. **Δ Gini**: `gini_delta` (with ↑↓ arrow)
4. **% Change**: `gini_pct_change` (color coded)
5. **Nakamoto**: `nakamoto_coeff_33`
6. **Stakers**: `total_stakers` (formatted)
7. **Trend**: Inline sparkline (10-epoch history)

### Sparkline

- Mini line chart in table cell
- Shows last 10 epochs
- Direction indicators: ↗ (up), ↘ (down), → (stable)

### Color Coding

- Green text: Positive change (desirable)
- Red text: Negative change (undesirable)

---

## Key Insights Panel

**Display**: Text panel with auto-generated bullet points

**Data Source**: `validator_metrics_by_epoch` table

### Logic

**✅ Positive Insights** (green checkmark):
```javascript
if (gini_percentile_rank < 35) {
    "Ranks in top 35% for decentralization"
}
if (gini_pct_change < -3) {
    "Gini improved by {abs(gini_pct_change)}% since last epoch"
}
if (nakamoto_coeff_33 > network_median_nakamoto) {
    "Nakamoto coefficient above network average"
}
```

**⚠️ Warnings** (yellow warning):
```javascript
if (top_1pct_concentration > 50) {
    "Top 1% controls {top_1pct_concentration}% of stake"
}
if (gini_z_score > 2) {
    "Significantly more concentrated than network average"
}
```

**📊 Neutral** (info icon):
```
"Total stakers: {total_stakers} ({stakers_percentile_rank}th percentile)"
"Median stake: {median_stake} SOL"
```

Display 5-7 insights, prioritized by importance.

---

## Implementation Priority

### Phase 1 (MVP)
1. ✅ KPI Cards (all 5)
2. ✅ Lorenz Curve
3. ✅ Stake Tier Treemap
4. ✅ Time Series Chart
5. ✅ Simple table view

### Phase 2
6. Network Position Radar
7. Z-Score Chart
8. Concentration Bar Chart
9. Box Plot Comparison
10. Insights Panel

### Phase 3
11. Reward Distribution Scatter
12. Epoch Delta Table
13. Additional filters/controls

---

## Next Steps

- **Sample Data**: See [SQL Queries](sql-queries.md) for test queries
- **API Integration**: See [API Endpoints](api-endpoints.md)
- **Table Structure**: See [Table Schemas](table-schemas.md)

