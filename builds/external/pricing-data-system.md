# Pricing Data System

> Full-stack data pipeline for real-time pricing intelligence

## Overview

Built a complete data pipeline system for pricing intelligence, featuring a FastAPI backend, React frontend, and integrations with multiple data sources for real-time pricing analysis.

## The Problem

Pricing decisions were made with stale or incomplete data:
- Multiple data sources with no unified view
- Manual spreadsheet updates prone to errors
- No real-time visibility into market pricing
- Difficulty comparing across regions/products
- Reporting required hours of manual compilation

## Solution Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                      PRICING DATA SYSTEM                           │
├───────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DATA SOURCES            BACKEND                 FRONTEND          │
│  ┌──────────┐           ┌──────────┐           ┌──────────┐       │
│  │ External │           │          │           │          │       │
│  │  APIs    │──────────▶│ FastAPI  │──────────▶│  React   │       │
│  └──────────┘           │  Server  │           │   App    │       │
│  ┌──────────┐           │          │           │          │       │
│  │  Google  │──────────▶│          │           │          │       │
│  │  Sheets  │           │          │           │          │       │
│  └──────────┘           └────┬─────┘           └──────────┘       │
│  ┌──────────┐                │                                     │
│  │ Database │◀───────────────┘                                     │
│  │          │                                                      │
│  └──────────┘                                                      │
│                                                                    │
└───────────────────────────────────────────────────────────────────┘
```

## Technical Implementation

### FastAPI Backend

```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime

app = FastAPI(
    title="Pricing Data API",
    description="Real-time pricing intelligence system",
    version="1.0.0"
)

# CORS configuration for React frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Data models
class PricePoint(BaseModel):
    product_id: str
    region: str
    price: float
    currency: str
    effective_date: datetime
    source: str

class PriceQuery(BaseModel):
    product_ids: Optional[List[str]] = None
    regions: Optional[List[str]] = None
    date_from: Optional[datetime] = None
    date_to: Optional[datetime] = None

class PriceComparison(BaseModel):
    product_id: str
    prices_by_region: dict
    average_price: float
    min_price: float
    max_price: float
    variance_pct: float


# API endpoints
@app.get("/prices/{product_id}", response_model=List[PricePoint])
async def get_product_prices(
    product_id: str,
    region: Optional[str] = None,
    db: Database = Depends(get_db)
):
    """
    Get price history for a specific product.
    """
    query = {"product_id": product_id}
    if region:
        query["region"] = region

    prices = await db.prices.find(query).sort("effective_date", -1).to_list(100)

    if not prices:
        raise HTTPException(status_code=404, detail="Product not found")

    return [PricePoint(**p) for p in prices]


@app.post("/prices/compare", response_model=List[PriceComparison])
async def compare_prices(
    query: PriceQuery,
    db: Database = Depends(get_db)
):
    """
    Compare prices across regions for specified products.
    """
    comparisons = []

    for product_id in query.product_ids:
        prices = await db.prices.find({
            "product_id": product_id,
            "region": {"$in": query.regions} if query.regions else {"$exists": True}
        }).to_list(1000)

        if prices:
            prices_by_region = {}
            for p in prices:
                region = p["region"]
                if region not in prices_by_region:
                    prices_by_region[region] = []
                prices_by_region[region].append(p["price"])

            all_prices = [p["price"] for p in prices]

            comparisons.append(PriceComparison(
                product_id=product_id,
                prices_by_region={r: sum(ps)/len(ps) for r, ps in prices_by_region.items()},
                average_price=sum(all_prices) / len(all_prices),
                min_price=min(all_prices),
                max_price=max(all_prices),
                variance_pct=((max(all_prices) - min(all_prices)) / min(all_prices)) * 100
            ))

    return comparisons


@app.get("/prices/refresh")
async def refresh_prices(
    background_tasks: BackgroundTasks,
    db: Database = Depends(get_db)
):
    """
    Trigger background refresh of all price data.
    """
    background_tasks.add_task(refresh_all_prices, db)
    return {"status": "refresh_started", "timestamp": datetime.now()}
```

### Data Ingestion Pipeline

```python
class PriceIngestionPipeline:
    """
    Coordinates data ingestion from multiple sources.
    """

    def __init__(self, db: Database):
        self.db = db
        self.sources = {
            'sheets': GoogleSheetsSource(),
            'api': ExternalAPISource(),
            'manual': ManualUploadSource()
        }

    async def ingest_all(self) -> IngestionReport:
        """
        Pull data from all configured sources.
        """
        report = IngestionReport()

        for name, source in self.sources.items():
            try:
                records = await source.fetch()
                validated = self.validate_records(records)
                await self.upsert_prices(validated)

                report.add_success(name, len(validated))
            except Exception as e:
                report.add_failure(name, str(e))

        return report

    async def upsert_prices(self, prices: List[PricePoint]):
        """
        Insert or update prices with conflict resolution.
        """
        for price in prices:
            await self.db.prices.update_one(
                {
                    "product_id": price.product_id,
                    "region": price.region,
                    "effective_date": price.effective_date
                },
                {"$set": price.dict()},
                upsert=True
            )


class GoogleSheetsSource:
    """
    Ingests pricing data from Google Sheets.
    """

    def __init__(self):
        self.service = build('sheets', 'v4', credentials=self.get_credentials())

    async def fetch(self) -> List[dict]:
        """
        Fetch all rows from configured pricing sheet.
        """
        result = self.service.spreadsheets().values().get(
            spreadsheetId=PRICING_SHEET_ID,
            range='Prices!A2:F'
        ).execute()

        rows = result.get('values', [])

        return [
            {
                'product_id': row[0],
                'region': row[1],
                'price': float(row[2]),
                'currency': row[3],
                'effective_date': parse_date(row[4]),
                'source': 'google_sheets'
            }
            for row in rows
            if len(row) >= 5
        ]
