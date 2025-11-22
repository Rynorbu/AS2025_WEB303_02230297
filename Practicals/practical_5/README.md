# Practical 5: Refactoring a Monolithic Web Server to Microservices

**Student Name:** Ranjung Yeshi Norbu 
**Course:** WEB303 - Microservices & Serverless Applications  
**Date:** October 15, 2025

**GitHub Repository:** https://github.com/Rynorbu/Refactoring_a_Monolithic_Web_Server_to_Microservices

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Service Boundaries Justification](#service-boundaries-justification)
4. [Implementation Details](#implementation-details)
5. [Challenges Encountered](#challenges-encountered)
6. [Setup and Running Instructions](#setup-and-running-instructions)
7. [Testing Guide](#testing-guide)
8. [Reflection Essay](#reflection-essay)
9. [Screenshots](#screenshots)
10. [Conclusion](#conclusion)

---

## Project Overview

This practical demonstrates the systematic refactoring of a monolithic Student Cafe application into a microservices architecture. The project showcases:

- **Domain-Driven Design** principles for identifying service boundaries
- **Database-per-service** pattern implementation
- **Inter-service communication** via REST APIs
- **Service Discovery** using HashiCorp Consul
- **API Gateway** pattern for unified client access
- **Container orchestration** with Docker Compose

### Services Implemented

| Service | Port | Database | Responsibility |
|---------|------|----------|----------------|
| **Monolith** | 8081 | student_cafe (PostgreSQL) | Baseline monolithic application |
| **User Service** | 8084 | user_db (PostgreSQL) | User management and authentication |
| **Menu Service** | 8082 | menu_db (PostgreSQL) | Food catalog management |
| **Order Service** | 8083 | order_db (PostgreSQL) | Order processing and management |
| **API Gateway** | 8080 | N/A | Request routing and unified API |
| **Consul** | 8500 | N/A | Service discovery and health checks |

---

## Architecture Diagram

```
                                    ┌─────────────────┐
                                    │   API Gateway   │
                                    │   (Port 8080)   │
                                    └────────┬────────┘
                                             │
                    ┌────────────────────────┼──────────────────────┐
                    │                        │                      │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │  User Service  │      │  Menu Service  │      │ Order Service  │
            │   (Port 8084)  │      │   (Port 8082)  │      │   (Port 8083)  │
            └───────┬────────┘      └───────┬────────┘      └───────┬────────┘
                    │                       │                       │
            ┌───────▼────────┐      ┌───────▼────────┐      ┌───────▼────────┐
            │    user_db     │      │    menu_db     │      │   order_db     │
            │  (Port 5435)   │      │  (Port 5433)   │      │  (Port 5436)   │
            └────────────────┘      └────────────────┘      └────────────────┘

                    ┌─────────────────────────────────────┐
                    │      Consul (Service Discovery)     │
                    │           (Port 8500)               │
                    └─────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │        Monolith (Baseline)          │
                    │   Port 8081 → student_cafe DB       │
                    └─────────────────────────────────────┘
```

### Communication Flow
1. **Client → API Gateway** (Port 8080)
2. **API Gateway → Consul** (Service Discovery)
3. **API Gateway → Microservices** (Based on discovered addresses)
4. **Order Service → User/Menu Services** (Inter-service communication via Consul)
5. **Each Service → Its Database** (Database-per-service pattern)

---

## Service Boundaries Justification

### Why This Split Makes Sense

#### 1. **User Service** (User Context - Bounded Context)
- **Entities:** User (identified by user_id)
- **Business Capability:** User registration, profile management
- **Changes When:** User profile requirements evolve
- **Scales When:** High user registration/login traffic
- **Independence:** User data is completely independent of menu items and orders
- **Low Coupling:** Other services only need to validate user existence via user_id

#### 2. **Menu Service** (Menu Context - Bounded Context)
- **Entities:** MenuItem (identified by menu_item_id)
- **Business Capability:** Food catalog management, pricing updates
- **Changes When:** Menu items added/removed, prices updated
- **Scales When:** Heavy browsing during peak hours (lunch/dinner)
- **Independence:** Menu can be browsed without creating orders
- **High Cohesion:** Menu items and pricing always change together

#### 3. **Order Service** (Order Context - Bounded Context)
- **Entities:** Order, OrderItem (aggregates)
- **Business Capability:** Order creation, order tracking, order history
- **Changes When:** Order workflow changes (delivery, payment integration)
- **Scales When:** Order rush periods (lunch hours)
- **Dependencies:** References users and menu items but doesn't own them
- **Inter-service Communication:** Validates user_id and menu_item_id via HTTP calls

### Domain-Driven Design Principles Applied

1. **Bounded Contexts:** Each service represents a distinct business domain
2. **High Cohesion:** Related data and operations are grouped together
3. **Low Coupling:** Services communicate via well-defined APIs
4. **Aggregates:** Order service owns Order + OrderItems as a unit
5. **Business Capabilities:** Each service maps to a clear business function

---

## Implementation Details

### Technology Stack
- **Language:** Go 1.23
- **Web Framework:** Chi Router v5
- **ORM:** GORM
- **Database:** PostgreSQL 13
- **Service Discovery:** HashiCorp Consul
- **Containerization:** Docker & Docker Compose

### Key Design Decisions

#### Database-per-Service Pattern
Each microservice has its own PostgreSQL database:
- **Advantage:** Service independence, schema evolution, technology heterogeneity
- **Trade-off:** No cross-service transactions, eventual consistency required

#### Price Snapshotting in Orders
Order service stores menu item prices at order time:
```go
orderItem := models.OrderItem{
    MenuItemID: item.MenuItemID,
    Quantity:   item.Quantity,
    Price:      menuItem.Price, // Historical snapshot
}
```
**Rationale:** Historical orders shouldn't change when menu prices update

#### Consul-based Service Discovery
Services register themselves with Consul on startup:
```go
registration := &consulapi.AgentServiceRegistration{
    ID:      fmt.Sprintf("%s-%s", serviceName, hostname),
    Name:    serviceName,
    Port:    port,
    Address: hostname,
    Check: &consulapi.AgentServiceCheck{
        HTTP:     fmt.Sprintf("http://%s:%d/health", hostname, port),
        Interval: "10s",
        Timeout:  "3s",
    },
}
```
**Benefits:** Dynamic discovery, health checking, resilience

---

## Challenges Encountered

Building this microservices architecture wasn't a smooth, linear process. I encountered several roadblocks that forced me to rethink my approach and dig deeper into distributed systems concepts. Here's what I learned the hard way:

### Challenge 1: Getting Services to Talk to Each Other

When I first tried to make the Order Service validate users, I did what seemed obvious, hardcoded the URL as `http://user-service:8081`. It worked in my local Docker network, so I thought I was done. But then I started thinking: what happens when user-service moves to a different port? What if there are multiple instances for load balancing? My hardcoded approach would break immediately.

That's when I dove into Consul for service discovery. The learning curve was steep, understanding service registration, health checks, and DNS-based discovery took time. But once I got it working, it felt like magic. Services could now find each other dynamically, and if one instance went down, Consul would route traffic to healthy ones. This challenge taught me that in distributed systems, nothing should be static. Everything needs to be discoverable and resilient.

### Challenge 2: The Database Isolation Dilemma

Coming from a monolithic background where you can just JOIN tables across the entire database, the microservices approach felt restrictive. The Order Service needed to verify that a user exists before creating an order, but it couldn't directly query the user database. My first reaction was, "Why are we making this so complicated?"

I had to make HTTP calls to the User Service just to check if a user_id was valid. This meant network latency, potential failures, and more complexity. I seriously considered just sharing the database, it would be so much simpler! But as I researched more, I realized that database-per-service is what gives microservices their independence. If I shared the database, I'd have a distributed monolith, all the complexity of microservices with none of the benefits.

So I accepted the trade-off. Yes, there's latency. Yes, it's more complex. But the Order Service can now evolve independently, we can scale databases separately, and one service's database issues won't cascade to others. I also learned about eventual consistency patterns and caching strategies that could mitigate the performance impact in a production system.

### Challenge 3: The "Race Condition" of Service Startup

This one was frustrating. I'd run `docker-compose up`, watch the logs scroll by, and then see errors: "connection refused," "database does not exist," "panic: runtime error." What was happening? My services were starting before their databases were ready to accept connections.

I initially tried using Docker Compose's `depends_on`, thinking it would solve everything. Nope. `depends_on` only waits for the container to start, not for PostgreSQL to actually be ready to accept connections. So I added retry logic in my Go code, if the database connection fails, wait a second and try again. This helped, but it felt hacky.

The real solution was implementing proper health check endpoints (`/health`) that Consul could monitor. Combined with connection retry logic and strategic use of `depends_on`, services now start gracefully. This taught me that in distributed systems, you can't assume anything is ready when you need it, you have to build resilience into every component.

### Challenge 4: The Mysterious Go Module Checksum Errors

This one had me scratching my head for a while. I'd try to build a service, and Docker would throw cryptic errors about checksum mismatches in `go.sum`. I tried rebuilding, clearing caches, nothing worked. The builds that worked yesterday were suddenly failing.

After some research, I learned that Go modules maintain checksums to ensure dependency integrity. When I updated a dependency or changed Go versions, the `go.sum` file could get out of sync. The fix was simple but not obvious: delete `go.sum` and regenerate it with `go mod tidy`. I had to do this for each service individually.

This taught me the importance of dependency management and reproducible builds. Now I make it a habit to run `go mod tidy` after any dependency change and commit the updated `go.sum` file. It's one of those small practices that saves hours of debugging later.

### Challenge 5: Port Conflicts Everywhere

When I first set up multiple PostgreSQL databases, I naively tried to expose them all on port 5432. Docker immediately complained—"port already in use." Of course it was; I was trying to run four PostgreSQL instances on the same port!

The solution was understanding Docker's port mapping. Each container could internally use port 5432 (the PostgreSQL default), but I needed to map them to different host ports. So I created a mapping scheme:
- Menu DB: `5433:5432`
- User DB: `5435:5432`  
- Order DB: `5436:5432`
- Monolith DB: `5434:5432`

```yaml
user-db:
  ports:
    - "5435:5432"  # Host:Container
```

This way, each database is accessible on a unique host port, but internally they all run on their familiar port 5432. It seems obvious now, but at the time it was a valuable lesson in how containerization handles networking. I also learned to keep track of my port assignments in documentation, when you're managing 10+ containers, it's easy to lose track.

### What These Challenges Taught Me

Every challenge pushed me to understand distributed systems more deeply. I learned that microservices aren't just about splitting code, they're about embracing uncertainty, building for failure, and accepting complexity in exchange for flexibility. The frustrations were real, but working through them gave me practical skills that I couldn't have gained from just reading documentation.

---


## Reflection Essay

### Monolith vs Microservices: A Comparative Analysis

#### When Microservices Make Sense
For the Student Cafe application, microservices provide several advantages:

1. **Independent Scaling:** Menu service experiences high read traffic during browsing, while order service peaks during lunch hours. With microservices, we can scale each independently.

2. **Team Autonomy:** Separate teams can own user management, menu catalog, and order processing without stepping on each other's toes.

3. **Technology Flexibility:** If we want to add a recommendation engine using Python/ML libraries, we can build a new service without rewriting existing Go code.

4. **Deployment Independence:** Updating menu pricing logic doesn't require redeploying the entire system.

#### Trade-offs of Database-per-Service Pattern

**Advantages:**
- **Service Independence:** Each service controls its schema evolution
- **Technology Heterogeneity:** Could use MongoDB for menu catalog if needed
- **Failure Isolation:** Menu database issues don't crash user service

**Disadvantages:**
- **No ACID Transactions:** Can't atomically update users and orders together
- **Data Duplication:** Order service stores user_id without full user details
- **Query Complexity:** Joining data across services requires multiple API calls
- **Eventual Consistency:** Services may temporarily have stale views of other services' data

**Example Trade-off:**  
When creating an order, we validate user_id exists via HTTP call. If the user is deleted between validation and order creation, we get orphaned orders. Solutions include:
- Soft deletes for users
- Eventual consistency reconciliation jobs
- Saga pattern for distributed transactions

#### When NOT to Split a Monolith

Microservices add significant operational complexity. Avoid splitting when:

1. **Small Team:** < 5 developers can't effectively manage 10+ services
2. **Low Traffic:** If all services fit on one server, monolith simplicity wins
3. **Tight Coupling:** If every operation spans multiple services, you've created a distributed monolith
4. **Immature Domain:** When business boundaries aren't clear, premature splitting causes constant refactoring

**For Student Cafe:**  
If this was a startup with 2 developers and 100 users/day, a monolith would be better. Microservices make sense at scale (1000s of users, multiple teams).

#### Inter-Service Communication Without Direct Database Access

**The Challenge:**  
Order service needs to validate `user_id` exists without querying the user database directly.

**Solution Implemented:**
```go
// Discover user-service via Consul
userServiceURL, err := discoverService("user-service")

// HTTP GET to validate user
userResp, err := http.Get(fmt.Sprintf("%s/users/%d", userServiceURL, req.UserID))
if err != nil || userResp.StatusCode != http.StatusOK {
    return "User not found"
}
```

**Why This Works:**
- Maintains service independence (no shared database)
- User service owns user validation logic
- Consul provides resilience (discovers healthy instances)

**Alternatives Considered:**
1. **Event-Driven:** User service publishes "UserCreated" events; order service caches valid users
2. **gRPC:** Faster than HTTP/REST for service-to-service calls
3. **Shared Database:** Violates microservices principles but simplifies queries

#### What Happens If Menu Service Is Down?

**Current Behavior:**
```go
menuServiceURL, err := discoverService("menu-service")
if err != nil {
    return "Menu service unavailable" // HTTP 503
}
```

**Impact:**
- Orders cannot be created (menu items can't be validated)
- Existing orders can still be retrieved (no dependency)
- Menu browsing fails

**Resilience Patterns to Implement:**

1. **Circuit Breaker:**
   - After 5 consecutive failures, stop calling menu service for 30s
   - Prevents cascading failures

2. **Retry with Backoff:**
   ```go
   for i := 0; i < 3; i++ {
       resp, err := http.Get(menuURL)
       if err == nil { break }
       time.Sleep(time.Second * (1 << i)) // 1s, 2s, 4s
   }
   ```

3. **Cached Fallback:**
   - Order service caches menu items locally
   - Uses stale data if menu service is down
   - Trade-off: Orders might use outdated prices

4. **Graceful Degradation:**
   - Allow orders with known menu_item_ids without validation
   - Asynchronously reconcile when menu service recovers

#### Adding Caching for Performance

**Current Bottleneck:**  
Every order creation makes 2+ HTTP calls (user validation + N menu items).

**Caching Strategy:**

1. **API Gateway Level (Redis):**
   ```
   Cache-Key: GET /api/menu
   TTL: 5 minutes
   Invalidation: On POST /api/menu
   ```
   **Benefit:** Reduces load on menu service for read-heavy traffic

2. **Service-Level Caching:**
   ```go
   var menuCache sync.Map // In-memory cache
   
   func getMenuItem(id uint) (MenuItem, error) {
       if cached, ok := menuCache.Load(id); ok {
           return cached.(MenuItem), nil
       }
       // Fetch from database, store in cache
   }
   ```

3. **Consul DNS Caching:**
   - Cache service discovery results for 10s
   - Reduces Consul query overhead

4. **Database Query Caching:**
   - PostgreSQL prepared statements
   - GORM query caching for repeated queries

**When to Cache:**
- Menu items (high read, low write)
- User profiles (authentication tokens)

**When NOT to Cache:**
- Order status (needs real-time accuracy)
- Payment information (security risk)

### Conclusion: Lessons Learned

This practical taught me that microservices are **not a silver bullet**. They solve specific problems (scale, team autonomy) but introduce new challenges (distributed transactions, operational complexity).

**Key Takeaway:**  
Start with a monolith. Extract services only when:
1. Clear service boundaries emerge from the domain
2. Scaling needs justify the complexity
3. You have the team and tools to manage distributed systems

For Student Cafe at its current scale, the monolith might actually be the better choice. But this exercise demonstrated the **systematic thinking** required to evolve architectures as systems grow.

---

## Screenshots

### 1. Consul UI - Service Discovery & Health Checks
**Consul Dashboard showing all microservices registered and healthy:**

![Consul UI](assets/consul.png)

*This screenshot demonstrates:*
- All microservices successfully registered with Consul
- Health checks passing (green status)
- Service discovery mechanism working properly
- Each service's instance count and health status visible

---

### 2. Docker Compose - All Services Running
**Docker containers successfully built and running:**

![Docker Compose Up](assets/docker_up.png)

![Docker Compose Build](assets/docker_up1.png)

*These screenshots show:*
- All microservices starting successfully
- Database connections established
- Services registering with Consul
- No port conflicts or startup errors

---

### 3. Docker PS - Container Status
**All containers running and healthy:**

![Docker PS Output](assets/docker%20ps.png)

*This screenshot confirms:*
- 10+ containers running simultaneously
- User Service (port 8084)
- Menu Service (port 8082)
- Order Service (port 8083)
- API Gateway (port 8080)
- Consul (port 8500)
- Multiple PostgreSQL databases (ports 5433-5436)

---

### 4. Health Check Endpoint
**Service health endpoints responding:**

![Health Check](assets/health_check.png)

*This demonstrates:*
- `/health` endpoints implemented for all services
- Consul using these endpoints for health monitoring
- Services reporting healthy status
- Readiness checks working properly

---

### 5. Postman API Testing

#### Creating a User (User Service)
![Postman - Create User](assets/postamn_user.png)

*Test Result:*
-  POST request to `/api/users` successful
-  User created with ID, name, and email
-  API Gateway routing to user-service working
-  Database persistence confirmed

#### Creating Menu Items (Menu Service)
![Postman - Menu Items](assets/postman_menu.png)

*Test Result:*
-  POST request to `/api/menu` successful
-  Menu items created with name, description, and price
-  API Gateway routing to menu-service working
-  Menu database storing items correctly

#### Creating Orders (Order Service with Inter-Service Communication)
![Postman - Create Order](assets/postman_orders.png)

*Test Result:*
-  POST request to `/api/orders` successful
-  Order service validated `user_id` via HTTP call to user-service
-  Order service fetched menu item prices from menu-service
-  Order created with snapshotted prices
-  Inter-service communication working via Consul discovery

#### Order Creation Output (Console Logs)
![Order Service Output](assets/order_output.png)

*This shows:*
- Order service discovering user-service via Consul
- HTTP validation of user existence
- Order service discovering menu-service via Consul
- Fetching menu item details for price snapshotting
- Successful order creation with all validations passed

---

### 6. Docker Networking
**Docker networks enabling service communication:**

![Docker Network](assets/docker.png)

*This screenshot demonstrates:*
- Custom bridge network for microservices
- Service-to-service communication via container names
- Database isolation on the same network
- Consul accessible to all services

---

## Conclusion

Working through this practical has been an eye-opening journey into the world of microservices architecture. What started as a simple monolithic Student Cafe application transformed into a distributed system with multiple services, databases, and orchestration layers.

The refactoring process taught me that breaking down a monolith isn't just about splitting code into smaller pieces. It requires careful thought about domain boundaries, communication patterns, and operational complexity. Deciding where to draw the lines between services meant understanding the business domain deeply, recognizing that users, menu items, and orders represent distinct bounded contexts that naturally deserve their own services.

One of the biggest challenges was wrapping my head around the database-per-service pattern. Initially, it felt wrong not being able to join tables across services. But as I implemented inter-service communication through HTTP calls and Consul discovery, I realized this constraint actually forces better design. The Order Service doesn't need to know the internal structure of the User database, it just needs to know whether a user exists. This separation of concerns is powerful.

Getting everything to work together was definitely challenging. There were moments of frustration when services couldn't find each other, databases weren't ready, or port conflicts caused startup failures. But each problem solved deepened my understanding of distributed systems. Learning to use Consul for service discovery felt like unlocking a superpower, no more hardcoded URLs, just dynamic lookup of healthy service instances.

The testing phase was particularly satisfying. Watching Postman requests flow through the API Gateway, get routed to the right microservice via Consul, validate data across service boundaries, and return successful responses felt like conducting an orchestra. Each service played its part perfectly, and the Docker logs showed the beautiful dance of inter-service communication happening in real-time.

### Testing Summary

| Test Case | Status | Evidence |
|-----------|--------|----------|
| Consul Registration |  PASS | All services visible in Consul UI |
| Health Checks |  PASS | All health endpoints responding |
| User Creation |  PASS | Postman test successful |
| Menu Management |  PASS | Menu items created and retrieved |
| Order Creation |  PASS | Inter-service validation working |
| Docker Orchestration |  PASS | All containers running |
| Service Discovery |  PASS | Dynamic service lookup via Consul |
| Database Isolation |  PASS | Each service with separate DB |

### Final Thoughts

If I'm being honest, for an application of this size and complexity, the monolithic version might actually be the smarter choice. The microservices architecture introduced network latency, deployment complexity, and operational overhead that a simple cafe app probably doesn't need. But that's not the point of this exercise.

This practical was about learning the patterns, understanding the trade-offs, and gaining hands-on experience with tools and techniques that matter at scale. When I join a company working on a large platform with multiple teams and millions of users, I'll now understand why they've chosen microservices. I'll know how to design service boundaries thoughtfully, implement inter-service communication safely, and use tools like Consul and Docker Compose effectively.

The journey from monolith to microservices mirrors real-world software evolution. Systems grow, requirements change, and architectures must adapt. This practical gave me the mental models and technical skills to be part of that evolution. And perhaps most importantly, it taught me to always ask: "Does this complexity serve a real need, or are we just following a trend?" That critical thinking will serve me well throughout my career.



