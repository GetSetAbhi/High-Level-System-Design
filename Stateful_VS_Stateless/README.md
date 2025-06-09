## Web Tiers: Stateful vs. Stateless Architecture

### 1. Stateful Architecture

* **Definition:** 

Web servers **remember** client data (state) from one request to the next.

* **Data Storage:** 

Session data and user-specific information are stored **directly on individual web servers**.

* **Routing Problem:** 

Subsequent requests from the same user **must** be routed to the **same server** 
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

### 3. Key Components in Stateless Web Tier Design (as depicted in Figure 1-14)

* **User/Clients:** The end-users interacting with the system via web browsers or mobile applications.
* **DNS (Domain Name System):** Resolves human-readable domain names (e.g., `www.mysite.com`) to IP addresses.
* **CDN (Content Delivery Network):** Serves static content (like images, CSS, JavaScript files) to users from geographically closer servers, improving speed and reducing load on the main web servers.
* **Load Balancer:** Distributes incoming HTTP requests from users across multiple web servers, ensuring no single server is overloaded and enabling high availability.
* **Web Servers (Stateless Tier):** The application servers that process user requests. They are stateless, meaning they fetch necessary session or user data from a shared storage layer. They are designed for autoscaling.
* **Shared Storage:** The centralized layer where all stateful data is stored, accessible by any web server.
    * **Databases (e.g., Master/Slave DBs):** For persistent storage of structured data, often with replication for redundancy and read scaling.
    * **Cache (e.g., Memcached/Redis):** Used to store frequently accessed data in memory for fast retrieval, reducing load on databases.
    * **NoSQL Databases:** Offer flexible schema and high scalability for various types of data, often chosen for ease of scaling.

**In essence, Stateless architecture moves session data out of the web servers into shared storage, making the system highly scalable, robust, and simpler to manage compared to a stateful approach.**