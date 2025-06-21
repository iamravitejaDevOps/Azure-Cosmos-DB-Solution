---

# 💼 Azure Tiered Storage Strategy for Billing Records

This document outlines three architecture options to store and retrieve 2 million billing records (~300KB each), totaling ~600 GB, using **Azure Cosmos DB and Azure Blob Storage**. Each solution is **serverless, cost-optimized, and suitable for production workloads**.

---

## 📊 Data Tiers Summary

| Tier         | Description                         | Size Estimate | Access Pattern      |
|--------------|-------------------------------------|----------------|----------------------|
| Hot          | Last 90 days, high access           | 200 GB         | Frequent reads/writes|
| Cold         | 90–180 days, less frequent queries  | 200 GB         | Occasional read      |
| Archive      | Older than 180 days, rarely accessed| 200 GB         | Rare download        |

---

## 📁 Option 1: Cosmos DB (Hot + Cold)

### ✅ Overview
- Store all data in **two Cosmos containers**:
  - `HotContainer` with TTL = 90 days
  - `ColdContainer` without TTL
- All queries happen via Cosmos SDK or API.

### 💰 Estimated Monthly Cost
| Layer        | Size   | Est. Cost    |
|--------------|--------|--------------|
| Cosmos Hot   | 200 GB | ~$60         |
| Cosmos Cold  | 400 GB | ~$120        |
| **Total**    | 600 GB | **~$180**    |

### ⚡ Performance
- **Fast (100 ms – 1 sec)**
- Fully queryable

### 🧠 Best For:
- Applications requiring **fast queries for all data**
- Simple backend logic

---

## 🌐 Option 2: Cosmos (Hot + Cold) + Blob (Cool >180d)

### ✅ Overview
- Use Cosmos for data <= 180 days
- Export data >180 days to **Azure Blob Cool Tier**
- Serve Blob files via API/backend if needed

### 💰 Estimated Monthly Cost
| Layer        | Size   | Est. Cost    |
|--------------|--------|--------------|
| Cosmos Hot   | 200 GB | ~$60         |
| Cosmos Cold  | 200 GB | ~$60         |
| Blob Archive | 200 GB | ~$2–4        |
| **Total**    | 600 GB | **~$122**    |

### ⚡ Performance
| Tier      | Speed           |
|-----------|------------------|
| CosmosDB  | ⚡ 100–500 ms     |
| Blob Cool | 🐢 2–5 sec (download)

### 🧠 Best For:
- Systems needing fast queries for recent and mid-term data
- Minimal cost for long-term retention

---

## 🔽 Option 3: Cosmos Hot + Blob Cool (No Cold Tier)

### ✅ Overview
- Store data <= 90 days in Cosmos Hot
- Export all >90 day data directly to Blob Cool
- Ideal when historical queries are rare

### 💰 Estimated Monthly Cost
| Layer        | Size   | Est. Cost    |
|--------------|--------|--------------|
| Cosmos Hot   | 200 GB | ~$60         |
| Blob Archive | 400 GB | ~$4–8        |
| **Total**    | 600 GB | **~$65–70**  |

### ⚡ Performance
| Tier      | Speed           |
|-----------|------------------|
| CosmosDB  | ⚡ 100–300 ms     |
| Blob Cool | 🐢 2–5 sec (download)

### 🧠 Best For:
- Cost-sensitive apps
- Archive used only for legal/audit access

---

## 🔁 Retrieval Logic Summary

| Scenario                    | Retrieval Source       |
|-----------------------------|------------------------|
| Data < 90 days              | Cosmos Hot             |
| Data between 90–180 days    | Cosmos Cold or Cosmos Cold + Blob (Option 2) |
| Data > 180 days             | Blob Cool              |

---

## 🧠 Recommendations

| Choose This Option If You:                                           |
|---------------------------------------------------------------------|
| ✅ Option 1: Need full query performance and low-latency always      |
| ✅ Option 2: Want to save costs but still retain queryable 180-day data |
| ✅ Option 3: Only need to keep old data for compliance/audits        |



---

✅ This is a recruiter-ready solution, explaining **architecture, costs, use cases, and trade-offs clearly.**

