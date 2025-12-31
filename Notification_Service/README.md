# Notification Service

---

![Notification Service Diagram](notification.svg)

This section outlines the key components and their roles within the Notification Service architecture.

This design is based on **Chapter 10 Design a Notification System** from the Book **System Design Interview Volume 1 by Alex Xu**.

There is a **.excalidraw** file which can be imported in [Excalidraw](https://excalidraw.com/) for additional customization to the design proposed.

## Capacity estimation

if the system is handling 100M notifcations per day then it means 1K notifciation per second and on peak 10K notifications per second assuming load factor of 10x

- The system is write-heavy

	Inference

	Optimize for high write throughput and asynchronous processing.

- Asynchronous processing is mandatory

	At 10K notifications/sec:

	Synchronous delivery would block API threads

	External providers introduce unpredictable latency

## Components


### Notification Trigger Services

These are external or internal services that initiate requests to send notifications. 
When offering services to other clients, these trigger services expose APIs that clients can call to request notifications.
	

### Authentication and Rate Limiter

This crucial component sits between the "Notification Trigger Services" and the "Notification Service" itself. 
Its primary functions include:
* **Authentication:** Verifying the identity and permissions of the caller (either a "Notification Trigger Service" or a direct client API call) 
before allowing the request to proceed.

* **Rate Limiting:** Controlling the number of API calls a specific "Notification Trigger Service" (or client) 
can make within a defined timeframe. This prevents abuse, protects the notification system from being overloaded by excessive requests, and helps manage costs.

### The notification Service

```
POST /api/v1/notifications

Payload

{
  "user_id": "usr_12345",              // Required: The ID of the user to notify
  "template_id": "email_welcome_v1",   // Required: The ID of the notification template to use
  "channel_type": "email",             // Optional: Preferred channel (e.g., "email", "sms", "push"). If not provided, Notification Service uses user's default preferences.
  "template_data": {                   // Required: Dynamic data to populate the template
    "name": "Alice",
    "account_status": "active"
  },
  "priority": "normal",                // Optional: e.g., "low", "normal", "high", "critical" (defaults to "normal")
  "correlation_id": "req_abc_789",     // Optional: Unique ID for tracing the notification request end-to-end
  "send_at": "2025-08-07T16:30:00Z"    // Optional: ISO 8601 timestamp for scheduled sending (if supported)
}
```
	
### Notification and User Preferences DB

The "Notification and User Preferences Database" will specifically store:

* **User Preferences & Opt-Outs:** This includes details about which users prefer which notification channels, 
their language settings, timezones, and, critically, their opt-out statuses for various notification types.
* **Notification Categories/Types Information:** This defines the different classifications of notifications your system handles (e.g., transactional, marketing, alerts) 
and their associated metadata.


```sql
-- Table for storing basic user information (if not handled by an external User Service)
-- This table is crucial for linking preferences to users.
CREATE TABLE Users (
    user_id VARCHAR(255) PRIMARY KEY, -- Unique identifier for the user (e.g., UUID, client-specific ID)
    email VARCHAR(255) UNIQUE,        -- User's email address (for email notifications)
    phone_number VARCHAR(20) UNIQUE,  -- User's phone number (for SMS notifications)
    locale VARCHAR(10) DEFAULT 'en_US', -- User's preferred language/locale (e.g., 'en_US', 'fr_FR')
    timezone VARCHAR(50) DEFAULT 'UTC', -- User's preferred timezone for scheduling
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Table for defining different categories or types of notifications
-- E.g., 'Transactional', 'Marketing', 'System Alerts'
CREATE TABLE NotificationCategories (
    category_id VARCHAR(50) PRIMARY KEY, -- Unique identifier for the category (e.g., 'ORDER_CONFIRMATION', 'PROMOTIONAL_NEWSLETTER')
    category_name VARCHAR(100) NOT NULL UNIQUE, -- Human-readable name (e.g., 'Order Confirmation', 'Promotional Newsletter')
    description TEXT,                       -- A brief description of the category
    notification_channel VARCHAR(50) NOT NULL, -- The primary channel for this category (e.g., 'EMAIL', 'SMS', 'PUSH', 'IN_APP')
    is_opt_out_allowed BOOLEAN DEFAULT TRUE, -- Can users opt out of this category? (e.g., transactional might not allow opt-out)
    default_opt_in BOOLEAN DEFAULT TRUE,    -- Default state for new users (true for opt-in, false for opt-out)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Table for storing individual user preferences for each notification category
-- This handles opt-in/opt-out status for each user for each category.
CREATE TABLE NotificationPreferences (
    preference_id INT AUTO_INCREMENT PRIMARY KEY, -- Auto-incrementing unique ID for each preference entry
    user_id VARCHAR(255) NOT NULL,                -- Foreign Key referencing Users table
    category_id VARCHAR(50) NOT NULL,             -- Foreign Key referencing NotificationCategories table
    is_opted_out BOOLEAN DEFAULT FALSE,           -- TRUE if user has opted out, FALSE if opted in
    last_modified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    -- Constraints to ensure a user has only one preference entry per category
    UNIQUE (user_id, category_id),
    -- Foreign key constraints
    FOREIGN KEY (user_id) REFERENCES Users(user_id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES NotificationCategories(category_id) ON DELETE CASCADE
);
```

**Rationale for a Relational Database**

Given the refined scope, a relational database (SQL) like PostgreSQL or MySQL is an excellent choice for the Notification and User Preferences DB.

* **Structured and Relational Data:** The data for user preferences, opt-outs, and notification categories is inherently structured. 
Users have multiple preferences, and categories have specific attributes. 
A relational database excels at defining and enforcing these relationships, ensuring data integrity.

* **Data Consistency and Integrity (ACID):** It's crucial that user opt-out statuses and category definitions are always accurate and consistent. 
Relational databases provide **ACID properties** (Atomicity, Consistency, Isolation, Durability), guaranteeing reliable transaction processing. 
This prevents issues like a user accidentally receiving notifications they've opted out of.

* **Efficient Querying:** While Redis will be used as a cache for frequent lookups, the relational database will serve as the source of truth and handle cache misses. 
SQL databases are highly optimized for querying structured data, which is essential for managing and analyzing user preferences and category definitions.

### Notification Log DB

We do not want to lose any notification, so we will maintain a log of notifications that were sent

**Rationale for a Using Cassandra Database**

Designed for extreme scalability and high write throughput across distributed clusters, 
making them suitable for massive volumes of append-only log data. Provides high availability.

```
CREATE TABLE notification_logs (
    notification_id UUID,
    timestamp TIMESTAMP, -- When the notification attempt was made (Clustering Key)
    channel_type TEXT,
    recipient TEXT,  -- e.g., email address, phone number, device token, Slack channel
    status TEXT,
    message_body TEXT,
	error_message TEXT, 
    PRIMARY KEY ((notification_id), timestamp)
);
```

* `notification_id` as Partition Key: Ensures even distribution of data across the cluster. Each notification attempt gets its own unique ID.
* `timestamp` as Clustering Key (with DESC order): Orders records within a partition by time, allowing efficient retrieval of recent logs for a specific notification.
* Denormalization: All relevant information (user ID, recipient, message content, status) is stored directly in the log entry. This avoids joins, which Cassandra doesn't support, and optimizes for fast reads of complete log records.

### Notification Template Store 

**Justification for using MongoDB**

* Schema Flexibility and Evolution:
Notification templates, especially for channels like email (HTML) or push notifications, can have highly varied and evolving structures. 
NoSQL document databases do not enforce a rigid schema, allowing you to store documents with different fields without requiring schema migrations.

* Performance for Full Document Retrieval:
When a "Notification Worker Server" needs a template, it typically fetches the entire template document (e.g., the full HTML body, subject, and metadata. 
Document databases are optimized for retrieving entire documents quickly.

```
[
  {
    "_id": "email_welcome_template_v1",  // Unique ID for the template
    "template_name": "Welcome Email",
    "template_type": "EMAIL",           // e.g., "EMAIL", "SMS", "PUSH", "SLACK"
    "version": 1,
    "status": "ACTIVE",                 // e.g., "ACTIVE", "INACTIVE", "DEPRECATED"
    "language": "en-US",                // Language of the template
    "channels": ["email"],              // Array of channels this template is intended for
    "content": {
      "subject": "Welcome to Our Service, {{user.name}}!",
      "html_body": "<html><body><h1>Welcome!</h1><p>Hi {{user.name}}, thanks for joining! Your account is {{account.status}}.</p></body></html>",
      "text_body": "Hi {{user.name}}, thanks for joining! Your account is {{account.status}}."
    },
    "created_at": ISODate("2025-08-07T10:00:00Z"),
    "updated_at": ISODate("2025-08-07T10:00:00Z")
  },
  {
    "_id": "sms_otp_template_v2",
    "template_name": "OTP SMS",
    "template_type": "SMS",
    "version": 2,
    "status": "ACTIVE",
    "language": "en-US",
    "channels": ["sms"],
    "content": {
      "text_body": "Your OTP is {{otp_code}}. It expires in {{expiry_minutes}} minutes. Do not share this code."
    },
    "created_at": ISODate("2025-02-15T08:30:00Z"),
    "updated_at": ISODate("2025-08-01T14:15:00Z")
  }
]
```