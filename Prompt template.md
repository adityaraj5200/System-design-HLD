```
Make a folder for [XYZ system] and in that folder, Design a [XYZ system].

Break down your solution into the following parts:

1. **Functional Requirements (FR):**
   - What core features should the system support?
   - Define inputs/outputs and key user interactions.

2. **Non-Functional Requirements (NFR):**
   - Performance (latency, throughput)
   - Scalability (horizontal/vertical)
   - Availability, Reliability
   - Consistency (eventual or strong)
   - Security, Cost-effectiveness

3. **High-Level Design (HLD):**
   - Draw and explain a component-level architecture diagram
   - Include client, API gateway, service layer, DB, cache, external systems
   - Explain how components interact
   - Justify choices (e.g., message queue vs direct call, monolith vs microservices)

4. **Database Design:**
   - List main entities and their relationships
   - Schema (SQL or NoSQL), and why you chose it
   - Indexing, sharding, replication, partitioning
   - What would have gone wrong if you chose a different DB model?

5. **API Design:**
   - Define RESTful endpoints (or GraphQL/RPC if more suitable)
   - Include URL structure, method, request/response, error codes
   - Mention authentication (e.g., JWT/OAuth) and rate limiting

6. **Detailed Justifications:**
   - For each major design choice (e.g., Redis vs Memcached, Cassandra vs MongoDB),
     explain:
     - Why it was chosen
     - What trade-offs were made
     - What would have gone wrong with the alternative

7. **Scale Considerations:**
   - **Low-scale version (~1,000 users):**
     - What can be simplified (monolith? local DB? no caching?)
     - Single-node deployment
   - **High-scale version (millions/billions of users):**
     - Add horizontal scaling, load balancers, CDN, data sharding, rate limiting
     - Use async processing, distributed queues, eventual consistency if needed
     - Add observability (logging, metrics, alerting)

8. **Bonus (Optional):**
   - Caching strategy (what to cache, cache invalidation)
   - Data retention policies
   - CI/CD pipeline overview
   - Disaster recovery and rollback strategy

Answer each section **in depth**, and make sure to clearly highlight **why each choice was made and the trade-offs involved**.
```