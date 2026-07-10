# System Design — 30 Core Concepts Notes

> Format for each topic: **Definition (simple)** → **Real-world use case** → **Why we use it** → **Market options today**

---

## Phase 1: Foundation & Communication

### 1. Client-Server Architecture
**Definition:** One machine (client) asks for something, another machine (server) does the work and replies. Client = browser/app, Server = backend.
**Use case:** You open Instagram → your phone (client) asks Instagram's servers for your feed.
**Why:** Keeps the "asking" part (UI) separate from the "doing" part (logic/data), so each can be built, scaled, and fixed independently.
**Market options:** Node.js/Express, Django, Spring Boot, ASP.NET, Go (Gin/Fiber) on the server side; any browser/mobile app as client.

### 2. IP Addresses
**Definition:** A unique numeric address assigned to every device connected to the internet, used to identify and locate it.
**Use case:** A server hosting oceanevoke.com has a fixed IP address that routers use to deliver traffic to it.
**Why:** Without a numeric addressing system, machines couldn't find each other across networks.
**Market options:** IPv4 (still dominant), IPv6 (growing adoption); assigned via cloud providers like AWS, GCP, Azure, or ISPs.

### 3. DNS (Domain Name System)
**Definition:** The "phonebook" of the internet — it maps human-readable domain names (like `google.com`) to numeric IP addresses.
**Use case:** Typing `oceanevoke.com` in a browser — DNS resolves it to the server's IP before the request is sent.
**Why:** Humans remember names, not numbers; DNS makes the internet usable.
**Market options:** Cloudflare DNS, AWS Route 53, Google Cloud DNS, GoDaddy DNS, Namecheap.

### 4. Proxy Server
**Definition:** A server that sits in front of the *client*, forwarding its requests on its behalf so the original client's identity/IP stays hidden.
**Use case:** A company routes all employee internet traffic through a proxy to enforce content policies and mask internal IPs.
**Why:** Adds privacy, security filtering, and access control for the client side of a connection.
**Market options:** Squid Proxy, NGINX (as forward proxy), corporate VPN/proxy appliances, residential proxy services.

### 5. Reverse Proxy
**Definition:** A server that sits in front of *backend servers*, intercepting incoming requests and routing them to the right one — clients never talk to the backend directly.
**Use case:** Cloudflare sitting in front of a WooCommerce backend, forwarding requests to the right origin server while blocking bots.
**Why:** Enables load balancing, SSL termination, caching, and security — all in one layer in front of your real servers.
**Market options:** NGINX, HAProxy, Cloudflare, Envoy, Traefik, AWS ALB.

### 6. Latency
**Definition:** The time delay caused by the physical distance data has to travel between client and server.
**Use case:** A user in Mumbai gets served from an AWS Asia Pacific (Mumbai) data center instead of a US server, cutting response time drastically.
**Why:** Lower latency means faster page loads, better user experience, and better SEO.
**Market options:** AWS Regions, Google Cloud Regions, Azure Regions, Cloudflare's 300+ edge locations.

### 7. HTTP/HTTPS
**Definition:** HTTP is the rulebook for how browsers and servers exchange data. HTTPS is the same thing, but encrypted (SSL/TLS) so nobody can read or tamper with it in transit.
**Use case:** Entering your card details on a checkout page — HTTPS keeps that data private during transmission.
**Why:** Security, user trust (browsers flag non-HTTPS sites), and it's required for modern web features (service workers, HTTP/2).
**Market options:** Let's Encrypt (free SSL), Cloudflare SSL, AWS Certificate Manager.

### 8. APIs
**Definition:** An API is the "menu" a server exposes so other systems can request data or trigger actions, without needing to know how the server works internally.
**Use case:** A weather app calling a weather provider's API to fetch today's forecast instead of running its own sensors.
**Why:** Lets different systems (frontend, backend, third parties) talk to each other through a defined, stable contract.
**Market options:** Express/FastAPI/Spring for building APIs; Postman/Swagger for documenting and testing them.

