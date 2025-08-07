# 🤖 ChatGPT Prompt Log for Azure Cost Optimization Challenge

This project was ideated and refined using ChatGPT-4o to assist in designing a scalable, cost-effective solution architecture for managing billing data in Azure. Below are the prompts used during the process to explore, validate, and enhance the implementation.

---
**Prompt:**

### 🧠 Phase 1: Understanding the Problem
I have a serverless system in Azure storing over 2M billing records in Cosmos DB. Each record is ~300KB. It's read-heavy, but records older than 3 months are rarely accessed. How can I reduce costs while keeping data accessible within seconds and without changing API contracts?
---

## 🧱 Designing the Architecture
Propose an Azure architecture using Cosmos DB and Blob Storage for hot/cold data. It should be serverless, cost-efficient, have no downtime during transition, and maintain existing API behavior. Add a metadata index layer if needed.

---

## 🧪 Data Movement Strategy

Write Python pseudocode for a timer-triggered Azure Function that:

Queries Cosmos DB for records older than 90 days

Moves them to Blob Storage

Writes a metadata reference into Cosmos DB

Deletes the original record (optional)

---

## 🔐 Infra & Permissions

List the Azure services required and how to set them up securely using Azure CLI or Terraform: Cosmos DB, Blob Storage, Function App, App Insights, and IAM.

Copy
Edit
What Managed Identity roles should be used for secure access between Azure Functions, Cosmos DB, and Blob Storage?

yaml
Copy
Edit

---

## ⚠️ Failure Modes & Edge Cases

List possible failure points in this setup:

Missing metadata

Blob latency

Blob access denied

Partial archive batches
And how to monitor, alert, and handle these scenarios?

Copy
Edit
How to monitor cold data access rate and consider rehydration of frequently accessed blobs into Cosmos DB?

yaml
Copy
Edit

---

## 🧽 Final Touches

Write a clean, professional README for this solution:

Explain architecture

Include ASCII diagram

Add code snippets and pseudocode

Make it look like a production-grade repo
