# RetailScan - Scan, Pay, Go System

## 📚 Table of Contents

### User Journeys (Mobile App and Admin Dashboard)
- [1. Mobile User Journey (Scan, Pay, and Go with Loyalty Integration)](#1-mobile-user-journey-scan-pay-and-go-with-loyalty-integration)
- [2. Admin Dashboard (Brand Manager)](#2-admin-dashboard-brand-manager)
- [3. API Integrations with ERP](#3-api-integrations-with-erp)

### Architecture & Strategy
- [🧭 General Context](#-general-context)
- [🧩 Architecture Overview (Macro-level)](#-architecture-overview-macro-level)
- [🔄 Dual Stream Strategy: Product vs Price/Stock Updates](#-dual-stream-strategy-product-vs-pricestock-updates)
- [🔁 Web-Queue-Worker Pattern](#-web-queue-worker-pattern)
- [⚙️ Price/Stock Pipeline (Real-Time Friendly)](#-pricestock-pipeline-real-time-friendly)
- [📦 Product Import Pipeline (Staging → Publish)](#-product-import-pipeline-staging--publish)

### Optimization Strategies
- [🧠 Read Optimization with Redis Pattern](#-read-optimization-with-redis-pattern)
- [⚡ Real-Time Cache Busting + Event-Driven Sync](#-real-time-cache-busting--event-driven-sync)
- [⏱️ TTL Strategy, Read Preferences & Mongo-Specific Optimizations](#-ttl-strategy-read-preferences--mongo-specific-optimizations)
- [🔄 Application vs Database-Level Updates (Write Optimization Strategy)](#-application-vs-database-level-updates-write-optimization-strategy)
- [📦 Bulk Batching with Backpressure](#-bulk-batching-with-backpressure-for-high-volume-imports)
- [🔁 Retry Strategy for Failed Writes + Observability](#-retry-strategy-for-failed-writes--observability)

### Infrastructure & Deployment
- [🐳 Dockerfile Architecture (API / Worker / Web)](#-dockerfile-architecture-api--worker--web)
- [🚀 GitLab CI/CD (Dockerized Staging + Publish)](#-gitlab-cicd-dockerized-staging--publish)
- [🔔 Firebase Messaging + Edge Cache Fallback](#-firebase-messaging--edge-cache-fallback)
- [🧩 MongoDB Joins & Sharding – Strategy & Explanation](#-mongodb-joins--sharding--strategy--explanation)
- [⏱️ Worker Infrastructure & Queue Scaling Strategy](#-worker-infrastructure--queue-scaling-strategy)
- [🚀 Local → Staging → Production: Full DevOps Flow](#-local--staging--production-full-devops-flow)

### Data Modeling & Schema
- [💳 Brand Card Discount Feature](#-brand-card-discount-feature)
- [📐 MongoDB Schema Diagram](#-mongodb-schema-diagram)
- [📚 Technical Term Glossary](#-technical-term-glossary)

---

## 1. Mobile User Journey (Scan, Pay, and Go with Loyalty Integration)

The **mobile user** journey is key to ensuring a smooth and efficient checkout experience in a retail environment where speed and convenience are critical. Here's a breakdown of the journey:

- **Scan Products**: The user opens the mobile app, scans product barcodes using their phone's camera or NFC-enabled device.
    - **App feature**: Real-time product data is fetched from MongoDB and cached in Redis to ensure quick response times. The product details (e.g., price, stock level) are displayed in the app.
- **Brand Card Loyalty Integration**: The user scans their brand loyalty card, which is linked to the app. The app retrieves the loyalty program information from the backend and applies any applicable discounts.
    - **App feature**: FCM notifications alert the app when the brand card is scanned, showing loyalty points and applied discounts.
    - **Backend Integration**: Loyalty data is pulled from an integrated third-party system (e.g., an ERP or a partner service) to validate and apply discounts in real time.
- **Payment Process**: The user reviews their cart and proceeds to the checkout screen. Payments can be made through the mobile app (using **Stripe**, **PayPal**, etc.), and payment status is updated in real-time.
    - **App feature**: The app checks for inventory and product availability before proceeding to the payment step.
    - **Backend feature**: The payment gateway is integrated through a REST API, which processes payments and sends a confirmation notification to the user.
- **Checkout & Exit**: After successful payment, the user receives a digital receipt and can exit the store.
    - **App feature**: The QR code system allows the user to quickly exit without waiting in line, and the app notifies the backend to close the transaction.
    - **Backend feature**: The system updates the stock levels in MongoDB, reflecting the changes in real time, and publishes a product update event to the cache and the frontend.

## 2. Admin Dashboard (Brand Manager)

For **internal brand management** and bulk product updates, the **Admin Dashboard** allows brand managers to manage product data efficiently and in real-time. This system should be optimized for both performance and usability.

- **Bulk Product Upload**: Brand managers can upload product data in bulk through the admin dashboard, either through CSV, JSON, or ERP API integrations. The platform supports **scheduled uploads** and **real-time updates**.
    - **App feature**: The admin dashboard uses an intuitive UI for adding, editing, and deleting products, with clear error handling and validation checks for the product data.
- **ERP Integration**: The platform integrates with the brand's ERP system, allowing for seamless syncing of product data, including stock levels, pricing, and other metadata.
    - **Backend feature**: A scheduled worker fetches data from the ERP API and updates the product catalog in MongoDB. The worker also applies price and stock updates through **batch processing** to reduce the number of requests to the database.
- **Product Management**: Brand managers can manage product details, pricing, stock, and metadata. These updates are reflected in real-time across all stores.
    - **Backend feature**: Updates are pushed to Redis, invalidating cache entries for affected products and ensuring that the data is fresh across mobile apps and in-store systems.
- **Analytics & Reporting**: The dashboard provides insights into sales trends, product availability, and loyalty program performance. It also shows error logs and transaction statuses for product updates.
    - **Backend feature**: Data analytics are pulled from MongoDB and processed in the background, and the results are displayed in real time via a reporting module in the dashboard.

## 3. API Integrations with ERP

The **API integrations** allow for seamless product and inventory management across different systems. The main aspects include:

- **ERP Sync**: Product details, pricing, and stock levels are synced from the ERP system to the RetailScan platform at regular intervals or on demand.
    - **API feature**: The platform supports various integration methods, such as REST, SOAP, or GraphQL, depending on the ERP's capabilities.
- **Data Validation**: Before updating the product catalog, data is validated against business rules (e.g., price ranges, stock constraints, etc.).
    - **Backend feature**: A data validation service is used to ensure that incoming product data is correct and doesn't conflict with existing entries.
- **Real-Time Data Updates**: Price and stock changes are pushed in real time to ensure that the app and in-store systems reflect the latest data.
    - **Event-driven architecture**: Changes are propagated through the system using Redis Pub/Sub or Firebase messaging.

---

## 🧭 General Context

- Full MongoDB-first system (PostgreSQL fully removed)
- Prices, stock, discounts update independently of product structure
- Real-time API availability check based on scan action only
- Brand Card integration for contextual discounting

## 🧩 Architecture Overview (Macro-level)

```
[Mobile/Web App] → [REST API] → [Queue (Redis)] → [Worker Pool] → [MongoDB Primary]
                                                   ↓
                                               [Redis Cache]
                                                   ↓
                                    [MongoDB Secondaries for Read-Only API]
```

This diagram illustrates the core architecture of RetailScan:

- **Mobile/Web App**: User-facing applications for scanning products and completing transactions
- **REST API**: The entry point for all client requests, designed for minimal latency
- **Queue (Redis)**: Decouples real-time API responses from background processing
- **Worker Pool**: Handles resource-intensive operations asynchronously
- **MongoDB Primary**: Handles all write operations to ensure data consistency
- **Redis Cache**: Provides ultra-fast read access to frequently accessed data
- **MongoDB Secondaries**: Handle read-heavy operations without impacting write performance

## 🔄 Dual Stream Strategy: Product vs Price/Stock Updates

**Context:**

- Product metadata (title, description, brand, barcode) rarely changes.
- Price, stock levels, and discounts change frequently, even hourly.
- MongoDB is used as the single source of truth for both.

**Strategy Overview:**

```
                [Product Import Stream]                   [Price/Stock Update Stream]
                        │                                             │
      ┌─────────────────▼─────────────────┐       ┌──────────────────▼─────────────────┐
      │ [Staging Collection: Products]    │       │     [Realtime Price/Stock Updates] │
      │ • Full product metadata           │       │     • BulkUpdate operations        │
      │ • Enriched by admin dashboard     │       │     • TTL for outdated buffers     │
      └─────────────────┬─────────────────┘       └──────────────────┬─────────────────┘
                        │                                             │
        ┌───────────────▼────────────────┐             ┌─────────────▼──────────────┐
        │  [Worker: Staging → Publish]   │             │ [Worker: Apply Realtime]   │
        └────────────────┬───────────────┘             └─────────────┬──────────────┘
                         │                                             │
                ┌────────▼────────┐                        ┌──────────▼────────────┐
                │ [MongoDB Primary]│                        │ [Redis Queue + Cache]│
                └─────────────────┘                        └───────────────────────┘
```

**Benefits:**

- Avoids write amplification by isolating high-frequency updates.
- Keeps Product collection lean, with separate flows.
- Allows stock/price to be updated without touching the full product doc.
- Enables TTL for temporary buffers (e.g. price changes not yet validated).

## 🔁 Web-Queue-Worker Pattern

**Purpose:**

To decouple user requests from processing-heavy tasks while ensuring scalability, retries, and fault tolerance.

**Pattern Structure:**

```
[Client (App/Web)]
       │
       ▼
[REST API Server]
       │   → Immediately acknowledges request
       ▼
[Redis Queue (pub/sub)]
       │
       ▼
[Worker Pool (Async)]
       │
       ├─> [MongoDB Write (Primary)]
       └─> [Redis Cache + Pub/Sub Event]
```

**Core Concepts:**

- **Queue-first ingestion**: Incoming requests are enqueued instantly to ensure low latency at the API level.
- **Redis Queue**: Stores task payloads and acts as a bridge between the API and workers.
- **Worker Pool**: Pulls jobs from Redis, processes them asynchronously, and writes results to MongoDB or emits pub/sub events.
- **Eventual Consistency**: Ensures the system stays responsive even during heavy loads.

**Benefits:**

- Handles bursts of activity (e.g., flash sales).
- API remains responsive under pressure.
- Enables retries and backpressure management.
- Facilitates complex flows (multi-step imports, enrichment pipelines).

## ⚙️ Price/Stock Pipeline (Real-Time Friendly)

**Goal:**

Allow stores to update price & stock **independently of the product catalog**, without triggering full reimports.

**Pipeline Architecture:**

```
[External Inventory System]
       │
       ▼
[REST API Ingestion Endpoint]
       │ → { barcode, storeId, price, stock }
       ▼
[Redis Stream (Queue)]
       ▼
[Worker]
       ├─ Lookup `productId` via barcode/storeId
       ├─ Update price/stock in-place in MongoDB
       └─ Emit event → cache busting + sync
```

**MongoDB Update Pattern:**

```
db.products.updateOne(
  { barcode: "123", storeId: ObjectId("...") },
  {
    $set: {
      price: 899,
      stock: 12,
      updatedAt: new Date()
    }
  }
)
```

**Key Points:**

- Fast `lookup → update` operation (no schema joins needed).
- Works without touching the original product definition.
- TTL index on `updatedAt` enables optional clean-up windows.
- Real-time cache invalidation for frontends.
- Can handle **100K+ daily updates per brand** with isolated load.

## 📦 Product Import Pipeline (Staging → Publish)

**Flow Diagram:**

```
[Brand Admin Uploads File] → [Staging Collection]
                             ↓
[Worker Processes Bulk CSV / JSON]
                             ↓
[Validation Logs → Admin Dashboard]
                             ↓
[Final Approval / Scheduled Push]
                             ↓
[Published to "products" collection]
                             ↓
[Updated Products Indexed for Scan]
```

**Why this pipeline?**

| Step | Purpose |
| --- | --- |
| `Staging collection` | Avoids polluting production until data is cleaned/approved |
| `Worker async import` | Handles massive files (10k+ SKUs) without blocking |
| `Approval step` | Prevents accidental overrides or brand conflicts |
| `BulkWrite with upserts` | Scales efficiently and handles partial updates |
| `Mongo TTL on staging` | Prevents stale data buildup if admin forgets cleanup |

## 🧠 Read Optimization with Redis Pattern

### Problem

MongoDB is fast, but not fast enough for **ultra-high read frequency** like:

- Product detail pages
- Mobile scanning in aisles
- Real-time stock display

> Hitting MongoDB for each of these is overkill and risks saturation at scale.

### Solution: Redis Read Cache

**Pattern:**

```
[Client Request] → [API]
                   ├── Try Redis: GET product:{barcode}
                   ├── If hit → return
                   └── If miss:
                         → Query MongoDB
                         → Set Redis (EX = 60s)
                         → Return to client
```

**Example Code:**

```
// API route (Node.js + Redis client)
const cached = await redis.get(`product:${barcode}`);
if (cached) return JSON.parse(cached);

const product = await db.products.findOne({ barcode });
await redis.set(`product:${barcode}`, JSON.stringify(product), 'EX', 60);
return product;
```

**Bonus: Per-Field Redis Caching**

Use lightweight keys for fast refreshes:

```
await redis.set(`price:${productId}`, JSON.stringify({ price, stock }), 'EX', 30);
```

**Why This Matters**

| Concern | Solution & Reason |
| --- | --- |
| ⏱️ Ultra-fast read latency | Redis = ~1ms reads, ideal for product detail views |
| 💰 Save MongoDB ops | Avoid 100k reads/day → scales down cost |
| 🔄 Smart invalidation | TTL + manual evict via workers keeps data fresh |
| 🧩 Flexible cache granularity | Cache whole docs, sub-parts (price), or sessions |

## ⚡ Real-Time Cache Busting + Event-Driven Sync

### Context & Need

In a high-velocity environment like RetailScan (price/stock changes every few seconds per store), we **must prevent stale UI reads**. But hitting MongoDB for every client render is inefficient — we need **smart cache invalidation.**

### Strategy Overview

```
[Inventory Update] → [Worker] →
  ├── MongoDB Update
  ├── Redis Invalidation: `del product:{id}`
  └── WebSocket / Firebase PubSub Event → clients
```

### Cache Invalidation Techniques

1. **Redis as Read Cache**
    - Products are cached by `product:{barcode}` or `product:{id}`
    - Used by frontend apps and web dashboards
2. **On Update**
    - Workers detect critical writes (price, stock, discount)
    - Perform `DEL` command on the corresponding Redis key
3. **Downstream Sync**
    - Emit event via Firebase Cloud Messaging (FCM) or WebSocket to refresh views

### Example Redis Key Strategy

```
product:barcode:123456 → { ...productDoc }
price:store:789:barcode:123456 → { stock, price, updatedAt }
session:user:456 → { ...activeSession }
```

### Why this matters

| Reason | Explanation |
| --- | --- |
| ⏱️ UI Responsiveness | Avoid slow queries under heavy usage |
| 📦 Low MongoDB Load | Most reads come from Redis (milliseconds) |
| 🔄 Precise Updates | Only evict changed entries, not full cache |
| 📲 Mobile Live Sync | FCM/WebSocket event tells app to reload product |

## 💳 Brand Card Discount Feature

```
[User Scans Brand Card] → [API] → [Match Brand + User] → [Attach Discount to Session]
                                                      ↓
                                              [Checkout Engine] → Apply if present
```

This diagram illustrates the brand card discount flow:

1. User scans a loyalty/brand card in the app
2. API validates and identifies the brand and user
3. System attaches applicable discounts to the active user session
4. During checkout, the engine automatically applies the discount

## ⏱️ TTL Strategy, Read Preferences & Mongo-Specific Optimizations

- **TTL for ephemeral session/discount data**: MongoDB automatically removes expired documents using TTL (Time-To-Live) indexes
- **`readPreference: 'secondaryPreferred'` for scale**: Routes read operations to secondary nodes when available
- **`readConcern: 'majority'` for critical ops**: Ensures data consistency for operations like checkout by reading from a majority of replica set members
- **Optimized writes via `$set` and `bulkWrite`**: Updates only necessary fields and batches operations for efficiency
- **Compound indexes, partial indexes, query monitoring**: Ensures optimal query performance
- **MongoDB tools**: `.explain()` for query analysis, Application Performance Monitoring (APM), and TTL reindexing for maintenance

## 📐 MongoDB Schema Diagram

```
Brands
├── _id (ObjectId)
├── name (String)
├── theme (Object)
├── settings (Object)
├── features (Array)
└── modules (Array of ObjectIds → Modules)

Modules
├── _id (ObjectId)
├── name (String)
├── version (String)
├── config (Object)
├── status (String)
└── permissions (Object)

Products
├── _id (ObjectId)
├── barcode (String, Indexed)
├── brandId (ObjectId)
├── storeId (ObjectId)
├── title (String)
├── price (Number)
├── stock (Number)
├── metadata (Object)
└── updatedAt (ISODate, TTL-capable)

DiscountCards
├── _id (ObjectId)
├── userId (ObjectId)
├── brandId (ObjectId)
├── scannedAt (ISODate)
└── sessionId (String)

Sessions
├── _id (ObjectId)
├── userId (ObjectId)
├── discountCardId (ObjectId, optional)
├── createdAt (ISODate)
└── expiresAt (TTL Index)
```

This schema diagram illustrates the core collections in RetailScan's MongoDB database, their key fields, and relationships between collections.

## 🧩 MongoDB Joins & Sharding – Strategy & Explanation

### 1. **Joins in MongoDB (Manual / Lookup-based)**

MongoDB is **non-relational**, so it doesn't support SQL-style joins natively. However, you can **simulate joins** using `$lookup` in aggregation pipelines.

Example — join a product with its brand:

```
db.products.aggregate([
  {
    $lookup: {
      from: "brands",
      localField: "brandId",
      foreignField: "_id",
      as: "brand"
    }
  }
])
```

### In RetailScan:

- We **minimize the use of `$lookup`** in real-time APIs.
- Joins are **pre-computed** or denormalized into product documents where needed.
- Heavy aggregations (like dashboard stats) use **async workers** + caching.

### 2. **Sharding in MongoDB**

**Sharding** = splitting your database horizontally across shards (i.e., multiple MongoDB nodes).

Each shard holds a **subset of your data**, based on a **shard key**.

Example:

```
shardKey: { brandId: 1 }
```

### Why this matters:

- Prevents single-node overload
- Enables horizontal scaling
- Keeps write throughput high under load

### Pitfalls:

- **Bad shard keys** = unbalanced traffic
- You can't change shard key once set
- `$lookup` across shards can be expensive (scatter-gather)

### RetailScan Strategy:

| Concern | Solution |
| --- | --- |
| High write volume for prices/stocks | Shard by `brandId` and `storeId` — ensures updates are local to a shard |
| Admin dashboard joins (e.g. product + brand) | Use denormalization + nightly pipelines for analytics |
| Discount session resolution (brand + user) | Fast index lookup, no `$lookup` needed |
| Worker-side joins | Pre-fetched metadata or async enrichment |
| Product detail resolution | Served from read-optimized secondaries or Redis |

## 🔄 Application vs Database-Level Updates (Write Optimization Strategy)

### Core Problem

RetailScan handles **tens of thousands of write operations per brand per day**, especially for **price and stock updates**. Unoptimized writes:

- overload MongoDB under burst traffic,
- generate noisy logs,
- create race conditions or stale cache bugs,
- and reduce clarity during debugging.

### 1. Application-Level Diffing (TypeScript)

**What it does:**
Before updating the DB, fetch the existing document and only write changed fields.

```
const existing = await db.products.findOne({ barcode, storeId });
if (!existing) return;

const updates: any = {};
if (existing.price !== newPrice) updates.price = newPrice;
if (existing.stock !== newStock) updates.stock = newStock;

if (Object.keys(updates).length) {
  await db.products.updateOne(
    { barcode, storeId },
    { $set: { ...updates, updatedAt: new Date() } }
  );
}
```

**Benefits**

- Avoids unnecessary writes
- Precise logs: only real changes tracked
- Enables full diff history (for auditing)

**Drawbacks**

- **2 DB calls per write** → high I/O pressure under scale
- Adds TypeScript-side complexity
- Doesn't scale well for batch or real-time imports

### 2. Database-Level Direct Writes

**What it does:**
Always send an update, and let MongoDB internally optimize unchanged fields.

```
await db.products.updateOne(
  { barcode, storeId },
  {
    $set: {
      price: newPrice,
      stock: newStock,
      updatedAt: new Date()
    }
  }
);
```

**Benefits**

- Single DB call
- Works great with `bulkWrite()` batching
- Leverages MongoDB's write optimization
- Lower latency, ideal for high-throughput flows

**Drawbacks**

- Updates even if no actual change (but Mongo optimizes)
- No automatic diff tracking unless built manually

### Recommended Hybrid Strategy (RetailScan)

| Layer | Strategy | Why |
| --- | --- | --- |
| 🔧 Admin dashboard | App-level diffing | Precise audits + few ops |
| ⚙️ Worker imports | DB-level `$set` updates | High throughput & simplicity |
| 📱 Mobile/API | Direct writes + TTL | Real-time, cache-aware flows |

**Optional Enhancement**:

When needed, we can use a utility function like:

```
function buildMongoUpdateDiff(original, incoming) {
  const diff = {};
  for (const key in incoming) {
    if (incoming[key] !== original[key]) {
      diff[key] = incoming[key];
    }
  }
  return diff;
}
```

### Summary Table

| Strategy | Best For | Scalable? | Granular Diffs? | Performance |
| --- | --- | --- | --- | --- |
| App-level diffing | Admin & Analytics | ❌ | ✅ | 🐢 Slower |
| DB-level direct write | Workers & APIs | ✅ | ❌ | ⚡ Fast |
| Hybrid | RetailScan architecture | ✅ | ✅/❌ | ⚖ Balanced |

This strategy **ensures fast writes at scale**, without sacrificing debugging visibility where it matters (admin, audit, QA).

## 📦 Bulk Batching with Backpressure (for High-Volume Imports)

### Why it Matters

During mass product imports (10K+ SKUs per file), naive approaches overload:

- **MongoDB write capacity** (too many simultaneous writes)
- **Redis queues** (growing uncontrollably)
- **Memory** (worker RAM exhaustion if jobs are too large)

### Pattern: Chunked Processing + Concurrency Control

**1. Split import file into safe batches**

```
const BATCH_SIZE = 500;
const chunks = _.chunk(importedProducts, BATCH_SIZE);
```

**2. Use `Promise.allSettled()` with limited parallelism**

```
const limit = pLimit(5); // e.g., 5 workers max at once

await Promise.allSettled(
  chunks.map(chunk => limit(() => processBatch(chunk)))
);
```

**3. Inside `processBatch`**

```
await db.products.bulkWrite(
  chunk.map(p => ({
    updateOne: {
      filter: { barcode: p.barcode, storeId: p.storeId },
      update: { $set: { ...p, updatedAt: new Date() } },
      upsert: true
    }
  }))
);
```

### Why this works

| 🔥 Problem | ✅ Solved By |
| --- | --- |
| Write overload | Controlled concurrency (e.g., `pLimit`) |
| RAM exhaustion | Small fixed-size batches |
| Import timeouts | `Promise.allSettled()` = no full fail |
| Retry/failure handling | Track failed batches for reprocessing |

## 🔄 Read Synchronization & Consistency (Live UI + API)

### Problem

Even if writes scale, **reads may serve stale data** if:

- Cache is not invalidated
- Client sync is not triggered
- Secondary reads lag behind primaries

### Strategy Summary

| Layer | Consistency Tool | Description |
| --- | --- | --- |
| MongoDB | `readConcern: majority` | For checkout/critical reads |
| MongoDB Replica | `readPreference: secondaryPreferred` | For price/stock UI lookups |
| Redis | TTL + `DEL` strategy | On write, we purge affected cache keys |
| Clients | FCM/WebSocket refresh | Notified to re-fetch on update event |

### Example

```
// Product scan → Redis cache
const cached = await redis.get(`product:${barcode}`);
if (cached) return JSON.parse(cached);

// Fallback to DB
const product = await db.products.findOne({ barcode, storeId });

// Cache for next read
await redis.set(`product:${barcode}`, JSON.stringify(product), "EX", 300);
```

When a **worker updates stock**, it:

```
await redis.del(`product:${barcode}`);
fcm.send({ ... }); // notify app to refetch
```

This dual system ensures:

- Near-instant product UI updates
- Fast reads from Redis
- Live user feedback
- Reduced MongoDB read pressure

## 🔁 Retry Strategy for Failed Writes + Observability

### Why It Matters

- High-volume imports, price updates, or discount creation may fail due to:
    - **Write conflicts**
    - **Mongo overload (timeouts, write concern fails)**
    - **Validation mismatches**
    - **Transient network issues**

You don't want your system to silently fail or retry too aggressively.

### Retry Logic Pattern (Node.js / TypeScript)

```
async function safeWriteWithRetry(batch, retries = 3) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      await db.products.bulkWrite(batch, { ordered: false });
      return; // Success!
    } catch (err) {
      if (attempt === retries) throw err;
      console.warn(`Retry #${attempt} due to`, err.message);
      await delay(100 * attempt); // Exponential backoff
    }
  }
}
```

> `ordered: false` allows partial success — don't fail whole batch if one item fails.

### Failure Logging (Optional but powerful)

```
await db.import_failures.insertOne({
  batchId,
  reason: err.message,
  failedItems: batch,
  createdAt: new Date()
});
```

Later, these can be replayed or visualized in admin dashboards for follow-up.

## 📊 Monitoring Worker Health Under Load

### What to Monitor

| Metric | Why it matters |
| --- | --- |
| Queue length (Redis key size) | Backlog of jobs = scaling signal |
| Job duration (avg per batch) | Identify bottlenecks |
| Job success/failure ratio | Alert on spikes in failure rate |
| Memory usage per worker | Prevent OOM crashes |
| Active worker count | Are all pods/nodes alive? |

### Example: Exposing Worker Health (Node.js)

```
import express from "express";
const app = express();

app.get("/health", async (req, res) => {
  const queueSize = await redis.llen("import:queue");
  const memory = process.memoryUsage();

  res.json({
    queueSize,
    memory,
    timestamp: new Date()
  });
});
```

This endpoint can be scraped by **Prometheus**, **GCP Ops Agent**, or even polled by your dashboard.

### Auto-scaling Hooks (Cloud Run / Kubernetes)

- Set autoscaler to scale workers **based on Redis queue depth**
- Example: 1 worker per 500 pending jobs
- On GCP Cloud Run:

```bash
--max-instances=20
--concurrency=1
```

- On K8s:

```yaml
autoscaling:
  minReplicas: 2
  maxReplicas: 50
  targetCPUUtilizationPercentage: 70
```

### Summary

| Topic | Technique |
| --- | --- |
| Retry failed ops | `ordered: false`, exponential backoff |
| Failure tracking | Log to `import_failures` collection |
| Health visibility | `/health` endpoint with queue/memory |
| Autoscaling | Based on Redis queue or CPU/memory usage |

## 🧑‍💼 Admin Dashboards + Observability for Retry Outcomes

### Why It's Important

When scaling data ingestion or managing brand updates, admins must:

- **See what failed** (e.g. CSV rows, invalid products)
- **Understand why** (e.g. validation issue, stock missing)
- **Retry safely** after fixes

This avoids support tickets, guessing, and silent data loss.

### Admin Dashboard Integration

> All failed operations are stored in a MongoDB collection (import_failures).

Each doc includes:

```
{
  batchId: "batch-2024-001",
  reason: "Missing barcode",
  failedItems: [ /* partial product objects */ ],
  createdAt: ISODate,
  resolved: false
}
```

### Example Dashboard UI Features

| Feature | Purpose |
| --- | --- |
| Failed batches list | See which imports failed |
| CSV export of failed rows | Easy edit/fix |
| Retry button (per batch/item) | Trigger re-processing |
| Inline JSON validation errors | Help developer/admin debug fast |
| Status: Pending / Failed / Retried | Track import lifecycle |

### Retry Workflow in UI

```
[Admin clicks "Retry"]
      ↓
[API PUT /retries/:batchId]
      ↓
[Requeue failedItems into Redis]
      ↓
[Worker consumes and logs outcome]
      ↓
[Updates `import_failures.resolved = true` if successful]
```

## ⏱️ Session Lifecycle + TTL Management

### Why Sessions Exist

- Used to track user scanning & discount behavior
- Discount cards are **temporary** (valid X mins after scan)
- Session is invalidated on checkout or timeout

### Collections Involved

| Collection | TTL Index Field | Purpose |
| --- | --- | --- |
| `sessions` | `expiresAt` | Cleanup old checkout sessions |
| `discountcards` | `scannedAt` | Remove expired discount links |

```
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });
db.discountcards.createIndex({ scannedAt: 1 }, { expireAfterSeconds: 600 });
```

### Session Expiry Logic in Workers

```
// Worker fetch
const session = await db.sessions.findOne({ _id: sid });
if (!session || new Date() > session.expiresAt) {
  return { error: "Session expired" };
}
```

Workers **respect TTL cleanup**, ensuring:

- No checkout with stale data
- No use of expired brand discounts

### Combined Benefits

| Problem | Feature |
| --- | --- |
| Admin unsure why import failed | Dashboard with failure logs |
| Massive burst → partial writes | Retry + per-item status |
| Stale sessions ≠ valid cart | TTL + worker cleanup check |
| Brand cards used twice | TTL expiration after scan |
| Debugging async jobs | `/health`, logs, and dashboard UI |

## 🔔 Firebase Messaging + Edge Cache Fallback

### Purpose

To **instantly notify** apps (mobile/web) of relevant updates like:

- Product price/stock changes
- Discount card scan acknowledgment
- Session expiration or checkout window closing

### Firebase Cloud Messaging (FCM) Setup

```
// Backend Worker (Node.js)
import { getMessaging } from "firebase-admin/messaging";

const fcm = getMessaging();

await fcm.send({
  token: userDeviceToken,
  notification: {
    title: "Product Update",
    body: "New price or stock for item in your cart",
  },
  data: {
    type: "product-update",
    productId: "123456",
  },
});
```

### Mobile/Web Client Listener

```
// React Native or PWA
import messaging from '@react-native-firebase/messaging';

messaging().onMessage(remoteMessage => {
  if (remoteMessage.data?.type === "product-update") {
    refreshProduct(remoteMessage.data.productId);
  }
});
```

### CDN + Edge Cache Strategy

```
[User Request]
    ↓
[Cloud CDN or Edge Cache]
    ├── Serve cached version if valid
    └── On invalidation: revalidate from API or Redis

Invalidation Triggered By:
- FCM/WebSocket event
- Redis key eviction
- TTL expiry or stale-check
```

### Why this matters

| Goal | How it's achieved |
| --- | --- |
| 🔔 Instant feedback | FCM/WebSocket notifies UI of live changes |
| 🧠 Smart caching | Avoids full-page refreshes or heavy polling |
| 💸 Cost reduction | Reduces reads from primary MongoDB |
| 🔄 Edge performance | CDN ensures quick load even under global scale |

## 🐳 Dockerfile Architecture (API / Worker / Web)

### Dockerfiles + Scaling Rationale

```
[Dockerfile.api]
├─ Base: node:20-slim
├─ COPY: /services/api
├─ EXPOSE: 3000
└─ CMD: "node dist/main.js"
```

**Why?**

- API must scale horizontally to serve parallel user scans and checkout flows.
- Container size matters: `node:20-slim` keeps builds small & faster for CI/CD.
- Only exposes API layer; no worker logic or analytics overhead.

```
[Dockerfile.worker]
├─ Base: node:20-slim
├─ COPY: /services/worker
├─ CMD: "node dist/worker.js"
```

**Why?**

- Handles bulk write tasks (e.g., product import, discount application).
- Decouples business logic from the main API.
- Can scale **independently** based on event queue length (i.e., burst resilience).

```
[Dockerfile.web]
├─ Base: node:20-slim
├─ COPY: /services/web
├─ RUN: next build
├─ EXPOSE: 3000
└─ CMD: "next start"
```

**Why?**

- Next.js builds can be heavy → isolate from API/Worker for faster boot times.
- Only deployed where web dashboard is needed (e.g., admin/staff users).
- Avoids interfering with scan/checkout flow scaling.

## 🚀 GitLab CI/CD (Dockerized Staging + Publish)

### GitLab CI/CD Flow Diagram

```
[dev push] → [build stage]
           ├─ docker build (api)
           ├─ docker build (worker)
           ├─ docker build (web)
           ↓
         [push stage]
           ├─ docker push → gcr.io/project/api
           ├─ docker push → gcr.io/project/worker
           └─ docker push → gcr.io/project/web
           ↓
         [deploy stage]
           ├─ gcloud run deploy api
           ├─ gcloud run deploy worker
           └─ gcloud run deploy web
```

**Why this structure?**

| Stage | Scaling Purpose |
| --- | --- |
| `build` | Separate builds ensure cache optimization for each service |
| `push` | Pushes immutable versioned images (ready for rollback) |
| `deploy` | Allows **partial rollout** (e.g., only worker if price engine updates) |
| Cloud Run | Lets you scale **to 0** (no traffic = no cost), and **burst auto-scale** fast |

### `.gitlab-ci.yml` Example

```yaml
stages:
  - build
  - push
  - deploy

variables:
  PROJECT_ID: my-retailscan-project
  REGION: europe-west1
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

before_script:
  - echo $GCP_SERVICE_ACCOUNT_KEY | base64 -d > gcp-key.json
  - gcloud auth activate-service-account --key-file=gcp-key.json
  - gcloud config set project $PROJECT_ID
  - gcloud auth configure-docker europe-west1-docker.pkg.dev

build_api:
  stage: build
  script:
    - docker build -t api:$IMAGE_TAG ./services/api

build_worker:
  stage: build
  script:
    - docker build -t worker:$IMAGE_TAG ./services/worker

build_web:
  stage: build
  script:
    - docker build -t web:$IMAGE_TAG ./services/web

push_api:
  stage: push
  script:
    - docker tag api:$IMAGE_TAG europe-west1-docker.pkg.dev/$PROJECT_ID/api:$IMAGE_TAG
    - docker push europe-west1-docker.pkg.dev/$PROJECT_ID/api:$IMAGE_TAG

push_worker:
  stage: push
  script:
    - docker tag worker:$IMAGE_TAG europe-west1-docker.pkg.dev/$PROJECT_ID/worker:$IMAGE_TAG
    - docker push europe-west1-docker.pkg.dev/$PROJECT_ID/worker:$IMAGE_TAG

push_web:
  stage: push
  script:
    - docker tag web:$IMAGE_TAG europe-west1-docker.pkg.dev/$PROJECT_ID/web:$IMAGE_TAG
    - docker push europe-west1-docker.pkg.dev/$PROJECT_ID/web:$IMAGE_TAG

deploy_api:
  stage: deploy
  script:
    - gcloud run deploy api \
        --image europe-west1-docker.pkg.dev/$PROJECT_ID/api:$IMAGE_TAG \
        --region $REGION \
        --platform managed \
        --allow-unauthenticated

deploy_worker:
  stage: deploy
  script:
    - gcloud run deploy worker \
        --image europe-west1-docker.pkg.dev/$PROJECT_ID/worker:$IMAGE_TAG \
        --region $REGION \
        --platform managed \
        --allow-unauthenticated

deploy_web:
  stage: deploy
  script:
    - gcloud run deploy web \
        --image europe-west1-docker.pkg.dev/$PROJECT_ID/web:$IMAGE_TAG \
        --region $REGION \
        --platform managed \
        --allow-unauthenticated
```

## ⏱️ Worker Infrastructure & Queue Scaling Strategy

### Why Workers?

Workers decouple **real-time requests** (user scans, admin actions) from **heavier background operations** (imports, discount logic, stock sync).

They process:

- Product imports
- Price/stock updates
- Discount session validation
- Brand rule evaluation

### Architecture Overview

```
[API Server]
   │
   ├── Enqueue → Redis Queue
   ▼
[Worker Pool]
   ├── MongoDB Writes
   ├── Redis Invalidation
   └── Event Emit (WebSocket / FCM)

Each Worker:
- Listens to specific queues (e.g., "import", "price-updates")
- Processes tasks asynchronously
- Supports retries + dead-letter queue fallback
```

### Retry & Dead Letter Queue Strategy

- Retry `N` times with exponential backoff
- After max retries → push to `dead-letter` queue
- Alert admin if failure rate exceeds threshold

### Scaling Strategy

| Load Type | Scaling Decision |
| --- | --- |
| Price/stock updates | Scale workers **horizontally** by queue length |
| Product import | Parallelize across batches (bulkWrite + upsert) |
| Brand discount logic | Use lightweight flows with session TTL |
| Spikes (e.g. Black Friday) | Auto-scale queue consumers via Cloud Run concurrency |

### Worker Types & Roles

| Worker Name | Purpose |
| --- | --- |
| `price-sync` | Real-time price/stock update |
| `import-worker` | Batch upload → publish |
| `discount-check` | Validate scanned discount cards |
| `rule-eval` | Enforce dynamic brand/store rules |

This structure supports **millions of daily ops** with distributed load and isolated failure domains.

## 🚀 Local → Staging → Production: Full DevOps Flow

### 1. Local Dev Environment

#### Stack

| Component | Tech |
| --- | --- |
| API | Node.js + TypeScript + Express |
| Worker | Node.js + TypeScript (queue processor) |
| Web Dashboard | Next.js (React) |
| DB | MongoDB (Docker, with TTL config) |
| Queue/Cache | Redis (Docker) |

#### `docker-compose.dev.yml`

```yaml
version: "3.9"
services:
  api:
    build: ./services/api
    ports:
      - "3000:3000"
    env_file: .env
    depends_on: [mongodb, redis]

  worker:
    build: ./services/worker
    env_file: .env
    depends_on: [mongodb, redis]

  web:
    build: ./services/web
    ports:
      - "3001:3000"
    env_file: .env

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    command: mongod --replSet rs0
    volumes:
      - ./mongo-init:/docker-entrypoint-initdb.d

  redis:
    image: redis:7
    ports:
      - "6379:6379"
```

> init.js example in ./mongo-init/ sets up TTL indexes and replica sets for local simulation.

### MongoDB TTL in Local Docker (dev/test)

```yaml
# docker-compose.override.yml
version: '3.9'
services:
  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    command: mongod --replSet rs0
    environment:
      - MONGO_INITDB_ROOT_USERNAME=root
      - MONGO_INITDB_ROOT_PASSWORD=example
    volumes:
      - ./mongo-init:/docker-entrypoint-initdb.d

# ./mongo-init/init.js
```

```javascript
// ./mongo-init/init.js

rs.initiate({
  _id: "rs0",
  members: [{ _id: 0, host: "localhost:27017" }]
});

db = db.getSiblingDB('retailscan');

// TTL for scan session expiration (e.g. 5 minutes)
db.sessions.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);

// TTL for temporary brand discount links (e.g. 10 minutes)
db.discountcards.createIndex(
  { scannedAt: 1 },
  { expireAfterSeconds: 600 }
);

// Optional: TTL for staging import data (auto-purge stale)
db.product_staging.createIndex(
  { updatedAt: 1 },
  { expireAfterSeconds: 3600 } // 1 hour
);

// Shard-friendly compound index examples (for scale)
db.products.createIndex({ barcode: 1, storeId: 1 });
db.products.createIndex({ brandId: 1 });
```

### 2. Staging Environment (GitLab CI + GCP)

#### `.gitlab-ci.yml` (Staging Target)

```yaml
stages:
  - build
  - push
  - deploy

variables:
  REGION: europe-west1
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

before_script:
  - echo $GCP_KEY | base64 -d > gcp-key.json
  - gcloud auth activate-service-account --key-file=gcp-key.json
  - gcloud config set project $CI_PROJECT_ID
  - gcloud auth configure-docker europe-west1-docker.pkg.dev
```

#### Build & Push

```yaml
build_api:
  stage: build
  script:
    - docker build -t api:$IMAGE_TAG ./services/api

push_api:
  stage: push
  script:
    - docker tag api:$IMAGE_TAG europe-west1-docker.pkg.dev/$CI_PROJECT_ID/api:$IMAGE_TAG
    - docker push europe-west1-docker.pkg.dev/$CI_PROJECT_ID/api:$IMAGE_TAG
```

#### Deploy to GCP Cloud Run (Staging)

```yaml
deploy_api:
  stage: deploy
  script:
    - gcloud run deploy api \
        --image europe-west1-docker.pkg.dev/$CI_PROJECT_ID/api:$IMAGE_TAG \
        --platform managed \
        --region $REGION \
        --allow-unauthenticated
```

> Similar steps for worker and web.

### 3. Production Deployment Strategy

#### Blue-Green with Canary Releases

| Step | Technique |
| --- | --- |
| Docker versioning | `IMAGE_TAG = release-x.x.x` |
| Canary testing | Cloud Run traffic splitting |
| Rollback strategy | Manual or auto-fallback |
| Monitoring | Stackdriver + custom events |
| Crash alert | Slack or Opsgenie integration |

#### Why This Flow?

| Goal | How It's Solved |
| --- | --- |
| ⛳ Fast feedback loop | Local → Docker → TTL/Redis tested locally |
| 🌍 Staging parity | Same Dockerfiles & infra as prod |
| 🚀 Fast deploys | Cloud Run auto-scales + GitLab CI pipeline |
| ⏱️ Auto-scaling under pressure | Cloud Run with Redis/Mongo burst tolerance |
| 🔄 Partial deployments | API/Worker/Web deploy independently |

## 🧠 Handling Price Updates from Large Brand Uploads or ERP API Integrations

### Context

Prices primarily come from **large uploads of brand data** or **ERP API integrations**. We need to process these effectively without impacting user experience.

### 1. Batch Processing for Large Price Uploads

**Approach:**

- **Staging Collection:** Store uploaded price data in a temporary collection for validation
- **Worker Pools for Asynchronous Processing:** Use workers to validate and process price updates
- **Optimized Write Operations:** Use MongoDB's `bulkWrite` to apply changes efficiently
- **Example Flow**:
  1. Brand uploads a CSV with prices
  2. Data is stored temporarily in the **staging collection**
  3. Worker processes the batch (validates prices, applies changes to MongoDB)
  4. Invalidate affected Redis cache keys
  5. Notify clients via **Firebase Messaging** (FCM) to pull fresh data

**Why This Matters:**

- **Performance**: Reduces load on production database by processing large updates in the background
- **Accuracy**: Ensures no price updates are missed or applied incorrectly
- **Scalability**: Supports high concurrency using worker pools and background tasks

### 2. ERP API Integration for Real-Time Price Sync

**Approach:**

- **API Integration Strategy:** 
  - **Polling**: Periodically query ERP API for price changes
  - **Webhooks**: Receive real-time price changes directly from ERP system
- **Process:**
  1. ERP system sends a **price update webhook** to RetailScan backend
  2. Validate the price change
  3. Use `bulkWrite` to update prices in MongoDB
  4. Update **Redis** cache
  5. Emit an event via **FCM** to notify mobile/web apps

**Real-Time Data Flow**:

```
[ERP System] → [Webhooks / API Integration] → [REST API] → [Price Update Worker] → [MongoDB Update] → [Redis Cache Invalidation] → [FCM Notification to Clients]
```

### 3. Optimized Caching for Price Data

**Approach:**

- **Use Redis as the Primary Price Cache**
  - Cache product prices with appropriate TTL
  - Use unique Redis keys for each product
- **Invalidate Cache on Price Update**
  - Immediately invalidate Redis cache entries for updated products
- **Cache Expiry Strategy**
  - Set appropriate TTL for price cache entries

**Cache Invalidation Flow**:

1. Price update occurs (via bulk upload or ERP system)
2. Backend updates **MongoDB**
3. Redis cache for the product is **invalidated** (via `DEL` command)
4. Updated price is fetched from MongoDB and cached in Redis
5. **FCM/WebSocket event** notifies clients

## 📚 Technical Term Glossary

- **TTL (Time-To-Live)**: Automatic deletion of documents after a certain period.
- **readPreference**: MongoDB option to direct read ops to secondaries.
- **readConcern**: Guarantees for read data consistency.
- **bulkWrite()**: Efficient batch operations in MongoDB.
- **compound index**: Index across multiple fields.
- **partial index**: Indexing a filtered subset of data.
- **Redis Queue**: Pub/sub queueing between API & workers.
- **Worker Pool**: Distributed async task processors.
- **Session**: Represents user's temporary interaction window.
- **DiscountCard**: Link between user + brand for a temporary discount.
- **APM**: Application Performance Monitoring.
- **MongoDB .explain()**: Returns info on query performance.
- **FCM**: Firebase Cloud Messaging, for push notifications.
- **WebSocket**: Protocol for real-time bidirectional communication.
- **Optimistic Locking**: Concurrency control method that assumes conflicts are rare.
- **Sharding**: Horizontally partitioning data across multiple servers.
- **Replica Set**: Group of MongoDB instances maintaining the same data set.
- **CI/CD**: Continuous Integration/Continuous Deployment pipeline.
- **Docker**: Containerization platform for consistent environments.
- **Kubernetes**: Container orchestration system for automated deployment and scaling.
- **Cloud Run**: Google Cloud Platform's serverless container platform.
- **Dead Letter Queue**: Storage for messages that couldn't be processed successfully.