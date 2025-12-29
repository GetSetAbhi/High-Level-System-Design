# Common Points for System Design
---

* **Push vs Pull** - In Designing Message Queue or Metrics and Monitoring System, there is a common point of Push and Pull
	* In Message Queue, the consumer can either pull messages from broker or Broker can push messages to Consumers
	* Similarly in Metrics and Monitoring System, either the Metrics Collection Workers can pull data from the services or Collection Agent that runs
	on services will push data to the Collection Workers

* **Video Transcoding and Adaptive Bit Rate Streaming** - This is applicable in both the design youtube and design a live streaming service (Twitch)

# Capacity estimates


Assume 10M DAU, 10 actions/day → 100M requests/day → ~1K QPS average, ~5K peak. At ~100 GB/day append-only data, this fits a sharded relational DB or write-optimized NoSQL with caching. The key challenge is throughput, not storage.

1kb per event is the safe size for payload one can assume. 1kb per event combined with 1000 QPS translates to 1MB per second which then becomes 100GB per day assuming 100000 seconds per day

Peak QPS = 2-5 x QPS calculated

By pareto principle assume that 20% of the QPS are WRITES and 80% of QPS are reads

“If write QPS grows beyond ~10–20K/sec or data becomes multi-TB/day, I’d consider sharding or a NoSQL store.”

We target 99.99% availability, which allows about 50 minutes of downtime per year.


```
--------------------------------------------------------------------------------------------
| DAU Range     | QPS       |    What it implies       | Examples                          |
| ------------- |-----------|--------------------------| ----------------------------------|
| **100K – 1M** |           | Small/medium system      | Internal tools, B2B apps          |
| **~10M**      | 1000 QPS  | Large distributed system | Twitter-like, payments, messaging |
| **100M+**     | 10k QPS   | Internet-scale           | Facebook, WhatsApp, Google        |
--------------------------------------------------------------------------------------------

--------------------------------------------------------
| Peak QPS | Data growth   | Typical DB choice         |
| -------- | ------------- | ------------------------- |
| <1K      | <10 GB/day    | Single Postgres/MySQL     |
| 1–10K    | 10–100 GB/day | RDBMS + replicas          |
| 10–50K   | 100+ GB/day   | Sharded RDBMS / NoSQL     |
| 50K+     | TB/day        | Cassandra / DynamoDB / S3 |
--------------------------------------------------------

------------------------------
| Media Type | Safe Avg Size |
| ---------- | ------------- |
| Image      | **~300 KB**   |
| Video      | **~20 MB**    |
| Audio      | **~5 MB**     |
| Metadata   | **~1 KB**     |
------------------------------

----------------------------------------------------------------------------
| Availability           | Downtime / year | What it means practically     |
| ---------------------- | --------------- | ----------------------------- |
| **99%**                | ~3.65 days      | Frequent outages              |
| **99.9% (3 nines)**    | ~8.7 hours      | Acceptable for internal tools |
| **99.99% (4 nines)**   | ~52 minutes     | Good production service       |
| **99.999% (5 nines)**  | ~5 minutes      | Very hard, expensive          |
| **99.9999% (6 nines)** | ~30 seconds     | Extremely rare, elite systems |
----------------------------------------------------------------------------
```