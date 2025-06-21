---

# 📈 Azure Cosmos DB Serverless Architecture for Hot & Cold Data Tiering

## 📄 Problem Statement

### You have 2 million billing records (\~300KB each), totaling \~600GB. You require:

- **Fast access to the last 3 months' records (Hot data)**
- **Long-term retention for older records (Cold data)**
- **No changes to existing API contracts**
- **No downtime or data loss**
- **Simple, serverless architecture with low operational overhead**

---

## 📝 Solution Summary

| Layer      | Technology                            | Description                                    |
| ---------- | ------------------------------------- | ---------------------------------------------- |
| Hot Data   | Azure Cosmos DB (SQL API, Serverless) | Stores last 3 months of records with TTL       |
| Cold Data  | Azure Cosmos DB (SQL API, Serverless) | Stores older records without TTL               |
| Automation | Azure Functions (Timer Trigger)       | Daily archiving of records older than 3 months |
| Access API | Cosmos SDK / REST / Functions         | Same endpoints continue to serve reads/writes  |

---

## 🌐 Architecture Overview

```
[Clients / API Layer]
        |
        v
[Cosmos DB Hot Container (90-day TTL)] <--- Write/Read
        |
        v
[Azure Function Timer Trigger - Daily]
        |
        v
[Cosmos DB Cold Container (Archival)]
```

---

## ⚖️ Cost Estimate (Monthly)

| Resource              | Size      | Cost              |
| --------------------- | --------- | ----------------- |
| Cosmos DB Storage     | 600 GB    | \$150             |
| RU Consumption (est.) | \~100M RU | \$29              |
| **Total**             |           | **\~\$180/month** |

---

## 🚀 Step-by-Step Deployment

### 1. Create Resource Group

```bash
az group create --name BillingRG --location eastus
```

### 2. Create Cosmos DB Serverless Account

```bash
az cosmosdb create \
  --resource-group BillingRG \
  --name BillingCosmosAcct \
  --locations regionName=eastus failoverPriority=0 \
  --capabilities EnableServerless
```

### 3. Create Database & Containers

```bash
az cosmosdb sql database create \
  --account-name BillingCosmosAcct \
  --resource-group BillingRG \
  --name BillingDB

# Hot Container with 90-day TTL
az cosmosdb sql container create \
  --account-name BillingCosmosAcct \
  --resource-group BillingRG \
  --database-name BillingDB \
  --name HotContainer \
  --partition-key-path "/partitionKey" \
  --ttl 7776000

# Cold Container with TTL disabled
az cosmosdb sql container create \
  --account-name BillingCosmosAcct \
  --resource-group BillingRG \
  --database-name BillingDB \
  --name ColdContainer \
  --partition-key-path "/partitionKey" \
  --ttl -1
```


---

## 🔄 Bash Script Alternative (Using Azure CLI)

```bash
#!/bin/bash
ACCOUNT="BillingCosmosAcct"
DB="BillingDB"
HOT="HotContainer"
COLD="ColdContainer"
RG="BillingRG"
CUTOFF=$(date -d "90 days ago" +%s)

# Query documents
docs=$(az cosmosdb sql query \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name $DB \
  --container-name $HOT \
  --query "SELECT c.id, c.partitionKey FROM c WHERE c.timestamp < $CUTOFF" \
  --output json)

# Move each doc
echo "$docs" | jq -c '.[]' | while read row; do
  id=$(echo $row | jq -r '.id')
  pk=$(echo $row | jq -r '.partitionKey')
  doc=$(az cosmosdb sql document show \
    --account-name $ACCOUNT \
    --resource-group $RG \
    --database-name $DB \
    --container-name $HOT \
    --id "$id" \
    --partition-key "$pk" \
    --output json)

  az cosmosdb sql document create \
    --account-name $ACCOUNT \
    --resource-group $RG \
    --database-name $DB \
    --container-name $COLD \
    --partition-key "$pk" \
    --json "$doc"

  az cosmosdb sql document delete \
    --account-name $ACCOUNT \
    --resource-group $RG \
    --database-name $DB \
    --container-name $HOT \
    --id "$id" \
    --partition-key "$pk"

done
```

---

## 📝 Final Notes

- All APIs continue reading/writing to the same HotContainer.
- Archiving is transparent and automated.
- TTL ensures HotContainer always stays lean (last 3 months only).
- Cold data remains queryable (slower, infrequent use).

🚀 **Production Ready. Zero Downtime. Low Cost. Fully Serverless.**

