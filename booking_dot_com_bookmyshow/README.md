# Hotel reservation of event reservation system

For a hotel reservation system with 100M daily active users, assuming 20% of users make a reservation and each makes one booking per day, the system handles approximately 20M bookings per day, or about 200 bookings per second on average.

Accounting for a 10× peak load factor during traffic spikes, the system must be designed to handle ~2K booking requests per second.

Key inferences an interviewer expects you to draw

- Bookings are low-QPS but high-value operations
- Strong consistency is required (inventory, pricing, payments)
- Read-heavy system overall (browse vs book)
- Write path must be carefully serialized and protected
- Queue + transactional DB needed for bookings
- Caching applies only to browsing, not booking
- The write path / booking path should be durable and bookings should not be lost