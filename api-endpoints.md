# API Endpoints

REST API specification for the validator dashboard.

---

## Base URL

```
https://analytics.topledger.xyz/api/v1/validators
```

---

## Authentication

All requests require an API key:

```bash
curl -H "X-API-Key: YOUR_API_KEY" \
     https://analytics.topledger.xyz/api/v1/validators/{endpoint}
```

---

## Endpoints

### 1. Get Validator Metrics

**Endpoint**: `GET /validators/{vote_account}/metrics`

**Description**: Retrieve all metrics for a specific validator at a specific epoch.

**Parameters**:
- `vote_account` (path, required): Validator public key
- `epoch` (query, optional): Epoch number (default: latest)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/validators/xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk/metrics?epoch=850
```

**Response** (200 OK):
```json
{
  "vote_account": "xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk",
  "epoch": 850,
  "calculation_timestamp": "2025-10-08T12:00:00Z",
  
  "basic": {
    "total_stakers": 48234,
    "total_stake": 5200000.50
  },
  
  "concentration": {
    "gini_coefficient": 0.72,
    "nakamoto_coeff_33": 18,
    "nakamoto_coeff_51": 42,
    "top_1pct_concentration": 54.2,
    "top_5pct_concentration": 72.8,
    "top_10pct_concentration": 84.1
  },
  
  "distribution": {
    "mean_stake": 107.8,
    "median_stake": 8.9,
    "mean_median_ratio": 12.1,
    "std_dev": 1205.3,
    "p25": 2.5,
    "p50": 8.9,
    "p75": 45.2,
    "p90": 250.0,
    "p99": 5000.0
  },
  
  "benchmarking": {
    "network_median_gini": 0.68,
    "gini_percentile_rank": 65.3,
    "gini_z_score": 0.45,
    "nakamoto_percentile_rank": 35.2,
    "stakers_percentile_rank": 92.1
  },
  
  "deltas": {
    "prev_epoch_gini": 0.74,
    "gini_delta": -0.02,
    "gini_pct_change": -2.7,
    "stakers_delta": 382,
    "stakers_pct_change": 0.8
  },
  
  "rewards": {
    "avg_reward_rate_pct": 6.8,
    "median_reward_rate_pct": 6.7,
    "reward_gini_coefficient": 0.71,
    "reward_stake_gini_diff": 0.01
  },
  
  "tiers": [
    {
      "tier_name": "micro",
      "staker_count": 42000,
      "pct_of_total_stakers": 87.1,
      "pct_of_total_stake": 4.8,
      "avg_stake_in_tier": 5.9
    },
    {
      "tier_name": "small",
      "staker_count": 5000,
      "pct_of_total_stakers": 10.4,
      "pct_of_total_stake": 15.4,
      "avg_stake_in_tier": 160.0
    },
    {
      "tier_name": "medium",
      "staker_count": 1200,
      "pct_of_total_stakers": 2.5,
      "pct_of_total_stake": 21.2,
      "avg_stake_in_tier": 917.0
    },
    {
      "tier_name": "large",
      "staker_count": 30,
      "pct_of_total_stakers": 0.06,
      "pct_of_total_stake": 17.3,
      "avg_stake_in_tier": 30000.0
    },
    {
      "tier_name": "whale",
      "staker_count": 4,
      "pct_of_total_stakers": 0.008,
      "pct_of_total_stake": 41.3,
      "avg_stake_in_tier": 537500.0
    }
  ]
}
```

---

### 2. Get Time Series Data

**Endpoint**: `GET /validators/{vote_account}/timeseries`

**Description**: Retrieve historical metrics for a validator across multiple epochs.

**Parameters**:
- `vote_account` (path, required): Validator public key
- `start_epoch` (query, optional): Starting epoch (default: current - 20)
- `end_epoch` (query, optional): Ending epoch (default: latest)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/validators/xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk/timeseries?start_epoch=830&end_epoch=850
```

**Response** (200 OK):
```json
{
  "vote_account": "xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk",
  "start_epoch": 830,
  "end_epoch": 850,
  "data": [
    {
      "epoch": 830,
      "gini_coefficient": 0.75,
      "gini_delta": 0.01,
      "nakamoto_coeff_33": 16,
      "total_stakers": 45000,
      "network_median_gini": 0.70
    },
    {
      "epoch": 831,
      "gini_coefficient": 0.74,
      "gini_delta": -0.01,
      "nakamoto_coeff_33": 17,
      "total_stakers": 45500,
      "network_median_gini": 0.69
    },
    // ... more epochs
  ]
}
```

