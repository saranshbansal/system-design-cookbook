# CAP Theorem

The CAP theorem comes up when you're designing distributed systems and need to make tradeoffs about how your data behaves during failures. It states you can only have two of three properties at once. Consistency (all nodes see the same data), Availability (every request gets a response), and Partition tolerance (system works even when network connections fail between nodes). Since network partitions are unavoidable in distributed systems, you're really choosing between consistency and availability.

Here's what that means in practice. If you choose consistency, when a network partition happens, some nodes will refuse to serve requests rather than return potentially stale data. Your system might go down, but when it's up, the data is always correct. If you choose availability, every node keeps serving requests even during a partition. Users always get a response, but different nodes might temporarily have different data until the partition heals.

For most systems, availability is the right default. Users can tolerate seeing slightly stale data (your Instagram feed being 2 seconds old), but they can't tolerate the app being down. Social media feeds, recommendation systems, and analytics dashboards all work fine with eventual consistency.

Strong consistency matters when stale data causes actual business problems. Inventory systems need accurate stock counts or you'll oversell products. Banking systems need correct account balances or you'll allow fraud. Booking systems like Ticketmaster need to prevent double-booking the same seat. These are systems where reading stale data for even a few seconds can cost real money or create bad user experiences.

> You don't have to pick one model for your entire system. It's common to have different consistency requirements for different parts of the same application. In an e-commerce system, product descriptions and reviews can be eventually consistent (who cares if a review takes 5 seconds to appear), but inventory counts and order processing need strong consistency to prevent overselling.

In interviews, when you mention replication or distributed data, your interviewer might ask about consistency. The safe answer is eventual consistency unless the problem specifically involves money, inventory, or booking limited resources.