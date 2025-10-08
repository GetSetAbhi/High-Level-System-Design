# Time based leaderboard

<p align="center">
  <img src="leaderboard.png" alt="Collaborative Document Editing"/>
</p>

Suppose I'm creating a service which will show me top videos watched in last K minutes, K hours, H days.

Following example elaborates upon how this can be done.

https://www.hellointerview.com/learn/system-design/problem-breakdowns/top-k

## Creating time based buckets

Every time someone watches a video, we get a view event like the following and we get **millions of these events**.

```
video_id = "A1"
timestamp = 2025-08-08 10:35
```

For each time range we want something like this

```
Last 1 hour

A1 → 2000 views
B5 → 1800 views
C9 → 1700 views

Last 1 day

B5 → 90,000 views
A1 → 85,000 views
```

Instead of storing one giant number for “last 1 hour,”
we store smaller buckets of counts, for example 1 bucket per minute.

We can have time buckets like this `Map<TimeStamp, Bucket> buckets`

And Each bucket looks like this.

```
class Bucket {
	HashMap<videoId, count> for quick counting.

	Min-Heap (priority queue) of size K to track the top videos.
}
```

Why both?

HashMap → O(1) updates to counts when a new watch comes in.
Heap → O(log K) to maintain the Top-K list.

Example: for 10:00, 10:01, 10:02 …

```
bucket[10:00]["A1"] = 35
bucket[10:00]["B5"] = 40
...
bucket[10:59]["A1"] = 27
```
And then we drop the older buckets

“Last 1 hour” → keep last 60 buckets only.

“Last 1 day” → keep last 1,440 buckets (24×60).

“Last 1 week” → keep last 10,080 buckets (7×24×60).

Old buckets get deleted automatically — so your counts stay fresh.

Now to query top K.
For last 1 minute -> just return heap from the current minute bucket
For last 1 hour → just return the heap from the current hour bucket.
For last 1 day → merge heaps from the last 24 buckets into a new heap, then take top K.
For last 1 week → same merging logic but with 7 days of buckets.

Note that the heap will not be bounded to K elements, whenever a new event comes up it will be added as a new object into the heap.
Now when we need the top k video ids, we will retrieve elements from the priority queue and discard video ids whose count in priority queue does not
match the accurate count in map. This is how we are doing deduplication.

```
Event e = new Event(videoId, viewCount);
PriorityQueue<Event> pq;
pq.push(e)
```

```
            Incoming Video Watch Events
                     |
             ┌────────────────┐
             |  Event Router   |
             └────────────────┘
                     |
       ┌─────────────┼───────────────┐
       |             |               |
  1-Hour Tracker  1-Day Tracker  1-Week Tracker
       |             |               |
   ┌───┴───┐     ┌───┴───┐       ┌───┴───┐
   |60 min |     |24 hour|       |7 day  |
   |buckets|     |buckets|       |buckets|
   └───┬───┘     └───┬───┘       └───┬───┘
       |             |               |
       ▼             ▼               ▼
  ┌──────────┐  ┌──────────┐   ┌──────────┐
  | Map:     |  | Map:     |   | Map:     |
  | videoId→ |  | videoId→ |   | videoId→ |
  | viewCnt  |  | viewCnt  |   | viewCnt  |
  └────┬─────┘  └────┬─────┘   └────┬─────┘
       |             |               |
  ┌────▼─────┐  ┌────▼─────┐   ┌────▼─────┐
  | Min-Heap |  | Min-Heap |   | Min-Heap |
  |  (Top K) |  |  (Top K) |   |  (Top K) |
  └──────────┘  └──────────┘   └──────────┘

When querying:
   - "Top-K in last 1 hour" → merge Top-K from 60 minute buckets
   - "Top-K in last 1 day" → merge Top-K from 24 hourly buckets
   - "Top-K in last 1 week" → merge Top-K from 7 daily buckets

```

✅ At YouTube scale — this approach is viable if:

Buckets are stored in-memory (Redis, RocksDB, or Java heap if small enough).

K is small (50–500).

Expiration is done lazily or on a schedule.

If you tried to calculate “Top-K” by running a SQL GROUP BY over all events in a time range each time, it would be thousands of times slower.