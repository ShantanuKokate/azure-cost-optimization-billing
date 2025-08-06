# 📦 Azure Billing Records Tiered Storage Optimization

## 🧠 Architecture (ASCII Diagram)

```
                         +---------------------+
                         |     CLIENT / API    |
                         +----------+----------+
                                    |
                          Read / Write API Calls
                                    |
                                    v
                          +---------+----------+
                          |     Unified Access  |
                          |      Python API     |
                          +----+-----------+----+
                               |           |
                  Recent (<90d)|           | Older (>90d)
                      Access   |           |
                               v           v
                    +----------+--+    +---+----------------+
                    | Cosmos DB   |    | Azure Blob Storage |
                    |  (Hot Tier) |    |   (Cold Tier)       |
                    +-------------+    +---------------------+

                              ^
                              |
                   +----------+-----------+
                   | Azure Function Timer |
                   | Archive Records >90d |
                   +----------------------+
```

---

## 📁 Folder Structure

```
azure-billing-optimizer/
├── api/
│   └── unified_access.py         # Unified read logic (Cosmos + fallback Blob)
├── functions/
│   └── archive_old_records.py    # Archival script (moves old records)
├── infrastructure/
│   └── main.bicep                # Deploys Cosmos DB, Blob, Function App
├── utils/
│   └── logger.py                 # Logging to App Insights
├── .env.template                 # Sample env vars
├── requirements.txt             # Python dependencies
└── README.md                    # Project overview and deployment
```

---

## 🧾 Sample: unified_access.py

```python
from azure.cosmos import CosmosClient, exceptions
from azure.storage.blob import BlobServiceClient
import json, os
from utils.logger import log_event

COSMOS_CLIENT = CosmosClient(os.getenv("COSMOS_ENDPOINT"), os.getenv("COSMOS_KEY"))
DATABASE_NAME = os.getenv("DATABASE_NAME")
CONTAINER_NAME = os.getenv("CONTAINER_NAME")
BLOB_CONN_STRING = os.getenv("BLOB_CONN_STRING")
BLOB_CONTAINER = os.getenv("BLOB_CONTAINER")

blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STRING)
blob_client = blob_service.get_container_client(BLOB_CONTAINER)

def get_billing_record(record_id, customer_id):
    try:
        container = COSMOS_CLIENT.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)
        item = container.read_item(item=record_id, partition_key=customer_id)
        return item
    except exceptions.CosmosResourceNotFoundError:
        blob_path = f"{customer_id}/{record_id}.json"
        try:
            blob_data = blob_client.get_blob_client(blob_path).download_blob().readall()
            return json.loads(blob_data)
        except Exception:
            log_event("record_not_found", {"id": record_id})
            return None
```

---

## 🧾 Sample: archive_old_records.py

```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
from datetime import datetime, timedelta
import json, os

COSMOS_CLIENT = CosmosClient(os.getenv("COSMOS_ENDPOINT"), os.getenv("COSMOS_KEY"))
DATABASE_NAME = os.getenv("DATABASE_NAME")
CONTAINER_NAME = os.getenv("CONTAINER_NAME")
BLOB_CONN_STRING = os.getenv("BLOB_CONN_STRING")
BLOB_CONTAINER = os.getenv("BLOB_CONTAINER")

blob_service = BlobServiceClient.from_connection_string(BLOB_CONN_STRING)
blob_client = blob_service.get_container_client(BLOB_CONTAINER)

cutoff_date = datetime.utcnow() - timedelta(days=90)

def archive_old_records():
    container = COSMOS_CLIENT.get_database_client(DATABASE_NAME).get_container_client(CONTAINER_NAME)
    for record in container.query_items("SELECT * FROM c WHERE c.timestamp < @cutoff",
                                        parameters=[{"name": "@cutoff", "value": cutoff_date.isoformat()}],
                                        enable_cross_partition_query=True):
        blob_path = f"{record['customer_id']}/{record['id']}.json"
        blob_client.upload_blob(blob_path, json.dumps(record), overwrite=True)
        container.delete_item(item=record['id'], partition_key=record['customer_id'])
```

---

## 🧾 Sample: main.bicep (infrastructure)

```bicep
resource cosmosDb 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: 'billingCosmosDb'
  location: resourceGroup().location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: resourceGroup().location
        failoverPriority: 0
      }
    ]
  }
}

resource blobStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'billingarchiveblob'
  location: resourceGroup().location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {}
}

resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'billing-archive-func'
  location: resourceGroup().location
  kind: 'functionapp'
  properties: {
    serverFarmId: '/subscriptions/xxx/resourceGroups/xxx/providers/Microsoft.Web/serverfarms/xxx'
  }
}
```

---

## 🧾 Sample: logger.py

```python
import logging

def log_event(event_name, details):
    logging.info(f"Event: {event_name} | Details: {details}")
```

---

## ✅ .env.template

```env
COSMOS_ENDPOINT=https://xxxx.documents.azure.com
COSMOS_KEY=xxx==
DATABASE_NAME=billing-db
CONTAINER_NAME=records
BLOB_CONN_STRING=DefaultEndpointsProtocol=https;AccountName=...
BLOB_CONTAINER=billing-archive
```

---

## 🧪 Sample Blob Storage Structure

```
customer123/
├── abcde123.json
├── 2024-02-01-bill.json
```

---

## 🚨 Recommended Alerts

| Event                    | Alert Type        |
|-------------------------|-------------------|
| Archival failures       | Error alert       |
| Blob cold access spikes | Metric alert      |
| Blob read latency > 5s  | Performance alert |

---

## 🔮 Future Enhancements

| Risk                   | Mitigation                            |
|------------------------|----------------------------------------|
| Cold read latency      | Add Redis cache or Premium blob tier  |
| Rehydration logic      | Keep blob-to-Cosmos script ready      |
| Monitoring gaps        | Add App Insights custom dashboards    |
| Security gaps          | Use RBAC, private endpoints, encryption |

---

## ✅ Summary

- ⚡ Hot data in Cosmos DB (< 90 days)
- ❄️ Cold data in Blob Storage (> 90 days)
- 🔁 Unified access logic hides tier logic
- 📉 Reduces cost significantly while maintaining performance
- 🔒 Safe, zero-downtime, no API changes

---

## 📄 License

MIT License  
Author: Shantanu Kokate