### 9. REST API
**Definition:** A stateless, resource-based API style where each URL represents a resource, and standard HTTP methods (GET, POST, PUT, DELETE) act on it.
**Use case:** `GET /products/12` fetches product 12, `PUT /products/12` updates it, `DELETE /products/12` removes it.
**Why:** Simple, predictable, cache-friendly, and the most widely adopted API standard on the web.
**Market options:** Express.js, FastAPI, Django REST Framework, Spring Boot REST.

### 10. GraphQL
**Definition:** A query language for APIs that lets the client ask for exactly the fields it needs in a single flexible request, instead of hitting multiple fixed endpoints.
**Use case:** A GraphQL query can ask for just `{ name, price }` of a product in one call, instead of REST returning the entire product object every time.
**Why:** Reduces over-fetching and under-fetching, especially useful for complex frontends pulling data from many sources.
**Market options:** Apollo Server, Hasura, GraphQL Yoga, AWS AppSync.

---

## Phase 2: Storage & Scaling

### 11. Database
**Definition:** A dedicated system whose sole job is to store, organize, and manage your application's data, separate from your application/server logic.
**Use case:** Your app server handles requests and business logic, but the actual product/order/customer data lives on a separate database server.
**Why:** Separating data storage from application logic lets each be scaled, backed up, secured, and optimized independently.
**Market options:** AWS RDS, Google Cloud SQL, PlanetScale, Supabase, MongoDB Atlas, or self-hosted PostgreSQL/MySQL.

### 12. SQL vs. NoSQL
**Definition:** SQL databases store data in structured tables with fixed schemas and strict consistency rules (ACID). NoSQL databases store flexible, schema-less data (documents, key-value pairs, graphs) built for scale.
**Use case:** Bank ledgers → SQL (money must always be exact and consistent). A product catalog with wildly different attributes per item → NoSQL (MongoDB).
**Why:** Choose SQL for correctness-critical, relational data; NoSQL for flexibility and horizontal scale.
**Market options:** SQL — PostgreSQL, MySQL, MariaDB; NoSQL — MongoDB, DynamoDB, Cassandra, Firebase Firestore.

### 13. Vertical Scaling
**Definition:** Making a single machine more powerful — adding more CPU, RAM, or storage to the same server.
**Use case:** Upgrading a single AWS EC2 instance from t3.medium to t3.xlarge to handle more traffic.
**Why:** Simple to do with no architectural changes needed, but it hits a hardware ceiling and gets expensive fast.
**Market options:** AWS EC2 instance resizing, Google Cloud Compute Engine, Azure VM resizing.

### 14. Horizontal Scaling
**Definition:** Adding more machines to share the load, instead of making one machine bigger.
**Use case:** Running 5 smaller app server instances behind a load balancer instead of one giant server.
**Why:** This is what lets systems handle millions of users — it also improves reliability, since one server going down doesn't take everything with it.
**Market options:** AWS Auto Scaling Groups, Kubernetes (horizontal pod autoscaling), Google Cloud Compute Engine.

### 15. Load Balancer
**Definition:** A traffic cop that spreads incoming requests across multiple servers so no single one is overwhelmed.
**Use case:** Vercel or AWS ALB distributing incoming traffic across multiple Lambda/EC2 instances during a traffic spike.
**Why:** Prevents any one server from becoming a bottleneck or single point of failure; enables horizontal scaling.
**Market options:** AWS ALB/NLB, NGINX, HAProxy, Cloudflare Load Balancing, Google Cloud Load Balancer.

