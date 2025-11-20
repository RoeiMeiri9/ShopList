## **To add to the diagram**

1. **Kafka and not RabbitMQ**

   - _Reference - [1. Core Architecture Setup](./Conversation%20with%20claude%20-%20summerized.md#1.-core-architecture-setup)_

   $\color{green}Done!$

2. **DB Writer Service**\
   Consider splitting into categories if it starts to feel heavy. You can start with one service and then split it into categories.

   - _Reference - [2. Caching Strategy](./Conversation%20with%20claude%20-%20summerized.md#2.-database-%26-caching-strategy)_

   $\color{green}Done!$

3. **Redis is primary DB**\
   Every DB call is from Redis, write manipulates Redis, DB Writer puts the data in the DB (fetches changes from Kafka every 30sec)

   - _Reference - [2. Caching Strategy](./Conversation%20with%20claude%20-%20summerized.md#2.-database-%26-caching-strategy)_
   - _Reference - [17. Database Decisions](./Conversation%20with%20claude%20-%20summerized.md#17.-database-decisions)_

   $\color{green}Done!$

4. **All services are writing and reading from kafka for every event**

   - _Reference - [2. Caching Strategy](./Conversation%20with%20claude%20-%20summerized.md#2.-database-%26-caching-strategy)_
   - _Reference - [6. Chat-to-List Product Suggestions Flow](./Conversation%20with%20claude%20-%20summerized.md#6.-chat-to-list-product-suggestions-flow)_

   $\color{green}Done!$

5. **WS - as NodeJS and Socket.io**

   - _Reference - [3. Websocket Service](./Conversation%20with%20claude%20-%20summerized.md#3.-websocket-service)_

   $\color{green}Done!$

6. **Splict Message Processor**\
   Split it to

   1. `Message Analayzer Service`
   2. `Scoring Service`
   3. `Keyword Manager Service`

   - _Reference - [4. Service Splitting](./Conversation%20with%20claude%20-%20summerized.md#4.-service-splitting)_

   $\color{green}Done!$

7. **Add Auth0**

   - _Reference - [15. Auth0 Integration (Later)](<./Conversation%20with%20claude%20-%20summerized.md#15.-auth0-integration-(later)>)_

   $\color{green}Done!$

8. **Add Statistics (greyed out for later use)**

   - _Reference - [26. B2B Statistics Service](./Conversation%20with%20claude%20-%20summerized.md#26.-b2b-statistics-service)_

   $\color{green}Done!$

9. **Product List - Redis for Read / Write, DB for Backup**

   - _Reference - [2. Database & Caching Strategy](./Conversation%20with%20claude%20-%20summerized.md#2.-database-%26-caching-strategy)_

   $\color{green}Done!$

10. **Add Product List Orchestrator**\
     It will look at the products others put, and make sure to update the Product List in the Redis. Also, it will notify the DB Writer service to update the DB.\
    $\color{green}Done!$