---

### 3. Get Lorenz Curve Data

**Endpoint**: `GET /validators/{vote_account}/lorenz`

**Description**: Generate Lorenz curve coordinates for visualizing inequality.

**Parameters**:
- `vote_account` (path, required): Validator public key
- `epoch` (query, optional): Epoch number (default: latest)
- `points` (query, optional): Number of data points (default: 100)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/validators/xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk/lorenz?epoch=850
```

**Response** (200 OK):
```json
{
  "vote_account": "xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk",
  "epoch": 850,
  "gini_coefficient": 0.72,
  "data": [
    {
      "cumulative_pct_stakers": 0.0,
      "cumulative_pct_stake": 0.0
    },
    {
      "cumulative_pct_stakers": 1.0,
      "cumulative_pct_stake": 0.02
    },
    {
      "cumulative_pct_stakers": 2.0,
      "cumulative_pct_stake": 0.05
    },
    // ... continues to 100%
    {
      "cumulative_pct_stakers": 100.0,
      "cumulative_pct_stake": 100.0
    }
  ]
}
```

---

### 4. Get Network Statistics

**Endpoint**: `GET /network/stats`

**Description**: Get network-wide statistics for a specific epoch.

**Parameters**:
- `epoch` (query, optional): Epoch number (default: latest)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/network/stats?epoch=850
```

**Response** (200 OK):
```json
{
  "epoch": 850,
  "network_total_validators": 1000,
  
  "gini_stats": {
    "min": 0.35,
    "q1": 0.60,
    "median": 0.68,
    "mean": 0.70,
    "q3": 0.78,
    "max": 0.95,
    "std_dev": 0.12
  },
  
  "nakamoto_stats": {
    "min": 5,
    "median": 22,
    "mean": 24,
    "max": 85,
    "std_dev": 8.5
  },
  
  "stakers_stats": {
    "median": 12500,
    "mean": 15000,
    "total_network_stakers": 15000000
  }
}
```

---

### 5. Get Network Rankings

**Endpoint**: `GET /network/rankings`

**Description**: Get validators ranked by a specific metric.

**Parameters**:
- `epoch` (query, optional): Epoch number (default: latest)
- `metric` (query, optional): Ranking metric (default: "gini_coefficient")
  - Options: `gini_coefficient`, `nakamoto_coeff_33`, `total_stakers`, `total_stake`
- `order` (query, optional): Sort order - `asc` or `desc` (default: `desc` for Gini, `asc` for Nakamoto)
- `limit` (query, optional): Number of results (default: 100, max: 1000)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/network/rankings?epoch=850&metric=gini_coefficient&order=desc&limit=10
```

**Response** (200 OK):
```json
{
  "epoch": 850,
  "metric": "gini_coefficient",
  "order": "desc",
  "total_count": 1000,
  "data": [
    {
      "rank": 1,
      "vote_account": "ValidatorA...",
      "gini_coefficient": 0.95,
      "gini_percentile_rank": 99.9,
      "nakamoto_coeff_33": 3,
      "total_stakers": 1500
    },
    {
      "rank": 2,
      "vote_account": "ValidatorB...",
      "gini_coefficient": 0.89,
      "gini_percentile_rank": 95.2,
      "nakamoto_coeff_33": 5,
      "total_stakers": 2800
    },
    // ... more validators
  ]
}
```

---

### 6. Get Auto-Generated Insights

**Endpoint**: `GET /validators/{vote_account}/insights`

**Description**: Get auto-generated insights based on metrics and statistical analysis.

**Parameters**:
- `vote_account` (path, required): Validator public key
- `epoch` (query, optional): Epoch number (default: latest)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/validators/xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk/insights?epoch=850
```

**Response** (200 OK):
```json
{
  "vote_account": "xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk",
  "epoch": 850,
  "insights": [
    {
      "type": "positive",
      "category": "decentralization",
      "message": "Ranks in top 35% for decentralization",
      "metric": "gini_percentile_rank",
      "value": 65.3
    },
    {
      "type": "positive",
      "category": "trend",
      "message": "Gini improved by 2.7% since last epoch",
      "metric": "gini_pct_change",
      "value": -2.7
    },
    {
      "type": "warning",
      "category": "concentration",
      "message": "Top 1% controls 54.2% of stake",
      "metric": "top_1pct_concentration",
      "value": 54.2
    },
    {
      "type": "warning",
      "category": "outlier",
      "message": "Significantly more concentrated than network average",
      "metric": "gini_z_score",
      "value": 2.3
    },
    {
      "type": "neutral",
      "category": "info",
      "message": "Total stakers: 48,234 (92nd percentile)",
      "metric": "stakers_percentile_rank",
      "value": 92.1
    }
  ]
}
```

