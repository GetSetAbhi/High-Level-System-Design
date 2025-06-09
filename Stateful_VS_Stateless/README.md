## Web Tiers: Stateful vs. Stateless Architecture

### 1. Stateful Architecture

* **Definition:** Web servers **remember** client data (state) from one request to the next.

* **Data Storage:** Session data and user-specific information are stored **directly on individual web servers**.

* **Routing Problem:** Subsequent requests from the same user **must** be routed to the **same server** 
(e.g., User A always goes to Server 1, User B to Server 2). This often relies on "sticky sessions" via load balancers.

* **Challenges/Drawbacks:**
    * **Scaling Difficulty:** Hard to add or remove servers dynamically; requires careful management of sticky sessions, which adds overhead.
    * **Failure Handling:** If a server fails, all user sessions on that specific server are lost, leading to poor user experience.
    * **Complexity:** Adds complexity to load balancing, server maintenance, and ensuring high availability.

### 2. Stateless Architecture

* **Definition:** Web servers **do not store** any client data (state). They are "dumb" and process requests independently, fetching state as needed.

* **Data Storage:** Session data and all client-specific state are moved **out** of the web servers and stored in a **shared, persistent data store**.

* **Routing Advantage:** Any request from a user can be sent to **any** available web server because the state is fetched from the centralized shared store. 
Load balancers can distribute traffic more efficiently.

* **Benefits/Advantages:**
    * **Scalability:** Easy to scale horizontally by adding or removing web servers (autoscaling) as traffic changes, without affecting existing user sessions.
    * **Robustness/Resilience:** Server failures do not impact user sessions because state is externalized and not lost with a single server's demise.
    * **Simplicity:** Web servers are simpler, interchangeable, and easier to manage and deploy.
    * **Improved Load Balancing:** Eliminates the need for sticky sessions, allowing for more even distribution of load across servers.

### 3. Key Components in Stateless Web Tier Design 

* **Load Balancer:** Distributes incoming HTTP requests from users across multiple web servers, ensuring no single server is overloaded and enabling high availability.

* **Web Servers (Stateless Tier):** The application servers that process user requests. They are stateless, meaning they fetch necessary session or user data from a shared storage layer. 
They are designed for autoscaling.

* **Shared Storage:** The centralized layer where all stateful data is stored, accessible by any web server.

### 4. Common Shared Storage Options:

1. **Relational Databases (e.g., Master/Slave DBs):** 

	For persistent storage of structured data, often with replication for redundancy and read scaling.
	For high-traffic systems, a relational database is avoided as a shared storage option because it can become a performance bottleneck 
	due to the overhead of complex queries, disk I/O, and the need for joins if session data is spread across multiple tables.

2. **Distributed Key-Value Stores (e.g., Memcached/Redis):** 

	Used to store frequently accessed data in memory for fast retrieval, reducing load on databases.
	These options typically store shared data in a key-value pair.
	Ideal for applications that need lightning-fast session access and updates, such as high-traffic websites 
	and applications with frequent user interactions (e.g., shopping carts in e-commerce).

3. **NoSQL Databases:** 

	Offer flexible schema and high scalability for various types of data, often chosen for ease of scaling.
	We can opt for NoSQL storate When a system is expected to have a very large number of concurrent users 
	and significant volumes of session data that might not fit entirely in memory.

	Complex Session Data: If your session data is more complex than simple key-value pairs and benefits from a document-like structure 
	(e.g., nested JSON objects for user preferences, activity logs, etc.).

	NoSQL Databases offer stronger durability guarantees than an in-memory store

	In a real world scenario NoSQL db and Key-Value stores are used together for enhancing the performance. NoSQL Database acts as a persistence storage
	and Key-Value store acts as a cache to enable fast access to session data.

**In essence, Stateless architecture moves session data out of the web servers into shared storage, making the system highly scalable, robust, 
and simpler to manage compared to a stateful approach.**

### Real World Example