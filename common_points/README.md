# Common Points for System Design
---

* **Push vs Pull** - In Designing Message Queue or Metrics and Monitoring System, there is a common point of Push and Pull
	* In Message Queue, the consumer can either pull messages from broker or Broker can push messages to Consumers
	* Similarly in Metrics and Monitoring System, either the Metrics Collection Workers can pull data from the services or Collection Agent that runs
	on services will push data to the Collection Workers