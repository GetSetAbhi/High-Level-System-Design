# Database Scaling Guide

This guide explains how to scale a database from basic techniques like indexing to advanced methods like sharding.

`“If my system needs under 10–20k QPS, I’d start with a relational DB for strong consistency and rich queries. But if I'm crossing 100k+ QPS, especially with large-scale writes or flexible schema, I’d prefer a NoSQL system with partitioning and replication built in.”`

---

## Step 0: Vertical Scaling (Before Anything Else)

* **What:** Add more CPU, RAM, or SSD to the database server.
* **Why:** Easiest and cheapest short-term fix.
* **Limitation:** There's a hardware ceiling; doesn't scale indefinitely.

---

## Step 1: Indexing

* **What:** Add indexes to columns used in `WHERE`, `JOIN`, or `ORDER BY` clauses.
* **Why:** Significantly improves query performance.
* **When to Use:** Queries are slow, but data size is manageable.
* **Limitation:**

  * Too many indexes slow down writes.
  * Not helpful for write-heavy workloads.

---

## Step 2: Read Replicas (Read Scaling)

* **What:** Create one or more read-only replicas of the primary database.
* **Why:** Offloads read traffic from the primary node.
* **When to Use:** Read-heavy workloads such as dashboards or reporting.
* **Limitation:**

  * Writes still go to the master node.
  * No true write scalability.

---

## Step 3: Partitioning (Horizontal Partitioning Within a Single DB)

* **What:** Split large tables into smaller logical partitions.
* **Why:** Improves performance by reducing the data scanned during queries.
* **Types of Partitioning:**

  * **Range Partitioning:** Based on ranges of values (e.g., dates).
  * **List Partitioning:** Based on a predefined list (e.g., region).
  * **Hash Partitioning:** Based on hashing a key (e.g., `user_id % 4`).
* **When to Use:** Single table becomes too large for efficient querying.
* **Limitation:** Still limited to a single server.

---

## Step 4: Sharding (True Horizontal Scaling)

* **What:** Split the database across multiple physical servers (shards).
* **Why:** Enables scaling of storage and write throughput.
* **How It Works:**

  * Choose a **shard key** (e.g., `customer_id`, `tenant_id`).
  * Route queries based on shard key to the appropriate database instance.
* **When to Use:**

  * When the dataset is too large for a single machine.
  * Write throughput exceeds a single node's capability.
* **Limitation:**

  * Cross-shard queries are complex.
  * Difficult to rebalance shards.
  * Joins across shards are challenging.

---

## Summary Table

| Step          | Solves For                 | Pros                         | Cons                            |
| ------------- | -------------------------- | ---------------------------- | ------------------------------- |
| Indexing      | Query latency (reads)      | Fast reads                   | Slower writes, limited scale    |
| Read Replicas | Read-heavy traffic         | Scalable reads               | Writes still bottlenecked       |
| Partitioning  | Large single table         | Targeted I/O, faster queries | Still on one machine            |
| Sharding      | Data/write volume too high | True horizontal scaling      | Complex routing and maintenance |

---

## Final Notes

* Always measure before and after each change.
* Scaling isn't just about performance—it’s also about maintainability, cost, and operational complexity.
* Choose the right tool based on your workload and growth expectations.

---

For real-world examples (e.g., scaling an e-commerce DB), feel free to reach out or extend this guide.

# ⚡ Generalized System Performance Metrics (Throughput Only)

This table provides approximate **read and write throughput** for popular systems, useful for quick reference during system design interviews. All values are **ops/sec** (operations per second).

---

## 📊 Throughput Summary

| System         | Read Throughput (ops/sec)  | Write Throughput (ops/sec)   | Notes                                  |
|----------------|----------------------------|------------------------------|----------------------------------------|
| **Redis**      | ~1 Mil                     | ~800k                        | In-memory store with extremely high throughput |
| **MySQL**      | ~100k – 300k               | ~50k – 150k                  | ACID-compliant; great for OLTP workloads |
| **MongoDB**    | ~50k – 150k                | ~30k – 100k                  | Flexible schema, moderate write speed  |
| **Cassandra**  | ~50k – 100k                | ~40k – 90k                   | Tuned for distributed, write-heavy workloads |
| **Elasticsearch** | ~10,000 – 100,000       | ~5,000 – 50,000              | Best suited for search and analytics   |

---

> These values are ballpark figures based on typical SSD-backed, modern hardware with good configurations. Use them to reason about scalability and system choice in interviews.
