# Monitoring and Observability


## Monitoring

We monitor our services in a way that potential issues are detected before they impact customers. 
Our monitoring system raises alerts based on performance metrics when they deviate from expected thresholds.

**What metrics can be tracked ?**

On a very basic level, 4 metrics can be tracked 

* Latency – Time taken to service a request (includes successful and failed requests).
* Traffic – How much demand your system is receiving (e.g., requests/sec, messages/sec).
* Errors – Rate of failed requests (e.g., 5xx responses, timeouts, failed database calls).
* Saturation – How “full” your system is (e.g., CPU, memory, queue length, disk usage).

⚡ 1. Latency
Definition: Time taken to process a request or operation.

Example: E-commerce Checkout
A user clicks “Place Order”.

Normally takes 200ms.

Suddenly it takes 3s.

✅ What this tells you: Something is slowing down — maybe a downstream payment service is lagging.

📌 Latency = How fast are we serving?

📊 2. Traffic
Definition: Volume of requests or load on your system.

Example: News App During Breaking News
Normal: 5k requests/minute.

Breaking news: 200k requests/minute.

✅ What this tells you: Demand is surging. You might need to autoscale.

📌 Traffic = How much are we serving?

Could be measured in:

HTTP requests/sec

Kafka messages/sec

DB read/write throughput

CPU/memory usage (as a proxy)

❌ 3. Errors
Definition: Rate of failed or invalid requests.

Example: User Login Service
Normally 0.1% error rate.

Suddenly jumps to 5% — due to expired OAuth token or DB failure.

✅ What this tells you: Something’s broken. Even if latency is okay, bad responses are going out.

📌 Errors = Are we serving correctly?

Measured by:

HTTP 5xx or 4xx

Exception counts

Retry/failure rates

Business failures (e.g., "payment declined")

🧯 4. Saturation
Definition: How close your system is to its capacity limit.

Example: Kafka Queue
Each topic partition can buffer 1,000 messages.

Producer is writing faster than consumer can read.

Lag increases → saturation is rising.

✅ What this tells you: You're about to hit a ceiling; scale up or tune performance.

📌 Saturation = Can we keep serving?




**How Tracking saturation helps**

Saturation measures how close a resource is to its limit—essentially, it tells you if you're about to overload something.

🛠 Example Scenario: Web Service with Thread Pool
Imagine you run a backend service that handles incoming API requests using a thread pool of 50 threads.

🟢 Normal Case:
20 threads in use.

30 threads free.

Queue is empty.

Saturation is low.

🔴 Problem Case:

50 threads all busy.

20 new incoming requests waiting in the queue.

Some requests start timing out.

Saturation is high! → This tells you that your service is overwhelmed.

### Example

Let’s say latency is rising and users are complaining. Your logs show nothing, error rate is low, CPU is 50%. You're confused.
Then you check saturation and see the request queue is filling up — a clear sign your app can't keep up with demand.
This metric often catches performance bottlenecks before they become full outages.

---