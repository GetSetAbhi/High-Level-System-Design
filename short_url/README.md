# URL Shortener

---

![URL Shortener System](short_url_service.svg)

This section outlines the key components and their roles within the URL Shortner System.

This design is based on **Chapter 8 Design a URL Shortener** from the Book **System Design Interview Volume 1 by Alex Xu**, but i've taken
some liberty to add a modification.

There is a **.excalidraw** file which can be imported in [Excalidraw](https://excalidraw.com/) for additional customization to the design proposed.

## Components

There are 2 main components that i'll elaborate upon, rest of the things can be referred directly from the chapter in the book.

### 1. Role of zookeeper

In this design we generate **ID Block with Zookeeper**

The ID Block Allocation pattern with Zookeeper is a common and efficient strategy for generating globally unique, monotonically increasing IDs in a distributed system. It's designed to overcome the performance bottlenecks of a single centralized counter while still guaranteeing uniqueness and order.

**What is ID Block Allocation?**

Imagine you need to generate millions or billions of unique IDs (e.g., for user IDs, order IDs, or, in your case, the numerical backing for short URLs).

**Problem with Single Counter** 

If every service instance hits a single database or Redis instance for every ID, that single point becomes a massive bottleneck.

**Problem with Random IDs** 

If each instance generates random IDs, you risk collisions, 
and ensuring uniqueness becomes complex (e.g., constantly checking a database for existence).

**Solution** 

ID Block Allocation: Instead of getting one ID at a time, each distributed service instance (e.g., a "worker" or "application server") 
requests and holds a block of IDs (a contiguous range of numbers) from a central coordinator. 
It then uses these IDs locally from its allocated block, only returning to the coordinator when its block is exhausted.

**How it works**

 * When a URL Service starts up, or when it has used all the IDs from its currently allocated block, it sends a request
 to zookeeper for assigning an ID Block.
 
 * Once a worker has its allocated block ID Block, it stores this range in its local memory.
 
 * For every subsequent request sent to the URL Service to generate a short URL, the URL Service 
 increments its local counter (the value of the counter lies between the ID range assigned by zookeeper)
 and converts that counter value to a base 62 code which is the short url code.
 
 * If a URL Service instance crashed, then the ID Range assigned to it is lost and cannot be recovered.
 This is a known trade-off in this case. Given the vast number of IDs available (e.g., trillions for Base62), 
 losing a small fraction of IDs (e.g., 1 million per crash) is usually acceptable.
 
 
### 2. Choice of database

We chose a relational database for storing short code and long url combination.

* By choosing a relational database, we can enforce a Unique constraint on the short_code column 
making sure that a short_code generated is never used again for any other url.

* Databases typically create an index automatically when you add a UNIQUE constraint. 
This index makes lookups by short_code (which is the most frequent operation during redirection) extremely fast, which is critical for performance.

* It also simplifies our logic at the URL Service layer. While inserting a record,
The URL Service doesn't have to check if short_Code key exists or not, it can rely on the database to prevent duplicates.
The URL Service can just attempt an insert, and if it fails due to a unique constraint violation, it knows to generate another short_code.