# Safest Non Functional Requirements

* **Scalability**

  The system should handle growth in users, traffic, and data.

* **Availability**

  The system should be accessible when users need it.

* **Reliability**

  The system should behave correctly and consistently.

* **Performance / Latency**


A system can be 

* Highly available but not reliable

  👉 Frequently crashes but restarts quickly


* Reliable but not highly available

  👉 Rarely fails but takes long to recover

# Reliability

Reliability in system design means building a system that keeps working correctly over time—even when parts fail, traffic spikes, or unexpected situations occur. A reliable system delivers consistent, correct results and minimizes downtime.

At its core, reliability is about:

* Fault tolerance (surviving failures)
* Redundancy (having backups)
* Graceful degradation (partial functionality instead of total failure)
* Monitoring & recovery (detecting and fixing issues quickly)

Super short example (easy to remember):

Think of a food delivery app:

* If one server crashes, another server instantly takes over.
* The user still places orders without noticing anything broke.

# Durability

Durability is about data not being lost, even if things crash or fail.

If reliability = system keeps working,
then durability = your data stays safe no matter what.

Super short example (pair it with the previous one):

Same food delivery app:
* You place an order and it gets confirmed.
* Suddenly, the server crashes.
* When the system comes back, your order is still there.

👉 That persistence of data = durability.

# Common Points for System Design
---

* **Push vs Pull** - In Designing Message Queue or Metrics and Monitoring System, there is a common point of Push and Pull
	* In Message Queue, the consumer can either pull messages from broker or Broker can push messages to Consumers
	* Similarly in Metrics and Monitoring System, either the Metrics Collection Workers can pull data from the services or Collection Agent that runs
	on services will push data to the Collection Workers

* **Video Transcoding and Adaptive Bit Rate Streaming** - This is applicable in both the design youtube and design a live streaming service (Twitch)