```

### React Frontend

```typescript
// PricingDashboard.tsx
import React, { useState, useEffect } from 'react';
import {
  LineChart, Line, XAxis, YAxis, CartesianGrid,
  Tooltip, Legend, ResponsiveContainer
} from 'recharts';

interface PricePoint {
  productId: string;
  region: string;
  price: number;
  currency: string;
  effectiveDate: string;
}

interface PriceComparison {
  productId: string;
  pricesByRegion: Record<string, number>;
  averagePrice: number;
  minPrice: number;
  maxPrice: number;
  variancePct: number;
}

const PricingDashboard: React.FC = () => {
  const [prices, setPrices] = useState<PricePoint[]>([]);
  const [comparisons, setComparisons] = useState<PriceComparison[]>([]);
  const [selectedProduct, setSelectedProduct] = useState<string>('');
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (selectedProduct) {
      fetchPrices(selectedProduct);
    }
  }, [selectedProduct]);

  const fetchPrices = async (productId: string) => {
    setLoading(true);
    try {
      const response = await fetch(`/api/prices/${productId}`);
      const data = await response.json();
      setPrices(data);
    } catch (error) {
      console.error('Failed to fetch prices:', error);
    } finally {
      setLoading(false);
    }
  };

  const fetchComparison = async (productIds: string[]) => {
    const response = await fetch('/api/prices/compare', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ product_ids: productIds })
    });
    const data = await response.json();
    setComparisons(data);
  };

  return (
    <div className="pricing-dashboard">
      <header>
        <h1>Pricing Intelligence Dashboard</h1>
        <ProductSelector
          value={selectedProduct}
          onChange={setSelectedProduct}
        />
      </header>

      <main>
        <section className="price-chart">
          <h2>Price History</h2>
          <ResponsiveContainer width="100%" height={400}>
            <LineChart data={prices}>
              <CartesianGrid strokeDasharray="3 3" />
              <XAxis dataKey="effectiveDate" />
              <YAxis />
              <Tooltip />
              <Legend />
              <Line
                type="monotone"
                dataKey="price"
                stroke="#8884d8"
                activeDot={{ r: 8 }}
              />
            </LineChart>
          </ResponsiveContainer>
        </section>

        <section className="price-comparison">
          <h2>Regional Comparison</h2>
          <ComparisonTable comparisons={comparisons} />
        </section>

        <section className="price-alerts">
          <h2>Price Alerts</h2>
          <AlertsList productId={selectedProduct} />
        </section>
      </main>
    </div>
  );
};

// Comparison table component
const ComparisonTable: React.FC<{ comparisons: PriceComparison[] }> = ({
  comparisons
}) => (
  <table className="comparison-table">
    <thead>
      <tr>
        <th>Product</th>
        <th>Avg Price</th>
        <th>Min</th>
        <th>Max</th>
        <th>Variance</th>
      </tr>
    </thead>
    <tbody>
      {comparisons.map(c => (
        <tr key={c.productId}>
          <td>{c.productId}</td>
          <td>${c.averagePrice.toFixed(2)}</td>
          <td>${c.minPrice.toFixed(2)}</td>
          <td>${c.maxPrice.toFixed(2)}</td>
          <td className={c.variancePct > 10 ? 'high-variance' : ''}>
            {c.variancePct.toFixed(1)}%
          </td>
        </tr>
      ))}
    </tbody>
  </table>
);

export default PricingDashboard;
```

## Technologies Used

| Category | Technologies |
|----------|--------------|
| Backend | Python 3.10+, FastAPI, Pydantic |
| Frontend | React, TypeScript, Recharts |
| Database | MongoDB (or PostgreSQL) |
| Integration | Google Sheets API |
| DevOps | Docker, uvicorn |

## Results & Impact

### System Capabilities

| Feature | Capability |
|---------|------------|
| Data sources | 3+ integrated |
| Update frequency | Real-time to hourly |
| Products tracked | 100+ |
| Regions covered | 10+ |
| Report generation | < 5 seconds |

### Business Value

- **Real-time visibility** into global pricing
- **Automated alerts** for price anomalies
- **Cross-regional analysis** for arbitrage detection
- **Historical trends** for forecasting
- **Self-service reporting** for stakeholders

## Key Learnings

1. **FastAPI + Pydantic**: Type validation at the API boundary catches data issues early

2. **Background tasks**: Long-running ingestion shouldn't block API responses

3. **Upsert pattern**: Idempotent updates simplify re-ingestion and error recovery

4. **React hooks**: Clean state management for complex dashboard interactions

5. **Recharts**: Quick visualization without the complexity of D3

## Architecture Decisions

### Why FastAPI?
- Automatic OpenAPI documentation
- Pydantic validation built-in
- Async support for I/O operations
- Type hints throughout

### Why React + TypeScript?
- Type safety catches bugs at compile time
- Component reusability for dashboard widgets
- Strong ecosystem for charting libraries

### Why Background Ingestion?
- API remains responsive during data refresh
- Can schedule regular updates without blocking
- Failed ingestion doesn't affect serving

## Skills Demonstrated

- **Full-Stack Development**: Python backend + React frontend
- **API Design**: RESTful patterns, OpenAPI spec
- **Data Engineering**: ETL from multiple sources
- **Database Design**: Efficient queries, upsert patterns
- **Frontend Visualization**: Interactive charts, responsive design

---

*Part of my automation portfolio demonstrating full-stack capabilities and data pipeline design.*
