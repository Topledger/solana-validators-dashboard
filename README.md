# Solana Validator Dashboard - Technical Specification

> Comprehensive documentation for implementing the Solana Validator Stake Distribution Analytics Dashboard

## Overview

This dashboard provides deep insights into Solana validator stake distribution patterns, concentration metrics, and staker behavior. It enables validators, investors, and the Solana community to understand decentralization and stake distribution dynamics.

## Key Features

### Dashboard 1: Validator-Level Analysis
- **Concentration Metrics**: Gini coefficient, Nakamoto coefficient, top % concentration
- **Stake Distribution**: Lorenz curves, histograms, tier analysis
- **Network Benchmarking**: Compare against all validators
- **Time Series**: Track metrics across epochs
- **Reward Analysis**: Distribution fairness and APY metrics

### Data Architecture
- **2 core tables** storing pre-calculated metrics
- **~3M rows** across both tables (very efficient)
- **105 columns** in main metrics table
- **Epoch-level granularity** (updated every 2-3 days)

## Quick Links

📊 **[View All Visualizations](visualizations.md)** - Complete chart specifications

📁 **[Table Schemas](table-schemas.md)** - Database design and column definitions

🔍 **[SQL Queries](sql-queries.md)** - Sample queries and ETL logic

🚀 **[API Endpoints](api-endpoints.md)** - Backend API specifications

## Technology Stack

- **Database**: Trino (with Parquet files)
- **Storage**: Column-oriented (Parquet)
- **ETL**: SQL-based, runs once per epoch
- **Visualization**: To be implemented by frontend team

## Key Metrics

| Metric | Purpose | Range |
|--------|---------|-------|
| **Gini Coefficient** | Inequality measure | 0 (equal) to 1 (concentrated) |
| **Nakamoto Coefficient** | Decentralization score | # stakers for 33% control |
| **Top 10% Concentration** | Whale dominance | % of stake held by top 10% |
| **Percentile Rank** | Network position | 0-100 percentile |
| **Z-Score** | Statistical outlier detection | Standard deviations from mean |

## Data Granularity

- **Validator Level**: One validator, one epoch
- **Network Level**: All validators, one epoch
- **Time Series**: One validator, multiple epochs
- **Tier Level**: Breakdown by stake size categories

## Getting Started

1. **Understand the data architecture** → [Table Schemas](table-schemas.md)
2. **Review visualization requirements** → [Visualizations](visualizations.md)
3. **Check sample queries** → [SQL Queries](sql-queries.md)
4. **Implement API endpoints** → [API Endpoints](api-endpoints.md)

## Version

**Version**: 2.0  
**Last Updated**: October 8, 2025  
**Status**: Ready for implementation

## Support

For questions or feedback:
- Create an issue in this repository
- Contact the data team

---

**Next**: [Quick Start Guide](quick-start.md) →

