# 🚀 Practical 3: Full-Stack Microservices with gRPC, Databases, and Service Discovery

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [My Approach](#my-approach)
- [Implementation Steps](#implementation-steps)
- [Key Features](#key-features)
- [Challenges Encountered](#challenges-encountered)
- [Technical Solutions](#technical-solutions)
- [Testing Results](#testing-results)
- [Evidence Documentation](#evidence-documentation)
- [Project Structure](#project-structure)
- [API Documentation](#api-documentation)
- [Setup Instructions](#setup-instructions)
- [Learning Outcomes](#learning-outcomes)
- [Conclusion](#conclusion)

## 📖 Overview

This project implements a complete microservices ecosystem demonstrating modern cloud-native architecture patterns. The system consists of two independent microservices (Users and Products) that communicate via gRPC, utilize separate PostgreSQL databases, and register themselves with Consul for dynamic service discovery. An API Gateway serves as the single entry point, translating HTTP requests into gRPC calls and providing data aggregation capabilities.

**Repository**: [https://github.com/Rynorbu/AS2025_WEB303_02230297_Practical3](https://github.com/Rynorbu/AS2025_WEB303_02230297_Practical3)

## 🏗️ Architecture

The system implements a distributed microservices architecture with the following components:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   API Gateway   │◄──►│ Consul Discovery │◄──►│  Microservices  │
│   (Port 8080)   │    │   (Port 8500)    │    │                 │
│                 │    │                  │    │ Users Service   │
│ 1. Query Consul │    │ Service Registry │    │ (Port 50051)    │
│ 2. Get Addresses│    │                  │    │                 │
│ 3. Connect to   │    │ Health Checks    │    │ Products Service│
│    Services     │    │                  │    │ (Port 50052)    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                               │                 │
┌─────────────────┐    ┌──────────────────┐    │                 │
│   PostgreSQL    │◄───│   PostgreSQL     │◄───┘                 │
│   Users DB      │    │   Products DB    │                      │
│   (Port 5432)   │    │   (Port 5433)    │                      │
└─────────────────┘    └──────────────────┘                      │
```

### Core Components
| Component | Technology | Port | Purpose |
|-----------|------------|------|---------|
| API Gateway | Go + Gorilla Mux | 8080 | HTTP REST API with dynamic service discovery |
| Users Service | Go + gRPC + GORM | 50051 | User management microservice |
| Products Service | Go + gRPC + GORM | 50052 | Product management microservice |
| Consul | HashiCorp Consul | 8500 | Service discovery and health monitoring |
| Users Database | PostgreSQL 13 | 5432 | Isolated user data persistence |
| Products Database | PostgreSQL 13 | 5433 | Isolated product data persistence |

## 🎯 My Approach

### 1. **Understanding the Problem**
The main challenge was to fix the API Gateway to properly utilize Consul service discovery instead of hardcoded service addresses, and implement proper service communication for the composite endpoint that aggregates data from multiple services.

### 2. **Design Philosophy**
- **Microservices First**: Each service maintains its own database and business logic
- **Service Discovery**: Dynamic service location through Consul registry
- **Protocol Buffers**: Type-safe inter-service communication
- **Database Isolation**: Complete data separation between services
- **Containerization**: Docker-based deployment for consistency

### 3. **Problem Solving Strategy**
- Analyzed the existing codebase to understand the hardcoded connections
- Implemented Consul client integration for dynamic service discovery
- Fixed the composite endpoint to properly aggregate data from both services
- Enhanced error handling and concurrent processing

## 📋 Implementation Steps

### Step 1: Project Setup and Structure
```bash
# Created the project structure
mkdir -p practical-three/{proto/gen,api-gateway,services/{users-service,products-service}}
```

### Step 2: Protocol Buffer Definitions
- Defined `users.proto` and `products.proto` service contracts
- Generated Go gRPC code using `protoc` compiler
- Established strongly-typed communication interfaces

### Step 3: Database Configuration
- Configured separate PostgreSQL instances for each service
- Implemented GORM models for Users and Products
- Set up automatic schema migration

### Step 4: Microservices Implementation
**Users Service:**
- Implemented gRPC server with CRUD operations
- Database connectivity with connection pooling
- Consul service registration with health checks

**Products Service:**
- Similar architecture to Users Service
- Independent database and business logic
- Automatic service discovery registration

### Step 5: API Gateway Development
**Key fixes implemented:**
- **Dynamic Service Discovery**: Replaced hardcoded addresses with Consul queries
- **Service Resolution**: Real-time service address lookup
- **Error Handling**: Robust error management for service unavailability
- **Data Aggregation**: Fixed composite endpoint for multi-service data

### Step 6: Containerization
- Created multi-stage Dockerfiles for each service
- Configured docker-compose.yml for service orchestration
- Implemented proper service dependencies and networking

### Step 7: Testing and Validation
- Comprehensive API testing with curl and Postman
- Service health monitoring through Consul UI
- End-to-end functionality verification

## ✨ Key Features

### 🔄 Dynamic Service Discovery
- API Gateway queries Consul on each request for real-time service location
- Automatic service registration and deregistration
- Health check monitoring and failover capabilities

### 🚀 gRPC Communication
- High-performance binary protocol for inter-service communication
- Protocol Buffer schema validation
- Streaming support for future enhancements

### 🛡️ Database Isolation
- Each microservice maintains its own PostgreSQL database
- Complete data separation and independence
- Individual scaling capabilities

### ⚡ Concurrent Processing
- Parallel gRPC calls in composite endpoints
- Goroutine-based concurrent data retrieval
- Efficient resource utilization

### 🐳 Full Containerization
- Docker-based deployment with multi-stage builds
- Container networking and service mesh
- Easy scaling and deployment

## 🚧 Challenges Encountered

### Challenge 1: Service Discovery Integration
**Problem**: The original API Gateway used hardcoded service addresses (`users-service:50051`), preventing dynamic scaling and proper service discovery.

**Solution**: 
- Implemented Consul client integration
- Created dynamic service address resolution
- Added error handling for service unavailability

```go
func discoverService(serviceName string) (string, error) {
    config := consulapi.DefaultConfig()
    consul, err := consulapi.NewClient(config)
    if err != nil {
        return "", err
    }
    
    services, _, err := consul.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return "", err
    }
    
    if len(services) == 0 {
        return "", fmt.Errorf("no healthy instances of %s found", serviceName)
    }
    
    service := services[0]
    return fmt.Sprintf("%s:%d", service.Service.Address, service.Service.Port), nil
}
```

### Challenge 2: Composite Endpoint Implementation
**Problem**: The `/api/purchases/user/{userId}/product/{productId}` endpoint wasn't properly aggregating data from both services.

**Solution**:
- Implemented concurrent gRPC calls using goroutines
- Added proper error handling and synchronization
- Created structured response format

### Challenge 3: Container Networking
**Problem**: Services couldn't communicate within Docker network due to hostname resolution issues.

**Solution**:
- Configured proper Docker networking in docker-compose
- Used container names as hostnames
- Implemented connection retry logic

### Challenge 4: Database Connection Management
**Problem**: Intermittent database connection failures during container startup.

**Solution**:
- Added connection retry logic with exponential backoff
- Implemented health checks in docker-compose
- Added proper database initialization waiting

## 🔧 Technical Solutions

### Service Discovery Implementation
```go
// Dynamic service address resolution
func getServiceAddress(serviceName string) (string, error) {
    consul, err := consulapi.NewClient(consulapi.DefaultConfig())
    if err != nil {
        return "", err
    }
    
    services, _, err := consul.Health().Service(serviceName, "", true, nil)
    if err != nil || len(services) == 0 {
        return "", fmt.Errorf("service %s not found", serviceName)
    }
    
    return fmt.Sprintf("%s:%d", services[0].Service.Address, services[0].Service.Port), nil
}
```

### Concurrent Data Aggregation
```go
func getPurchaseDataHandler(w http.ResponseWriter, r *http.Request) {
    vars := mux.Vars(r)
    userId := vars["userId"]
    productId := vars["productId"]

    var wg sync.WaitGroup
    var user *pb.User
    var product *pb.Product
    var userErr, productErr error

    // Concurrent gRPC calls
    wg.Add(2)
    go func() {
        defer wg.Done()
        // Fetch user data
    }()
    go func() {
        defer wg.Done()
        // Fetch product data
    }()
    wg.Wait()
    
    // Aggregate and return response
}
```

## 🧪 Testing Results

### Successful API Endpoints Tested:

#### User Management
```bash
# Create User
curl -X POST -H "Content-Type: application/json" \
     -d '{"name": "Jane Doe", "email": "jane.doe@example.com"}' \
     http://localhost:8080/api/users

# Get User
curl http://localhost:8080/api/users/1
```

#### Product Management
```bash
# Create Product
curl -X POST -H "Content-Type: application/json" \
     -d '{"name": "Laptop", "price": 1200.50}' \
     http://localhost:8080/api/products

# Get Product
curl http://localhost:8080/api/products/1
```


#### Data Aggregation
```bash
# Get Combined Purchase Data
curl http://localhost:8080/api/purchases/user/1/product/1
```

### Testing Results Screenshot
![API Testing Results](assets/terminal.png)


The screenshot demonstrates successful:
- ✅ User creation and retrieval via dynamic service discovery
- ✅ Product management through Consul routing
- ✅ Real-time data aggregation from multiple microservices

## 📸 Evidence Documentation

This section provides comprehensive visual proof of the successful implementation and testing of the microservices system. Each screenshot demonstrates a critical aspect of the working system.

### 1. 🔍 Consul Service Discovery Dashboard
![Consul Dashboard](assets/consul.png)

**Evidence Shows:**
- ✅ All services properly registered with Consul
- ✅ Health status of `users-service` and `products-service` 
- ✅ Service discovery mechanism working correctly
- ✅ Dynamic service registration and health monitoring
- ✅ Consul UI accessible at `http://localhost:8500`

**What This Proves:**
- Services are automatically registering themselves with Consul upon startup
- Health checks are functioning and showing service availability
- The service discovery infrastructure is operational

### 2. 🐳 Docker Compose System Running
![Docker Compose Running](assets/docker_compose.png)

**Evidence Shows:**
- ✅ All containers successfully built and started
- ✅ No build errors or dependency issues
- ✅ Services starting in correct order due to `depends_on` configuration
- ✅ Container networking properly configured
- ✅ All ports mapped correctly

**What This Proves:**
- The entire microservices stack can be deployed with a single command
- Container orchestration is working as designed
- All services are communicating within the Docker network

### 3. 📦 Docker Containers Active Status
![Docker Containers Active](assets/docker.png)

**Evidence Shows:**
- ✅ All 6 containers running successfully:
  - `consul` - Service discovery
  - `users-db` - PostgreSQL database for users
  - `products-db` - PostgreSQL database for products  
  - `users-service` - User management microservice
  - `products-service` - Product management microservice
  - `api-gateway` - HTTP REST API gateway
- ✅ Proper port mappings and container health
- ✅ No crashed or exited containers

**What This Proves:**
- Complete system deployment successful
- All components are healthy and operational
- Database isolation working with separate PostgreSQL instances

### 4. 🧪 Postman API Verification
![Postman Testing](assets/postman.png)

**Evidence Shows:**
- ✅ Successful API endpoint testing through Postman
- ✅ HTTP status codes (200, 201) indicating successful operations
- ✅ Proper JSON request and response formatting
- ✅ All CRUD operations working correctly:
  - User creation and retrieval
  - Product creation and retrieval
  - Composite data aggregation endpoint
- ✅ Service discovery routing requests correctly

**What This Proves:**
- API Gateway successfully translating HTTP to gRPC calls
- Dynamic service discovery working in practice
- End-to-end functionality verified through external testing tool

### 5. 💻 Terminal User Creation and Retrieval
![API Testing Results](assets/terminal.png)

**Evidence Shows:**
- ✅ `curl` commands creating new users successfully
- ✅ User retrieval working with proper JSON responses
- ✅ Database persistence verified through consecutive operations
- ✅ gRPC communication between gateway and users-service
- ✅ PostgreSQL database integration working correctly

**Command Examples Tested:**
```bash
# Create User
curl -X POST -H "Content-Type: application/json" \
     -d '{"name": "Jane Doe", "email": "jane.doe@example.com"}' \
     http://localhost:8080/api/users

# Retrieve User  
curl http://localhost:8080/api/users/1
```

**What This Proves:**
- Complete end-to-end functionality from HTTP request to database storage
- Service discovery enabling proper request routing
- Data persistence working correctly
- Terminal-based testing validates the implementation

### 🎯 Evidence Summary

The screenshots collectively demonstrate:

| Component | Evidence | Status |
|-----------|----------|--------|
| Service Discovery | Consul dashboard showing registered services | ✅ Working |
| Container Orchestration | Docker compose and container status | ✅ Working |
| API Gateway | Postman successful API calls | ✅ Working |
| Database Integration | Terminal user CRUD operations | ✅ Working |
| End-to-End Flow | All components working together | ✅ Working |

**Implementation Verification Checklist:**
- [x] Service registration with Consul
- [x] Dynamic service discovery in API Gateway  
- [x] gRPC inter-service communication
- [x] Database isolation and persistence
- [x] HTTP to gRPC translation
- [x] Concurrent data aggregation
- [x] Container networking and deployment
- [x] Health monitoring and observability

This comprehensive evidence documentation proves that all practical requirements have been successfully implemented and the microservices system is fully operational.

## 📁 Project Structure

```
practical-three/
├── 🌐 api-gateway/                 # HTTP REST API Gateway
│   ├── main.go                     # Dynamic service discovery logic
│   ├── Dockerfile                  # Multi-stage build configuration
│   ├── go.mod                      # Go module dependencies
│   └── go.sum                      # Dependency checksums
├── 🔧 services/
│   ├── 👥 users-service/           # User management microservice
│   │   ├── main.go                 # gRPC server with database integration
│   │   ├── Dockerfile              # Service containerization
│   │   ├── go.mod                  # Service-specific dependencies
│   │   └── go.sum
│   └── 🛍️ products-service/        # Product management microservice
│       ├── main.go                 # gRPC server with database integration
│       ├── Dockerfile              # Service containerization
│       ├── go.mod                  # Service-specific dependencies
│       └── go.sum
├── 📡 proto/                       # Protocol Buffer definitions
│   ├── users.proto                 # User service contract
│   ├── products.proto              # Product service contract
│   └── gen/proto/                  # Generated Go code
│       ├── go.mod                  # Proto module
│       ├── users.pb.go             # User protobuf bindings
│       ├── users_grpc.pb.go        # User gRPC client/server
│       ├── products.pb.go          # Product protobuf bindings
│       └── products_grpc.pb.go     # Product gRPC client/server
├── 🐳 docker-compose.yml           # Multi-service orchestration
├── 📋 buf.yaml                     # Buf configuration
├── ⚙️ buf.gen.yaml                 # Code generation settings
├── 🔧 Makefile                     # Build automation
└── 📖 README.md                    # Project documentation
```

## 📚 API Documentation

### Base URL
```
http://localhost:8080
```

### Endpoints

#### User Management
| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| POST | `/api/users` | Create new user | `{"name": "string", "email": "string"}` |
| GET | `/api/users/{id}` | Get user by ID | None |

#### Product Management
| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| POST | `/api/products` | Create new product | `{"name": "string", "price": number}` |
| GET | `/api/products/{id}` | Get product by ID | None |

#### Data Aggregation
| Method | Endpoint | Description | Response |
|--------|----------|-------------|----------|
| GET | `/api/purchases/user/{userId}/product/{productId}` | Get combined user and product data | `{"user": {...}, "product": {...}}` |

## 🚀 Setup Instructions

### Prerequisites
- Docker and Docker Compose
- Go 1.18+ (for local development)
- Protocol Buffers Compiler
- Buf CLI (for proto generation)

### Quick Start
1. **Clone the repository:**
   ```bash
   git clone https://github.com/Rynorbu/AS2025_WEB303_02230297_Practical3.git
   cd AS2025_WEB303_02230297_Practical3
   ```

2. **Generate Protocol Buffers (if needed):**
   ```bash
   buf generate
   ```

3. **Build and run the entire stack:**
   ```bash
   docker-compose up --build
   ```

4. **Verify services are running:**
   ```bash
   docker-compose ps
   ```

5. **Monitor service health:**
   Visit [http://localhost:8500](http://localhost:8500) for Consul UI

## 🎯 Learning Outcomes

### Learning Outcome 2: gRPC and Protocol Buffers ✅
- **Achievement**: Successfully implemented efficient inter-service communication using gRPC
- **Evidence**: Protocol Buffer definitions, generated Go code, and bi-directional service communication
- **Skills Gained**: Binary serialization, type-safe APIs, performance optimization

### Learning Outcome 4: Data Persistence and State Management ✅
- **Achievement**: Implemented isolated database architecture with GORM integration
- **Evidence**: Separate PostgreSQL instances, automatic schema migration, and CRUD operations
- **Skills Gained**: Database isolation patterns, ORM usage, connection management

### Learning Outcome 8: Observability Solutions ✅
- **Achievement**: Integrated Consul for service discovery and health monitoring
- **Evidence**: Service registration, health checks, and real-time service status monitoring
- **Skills Gained**: Service mesh patterns, health monitoring, distributed system observability

## 🔍 Advanced Features Implemented

### Dynamic Service Discovery Flow
1. **Request Reception**: API Gateway receives HTTP request
2. **Consul Query**: Gateway queries Consul for service location
3. **Address Resolution**: Consul returns current service address (container:port)
4. **gRPC Connection**: Gateway establishes connection to discovered service
5. **Business Logic**: Service processes request with database interaction
6. **Response Aggregation**: Gateway combines multiple service responses (if needed)
7. **Client Response**: Final result returned to client

### Database Architecture Benefits
- **Complete Isolation**: Each microservice owns its data
- **Independent Scaling**: Services can scale based on individual needs
- **Fault Tolerance**: Database failure in one service doesn't affect others
- **Technology Flexibility**: Different services can use different database technologies

## 🏁 Conclusion

This practical successfully demonstrates a production-ready microservices architecture with the following key achievements:

### ✅ **Technical Excellence**
- **Service Discovery**: Fully functional Consul integration replacing hardcoded connections
- **gRPC Communication**: High-performance inter-service communication
- **Database Isolation**: Complete separation of concerns and data ownership
- **Containerization**: Docker-based deployment with proper networking

### ✅ **Problem Resolution**
- **Fixed API Gateway**: Implemented dynamic service discovery
- **Composite Endpoint**: Successfully aggregating data from multiple services
- **Error Handling**: Robust error management and recovery
- **Performance**: Concurrent processing for efficient resource utilization

### ✅ **Best Practices**
- **Clean Architecture**: Separation of concerns and single responsibility
- **Type Safety**: Protocol Buffers for contract-first development
- **Observability**: Health monitoring and service registration
- **Scalability**: Horizontal scaling capabilities through service discovery

### 🚀 **Future Enhancements**
- Load balancing across service instances
- Circuit breaker pattern for fault tolerance
- Distributed tracing with tools like Jaeger
- API rate limiting and authentication
- Kubernetes deployment manifests

The implementation successfully addresses all practical requirements while demonstrating modern microservices patterns and cloud-native architecture principles. The system is production-ready and showcases enterprise-level distributed system design.

---
**Built with ❤️ using Go, gRPC, PostgreSQL, and Consul**

**Student**: Ranjung Yeshi Norbu  
**Module**: WEB303 Microservices & Serverless Applications  
**Practical**: 3 - Full-Stack Microservices with gRPC, Databases, and Service Discovery