### 16. Indexing
**Definition:** A separate, sorted lookup structure (like a book's index) that lets the database find rows fast instead of scanning every row.
**Use case:** Searching a `products` table by `sku` — with an index, it's near-instant; without one, the database scans the entire table.
**Why:** Massive speed-up for read/search queries, at the cost of slightly slower writes and extra storage.
**Market options:** Built into PostgreSQL/MySQL (B-Tree, GIN, Hash indexes); Elasticsearch for full-text search indexing.

### 17. Replication
**Definition:** Keeping multiple live copies of a database. Usually one **primary** handles writes, and several **read replicas** handle read traffic.
**Use case:** An e-commerce site routes all "browse products" reads to replicas, while checkout writes go to the primary — keeping reads fast even under heavy write load.
**Why:** Improves read throughput and adds redundancy/failover if the primary goes down.
**Market options:** PostgreSQL streaming replication, MySQL replication, AWS RDS Read Replicas, MongoDB Replica Sets.

### 18. Sharding
**Definition:** Splitting a huge dataset **by rows** across multiple database servers, so each server only holds a slice of the data.
**Use case:** A global user table split by region — US users on shard 1, EU users on shard 2 — so each database stays a manageable size.
**Why:** Lets a single logical dataset grow beyond what one machine can store or serve.
**Market options:** MongoDB sharding, Vitess (for MySQL), Citus (for Postgres), DynamoDB (auto-sharded).

### 19. Vertical Partitioning
**Definition:** Splitting a table **by columns** instead of rows — frequently-used columns in one table, rarely-used/large columns in another.
**Use case:** Splitting a `users` table into `users_core` (name, email, login) and `users_profile` (bio, preferences, avatar) so the frequently-queried core stays lean and fast.
**Why:** Speeds up queries that only need "hot" columns by avoiding pulling large, rarely-used data along with them.
**Market options:** Manual schema design in PostgreSQL/MySQL; also common in columnar-storage tools like Snowflake, BigQuery.

### 20. Caching
**Definition:** Storing frequently-requested data in fast memory (RAM) so you don't hit the slow database/disk every time.
**Use case:** Caching a product page's data in Redis for 5 minutes so repeated visits don't re-query WooCommerce/MySQL each time.
**Why:** Massive speed improvement and reduced database load. The **Cache-Aside** pattern (check cache → if miss, fetch from DB → store in cache) is the most common approach.
**Market options:** Redis, Memcached, Cloudflare Cache/CDN caching, Varnish.

### 21. Denormalization
**Definition:** Deliberately duplicating data across tables (instead of keeping it perfectly normalized) to avoid expensive JOINs.
**Use case:** Storing a product's `category_name` directly on the order row instead of joining `orders → products → categories` every time you display an order history.
**Why:** Faster reads at the cost of extra storage and more complex writes (you must update duplicates consistently).
**Market options:** Common pattern in NoSQL (MongoDB); also used deliberately in SQL for read-heavy reporting tables/data warehouses.

---

## Phase 3: Advanced Architectures & Reliability

### 22. CAP Theorem
**Definition:** In any distributed system, when a network partition happens, you can only guarantee **two** of these three: **C**onsistency (everyone sees the same data), **A**vailability (system always responds), **P**artition Tolerance (system keeps working despite network splits). Partition tolerance is usually non-negotiable, so the real choice is **CP vs. AP**.
**Use case:** A banking system favors **CP** (consistency) — better to be briefly unavailable than show wrong balances. A social media "like count" favors **AP** (availability) — better to show a slightly stale count than go down.
**Why:** Helps architects consciously decide what to sacrifice when networks fail, instead of pretending you can have everything.
**Market options:** CP systems — MongoDB (default), HBase, Zookeeper; AP systems — Cassandra, DynamoDB, CouchDB.

### 23. Blob Storage
**Definition:** Storage designed for large, unstructured files — images, videos, PDFs — instead of structured database rows.
**Use case:** Product images and watermark assets for an e-commerce store stored in AWS S3 or Cloudflare R2, served via a CDN.
**Why:** Databases aren't built to efficiently store huge binary files; blob storage is cheap, durable, and scales infinitely.
**Market options:** AWS S3, Cloudflare R2, Google Cloud Storage, Azure Blob Storage.

### 24. CDN (Content Delivery Network)
**Definition:** A network of servers spread across the globe that cache your static files (images, JS, CSS) close to the user.
**Use case:** A user in Kolkata loading product images from an Oceanevoke store gets them from a nearby Cloudflare edge node instead of the origin server in another country.
**Why:** Cuts latency dramatically, reduces load on your origin server, and improves resilience during traffic spikes.
**Market options:** Cloudflare, Amazon CloudFront, Fastly, Akamai, Vercel Edge Network.

### 25. WebSockets
**Definition:** A persistent, two-way (full-duplex) connection between client and server that stays open, unlike normal HTTP requests which close after each reply.
**Use case:** A live chat app or a stock ticker where prices update instantly without the client having to keep asking "anything new?"
**Why:** Enables true real-time communication with low overhead, instead of constantly polling the server.
**Market options:** Socket.IO, native WebSocket API, Pusher, Ably, AWS API Gateway WebSockets.

### 26. Webhooks
**Definition:** A "push" notification system — instead of your app repeatedly asking another service "did anything happen?", that service calls *your* URL automatically when an event occurs.
**Use case:** ShipRocket calling your server's webhook URL the moment a shipment status changes, instead of your server polling ShipRocket's API every minute.
**Why:** Far more efficient than polling; near-instant updates with minimal wasted requests.
**Market options:** Native to most SaaS APIs (Stripe, ShipRocket, WooCommerce, GitHub); tools like Svix or Hookdeck for managing webhook delivery/retries.

### 27. Microservices
**Definition:** Instead of one giant application (monolith), the app is split into small, independently-deployable services, each owning one responsibility.
**Use case:** An e-commerce platform with separate services for `orders`, `inventory`, `payments`, and `shipping`, each deployable and scalable on its own.
**Why:** Teams can develop, deploy, and scale services independently; a failure in one service doesn't necessarily crash the whole system.
**Market options:** Docker + Kubernetes, AWS ECS/EKS, Google Cloud Run, Nomad.

### 28. Message Queues
**Definition:** A buffer between services — a **producer** drops a message into the queue, and a **consumer** processes it whenever it's ready, decoupling the two.
**Use case:** When an order is placed, the order service drops a "send confirmation email" message into a queue; an email-worker service processes it separately, so a slow email provider never delays checkout.
**Why:** Prevents a traffic spike or slow downstream service from crashing the whole system; enables async processing and retries.
**Market options:** RabbitMQ, Apache Kafka, AWS SQS, Google Pub/Sub, Redis Streams.

### 29. Rate Limiting
**Definition:** Restricting how many requests a single user/IP/API key can make in a given time window.
**Use case:** Limiting login attempts to 5 per minute per IP to block brute-force attacks and bots hammering an API.
**Why:** Protects backend resources from abuse, bots, and accidental overload; keeps service fair across all users.
**Market options:** Cloudflare Rate Limiting, NGINX `limit_req`, AWS API Gateway throttling, Redis-based custom limiters (token bucket/sliding window).

### 30. API Gateway
**Definition:** A single front door that all client requests pass through before reaching internal microservices — handles auth, rate limiting, and routing in one place.
**Use case:** A mobile app hits one API Gateway URL; the gateway checks the auth token, applies rate limits, then routes the request to the correct microservice (orders, users, payments).
**Why:** Avoids duplicating auth/rate-limiting logic in every microservice; gives one clean, secure entry point.
**Market options:** AWS API Gateway, Kong, Apigee, NGINX, Cloudflare API Gateway.

### 31. Idempotency
**Definition:** A property where performing the same operation multiple times produces the exact same result as doing it once — no duplicate side effects.
**Use case:** If a payment request times out and the client retries it, an **idempotency key** ensures the customer is charged only once, not twice.
**Why:** Networks are unreliable — retries are inevitable. Idempotency makes retries safe instead of dangerous.
**Market options:** Stripe/Razorpay idempotency keys (built-in), custom implementation using unique request IDs stored in Redis/DB before processing.

---

*Note: the source video numbers its concepts 1–31 (numbering rolls over after "30. API Gateway" to "31. Idempotency") — all 31 points from the video are covered above, grouped into the same 3 phases as the original.*