---

### 7. List All Validators

**Endpoint**: `GET /validators`

**Description**: Get a list of all validators with basic info.

**Parameters**:
- `epoch` (query, optional): Epoch number (default: latest)
- `limit` (query, optional): Results per page (default: 100)
- `offset` (query, optional): Pagination offset (default: 0)

**Example Request**:
```bash
curl https://analytics.topledger.xyz/api/v1/validators?epoch=850&limit=10
```

**Response** (200 OK):
```json
{
  "epoch": 850,
  "total_count": 1000,
  "limit": 10,
  "offset": 0,
  "data": [
    {
      "vote_account": "ValidatorA...",
      "total_stakers": 48234,
      "total_stake": 5200000.50,
      "gini_coefficient": 0.72,
      "nakamoto_coeff_33": 18
    },
    // ... more validators
  ]
}
```

---

## Error Responses

### 400 Bad Request
```json
{
  "error": "Invalid epoch number",
  "code": "INVALID_PARAMETER",
  "details": "Epoch must be a positive integer"
}
```

### 404 Not Found
```json
{
  "error": "Validator not found",
  "code": "VALIDATOR_NOT_FOUND",
  "details": "No data found for vote_account in epoch 850"
}
```

### 429 Too Many Requests
```json
{
  "error": "Rate limit exceeded",
  "code": "RATE_LIMIT",
  "details": "Maximum 100 requests per minute",
  "retry_after": 30
}
```

### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "code": "INTERNAL_ERROR",
  "details": "An unexpected error occurred"
}
```

---

## Rate Limits

- **Authenticated requests**: 1000 requests/hour
- **Unauthenticated requests**: 100 requests/hour
- **Burst limit**: 10 requests/second

---

## Caching

Responses are cached with the following TTLs:

- `/metrics`: 1 hour
- `/timeseries`: 6 hours
- `/lorenz`: 6 hours
- `/network/stats`: 1 hour
- `/network/rankings`: 1 hour

Cache headers:
```
Cache-Control: public, max-age=3600
ETag: "abc123..."
```

---

## Implementation Example

### Python

```python
import requests

API_BASE = "https://analytics.topledger.xyz/api/v1/validators"
API_KEY = "your_api_key_here"

def get_validator_metrics(vote_account, epoch=None):
    headers = {"X-API-Key": API_KEY}
    params = {"epoch": epoch} if epoch else {}
    
    response = requests.get(
        f"{API_BASE}/{vote_account}/metrics",
        headers=headers,
        params=params
    )
    response.raise_for_status()
    return response.json()

# Usage
data = get_validator_metrics("xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk", epoch=850)
print(f"Gini: {data['concentration']['gini_coefficient']}")
```

### JavaScript

```javascript
const API_BASE = 'https://analytics.topledger.xyz/api/v1/validators';
const API_KEY = 'your_api_key_here';

async function getValidatorMetrics(voteAccount, epoch = null) {
  const url = new URL(`${API_BASE}/${voteAccount}/metrics`);
  if (epoch) url.searchParams.append('epoch', epoch);
  
  const response = await fetch(url, {
    headers: {
      'X-API-Key': API_KEY
    }
  });
  
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }
  
  return response.json();
}

// Usage
const data = await getValidatorMetrics('xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk', 850);
console.log(`Gini: ${data.concentration.gini_coefficient}`);
```

---

## Webhook Support (Future)

Subscribe to notifications when metrics change significantly:

```bash
POST /webhooks/subscribe
{
  "url": "https://your-server.com/webhook",
  "events": ["gini_change", "nakamoto_change"],
  "filters": {
    "vote_account": "xtA5ixQt5rvEHfqfGGJkdKqoV9TH14NPfkhsdXsshrk",
    "threshold": {
      "gini_pct_change": 5.0
    }
  }
}
```

---

## Next Steps

- **Query Examples**: See [SQL Queries](sql-queries.md) for backend implementation
- **Visualizations**: See [Visualizations](visualizations.md) for using API data
- **Schema Reference**: See [Table Schemas](table-schemas.md) for data structure

