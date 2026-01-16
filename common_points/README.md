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

# Common Points for System Design
---

* **Push vs Pull** - In Designing Message Queue or Metrics and Monitoring System, there is a common point of Push and Pull
	* In Message Queue, the consumer can either pull messages from broker or Broker can push messages to Consumers
	* Similarly in Metrics and Monitoring System, either the Metrics Collection Workers can pull data from the services or Collection Agent that runs
	on services will push data to the Collection Workers

* **Video Transcoding and Adaptive Bit Rate Streaming** - This is applicable in both the design youtube and design a live streaming service (Twitch)