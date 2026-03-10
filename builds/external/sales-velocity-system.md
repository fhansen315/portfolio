# Sales Velocity Analytics System

> Real-time pipeline intelligence and bottleneck detection for enterprise sales

## Overview

Built a modular sales velocity analytics system that transforms CRM data into actionable pipeline insights, identifying bottlenecks and predicting deal outcomes with 95%+ forecasting precision.

## The Problem

Sales teams lacked visibility into pipeline health beyond basic reports. Manual analysis took hours and missed critical patterns:
- No real-time velocity tracking
- Stuck opportunities went unnoticed
- Bottleneck identification was reactive, not proactive
- Leadership lacked predictive insights for forecasting

## Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SALES VELOCITY SYSTEM                     │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  CRM Data    │  │   Velocity   │  │   Output     │       │
│  │  Extractor   │──│  Calculator  │──│  Formatter   │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│         │                 │                 │               │
│         ▼                 ▼                 ▼               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │   Stage      │  │  Bottleneck  │  │   Multi-     │       │
│  │  Analyzer    │  │  Detector    │  │   Format     │       │
│  └──────────────┘  └──────────────┘  │   Export     │       │
│                                      └──────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### Core Components

**1. Velocity Calculator**
- Implements classic formula: `(Opportunities × Avg Deal Size × Win Rate) / Median Sales Cycle`
- Uses median (robust to outliers) vs. average for cycle time
- Intelligent date parsing with timezone handling and fallbacks

**2. Stage Analyzer**
- Tracks time-in-stage for every opportunity
- Identifies stuck deals using configurable thresholds
- Generates stage-by-stage conversion metrics

**3. Bottleneck Detection Algorithm**
- Weighted scoring: `score = avg_days + (stuck_count × 5)`
- Context-aware recommendations based on thresholds
- Prioritizes stages with highest revenue at risk

### Technologies Used

| Category | Technologies |
|----------|--------------|
| Language | Python 3.10+ |
| Data Processing | Pandas, NumPy |
| CRM Integration | Salesforce CLI, REST API |
| Output Formats | CSV, JSON, Markdown |

## Results & Impact

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Analysis time | 4 hours | 5 minutes | 48x faster |
| Bottleneck detection | Reactive | Proactive | Days earlier |
| Forecasting precision | ~70% | 95%+ | +35% |

### Business Value

- **Revenue protection**: Early identification of at-risk deals
- **Process optimization**: Data-driven pipeline improvements
- **Leadership visibility**: Real-time dashboards

## Key Learnings

1. **Median over Mean**: Sales cycle data is heavily skewed; median provides more accurate velocity
2. **Weighted scoring**: Simple linear combinations outperform complex ML for bottleneck detection
3. **Multi-format export**: Different stakeholders need different formats

## Skills Demonstrated

- **Data Engineering**: ETL from CRM, data transformation, statistical analysis
- **Python**: Clean architecture, type hints, error handling
- **Business Logic**: Translating sales concepts into algorithms
- **Integration**: CRM API consumption and CLI tooling

---

*Part of my automation portfolio demonstrating the transition from SDR to technical roles.*
