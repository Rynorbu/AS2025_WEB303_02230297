# WEB303 Practical 1: Microservices with gRPC and Docker

## Student Information
- **Student ID**: 02230297
- **Repository**: [AS2025_WEB303_02230297_Practical1](https://github.com/Rynorbu/AS2025_WEB303_02230297_Practical1)


## Overview

This practical demonstrates the implementation of communicating microservices using Go, gRPC, Protocol Buffers, and Docker Compose. The project consists of two microservices:

1. **Time Service**: Provides current timestamp functionality
2. **Greeter Service**: Provides personalized greetings with current time by calling the Time Service

## Architecture

```
┌─────────────────┐    gRPC Call    ┌─────────────────┐
│                 │ ──────────────► │                 │
│ Greeter Service │                 │  Time Service   │
│   Port: 50051   │ ◄────────────── │   Port: 50052   │
│                 │   Time Response │                 │
└─────────────────┘                 └─────────────────┘
        │                                    │
        │                                    │
        ▼                                    ▼
   Public API                         Internal Service
   (External Access)                  (Container Network)
```



## Implementation Approach

The implementation of this practical followed a systematic approach focused on building a robust microservices architecture from the ground up. The primary goal was to create two independent services that could communicate efficiently using gRPC while being containerized and orchestrated through Docker Compose.

### 1. Service Contracts (Protocol Buffers)

The first step was defining clear service contracts using Protocol Buffers. This approach ensures type safety and provides a language-agnostic way to define service interfaces. By starting with the contract definitions, I established a clear understanding of what each service would provide before writing any implementation code.

#### Time Service Contract (`proto/time.proto`)
```protobuf
syntax = "proto3";
option go_package = "practical-one/proto/gen;gen";
package time;

service TimeService {
  rpc GetTime(TimeRequest) returns (TimeResponse);
}

message TimeRequest {}
message TimeResponse {
  string current_time = 1;
}
```

The Time Service was designed to be simple and focused, providing only one function: returning the current server time. This follows the microservices principle of having services that do one thing well.

#### Greeter Service Contract (`proto/greeter.proto`)
```protobuf
syntax = "proto3";
option go_package = "practical-one/proto/gen;gen";
package greeter;

service GreeterService {
  rpc SayHello(HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}
message HelloResponse {
  string message = 1;
}
```

The Greeter Service accepts a name parameter and returns a personalized greeting. While this seems straightforward, the real complexity comes from its dependency on the Time Service, which demonstrates inter-service communication.

### 2. Code Generation

After defining the contracts, I used the Protocol Buffer compiler to generate Go code. This automated code generation is one of the major advantages of using gRPC, as it eliminates manual serialization code and reduces the potential for errors.

```bash
protoc --go_out=./proto/gen --go_opt=paths=source_relative \
       --go-grpc_out=./proto/gen --go-grpc_opt=paths=source_relative \
       proto/*.proto
```

The generated code includes both the message structures and the gRPC service interfaces, providing a foundation for implementing the actual service logic.

### 3. Service Implementation

#### Time Service
- **Port**: 50052 (internal)
- **Functionality**: Returns current timestamp in RFC3339 format
- **Dependencies**: Standard Go libraries + gRPC
- **Design Decision**: Kept this service internal only, not exposing it to the host machine, as it should only be called by other services within the Docker network

The Time Service implementation was straightforward but required careful consideration of time formatting to ensure consistency across different systems and time zones.

#### Greeter Service
- **Port**: 50051 (exposed)
- **Functionality**: 
  - Accepts greeting requests with a name
  - Makes gRPC call to Time Service
  - Returns combined greeting with current time
- **Dependencies**: gRPC client for Time Service communication
- **Design Decision**: This service acts as the public-facing API, demonstrating how one microservice can orchestrate calls to other services

The Greeter Service required implementing both server and client functionality. As a server, it handles incoming greeting requests. As a client, it makes requests to the Time Service. This dual role demonstrates a common pattern in microservices architectures.

### 4. Containerization Strategy

#### Multi-stage Docker Builds
Both services use optimized multi-stage Dockerfiles to minimize the final image size and improve security:
1. **Builder Stage**: golang:1.24-alpine for compilation with all build tools
2. **Runtime Stage**: alpine:latest for minimal production image containing only the compiled binary

This approach significantly reduces the attack surface and deployment size. The builder stage includes all the tools needed to compile Go code, while the runtime stage includes only the essential libraries needed to run the binary.

#### Service Communication
- Services communicate via Docker internal network using service names as hostnames
- Greeter service connects to `time-service:50052` hostname
- Only Greeter service (port 50051) exposed to host machine for external access
- Internal service discovery handled automatically by Docker Compose networking

The networking configuration ensures that services can find each other reliably while maintaining security through network isolation.

### 5. Orchestration with Docker Compose
```yaml
version: '3.8'
services:
  time-service:
    build:
      context: .
      dockerfile: time-service/Dockerfile
    hostname: time-service

  greeter-service:
    build:
      context: .
      dockerfile: greeter-service/Dockerfile
    hostname: greeter-service
    ports:
      - "50051:50051"
    depends_on:
      - time-service
```

Docker Compose provides a declarative way to define the entire application stack. The configuration includes build contexts, networking, port mapping, and service dependencies. The `depends_on` directive ensures that the Time Service starts before the Greeter Service, although in a production environment, additional health checks would be necessary to ensure the Time Service is fully ready before the Greeter Service attempts to connect.

## Steps Taken

### Phase 1: Environment Setup
1.  Installed and configured Go 1.24.5
2.  Installed Protocol Buffer compiler and Go plugins
3.  Set up Docker Desktop
4.  Verified installations with test commands

### Phase 2: Project Setup
1.  Created directory structure
2.  Defined service contracts in Protocol Buffers
3.  Generated Go code from proto definitions
4.  Initialized Go modules for each service

### Phase 3: Service Implementation
1.  Implemented Time Service with gRPC server
2.  Implemented Greeter Service with gRPC server and client
3.  Configured inter-service communication
4.  Added proper error handling and logging

### Phase 4: Containerization
1.  Created optimized Dockerfiles for both services
2.  Set up Docker Compose for orchestration
3.  Configured service dependencies and networking
4.  Exposed appropriate ports

### Phase 5: Testing and Validation
1.  Built and deployed services using Docker Compose
2.  Installed and configured grpcurl for testing
3.  Verified end-to-end functionality
4.  Confirmed inter-service communication

## Challenges Encountered and Solutions

Throughout the development of this microservices architecture, I encountered several technical challenges that required careful problem-solving and a deeper understanding of Go modules, Docker, and gRPC. Each challenge provided valuable learning opportunities.

### 1. Go Version Compatibility Issue

**Problem**: The initial Dockerfiles specified Go version 1.24.2 in the base image, but the go.mod files required Go 1.24.5 or higher. This created a version mismatch that prevented the services from building inside Docker containers.

```
go: go.mod requires go >= 1.24.5 (running go 1.24.2; GOTOOLCHAIN=local)
```

**Root Cause**: I had initially pinned a specific Go version in the Dockerfile without considering that the go.mod file might require a newer version. This highlighted the importance of version compatibility across the entire build pipeline.

**Solution**: I updated the Dockerfiles to use `golang:1.24-alpine` instead of `golang:1.24.2-alpine`. This change allows Docker to automatically pull the latest patch version within the 1.24 series, ensuring compatibility with the go.mod requirements. This approach also makes the Dockerfile more maintainable since it will automatically use newer patch versions that include bug fixes and security updates.

**Lesson Learned**: When working with versioned tools, it is important to balance between pinning specific versions for reproducibility and allowing flexibility for patch updates. Using minor version tags with automatic patch updates is often the best compromise.

### 2. Module Import Path Resolution

**Problem**: During the Docker build process, the Go compiler could not locate the generated protobuf code, resulting in import path errors.

```
package practical-one/proto/gen is not in std
```

**Root Cause**: Initially, I had created separate go.mod files for each service, which meant that the shared protobuf code in the proto directory was not accessible to both services. The Go module system needs clear paths to resolve dependencies, and the fragmented module structure created ambiguity.

**Solution**: I restructured the project to use a single unified go.mod file at the root level that encompasses all services and the shared proto directory. I then added a replace directive to explicitly map the proto package:

```
replace practical-one/proto/gen => ./proto/gen/proto
```

This solution required updating the Dockerfiles to work from the root context and copy the unified go.mod and go.sum files. The replace directive tells Go exactly where to find the proto package during both local development and Docker builds.

**Lesson Learned**: In microservices projects with shared code, careful consideration must be given to module structure. A monorepo approach with a single go.mod can simplify dependency management for shared libraries, even when services are deployed independently.

### 3. Docker Build Context Issues

**Problem**: When building Docker images, the build process could not access the protobuf files because they were outside the build context of individual service directories.

**Root Cause**: Docker builds work within a specific context directory, and files outside this context are not accessible. My initial approach of having each service build from its own directory meant that the shared proto directory was inaccessible.

**Solution**: I reconfigured the Docker Compose file to use the project root as the build context for all services:

```yaml
build:
  context: .
  dockerfile: time-service/Dockerfile
```

This change, combined with the unified module approach, ensured that all necessary files including the proto directory, go.mod, and go.sum were available during the build process. I also updated the Dockerfiles to copy files from the appropriate paths relative to the root context.

**Lesson Learned**: Docker build contexts must be carefully planned when working with shared code. Using the project root as the context provides flexibility but requires careful Dockerfile construction to avoid copying unnecessary files into the image.

### 4. Dependency Checksum Mismatches

**Problem**: The go.sum file contained checksums that did not match the actual dependencies being downloaded during the Docker build, causing the build to fail with verification errors.

**Root Cause**: This issue arose after restructuring the module system. The go.sum file had become stale and contained checksums from the previous module structure that were no longer valid.

**Solution**: I regenerated the go.sum file from scratch by running the following commands:

```bash
go mod tidy
go mod download
```

These commands ensured that go.sum contained accurate checksums for all current dependencies. I then committed the updated go.sum file to ensure reproducible builds across different environments.

**Lesson Learned**: The go.sum file is critical for reproducible builds and security. Whenever the module structure or dependencies change significantly, regenerating go.sum ensures integrity. This file should always be committed to version control.

### 5. Service Discovery in Containers

**Problem**: The Greeter Service was attempting to connect to the Time Service using `localhost:50052`, which worked during local development but failed when running in Docker containers because each container has its own network namespace.

**Root Cause**: In a containerized environment, localhost refers to the container itself, not the host machine or other containers. This is a common mistake when transitioning from local development to containerized deployment.

**Solution**: I updated the connection string in the Greeter Service to use the Docker service name as the hostname:

```go
conn, err := grpc.Dial("time-service:50052", grpc.WithInsecure())
```

Docker Compose automatically creates a network where services can discover each other using their service names as DNS hostnames. This approach is more robust than using IP addresses and works seamlessly with Docker's built-in service discovery.

**Lesson Learned**: Containerized applications require different networking considerations than traditional applications. Service discovery mechanisms like Docker's DNS-based approach simplify inter-service communication and should be leveraged rather than hardcoding IP addresses or using localhost.

### 6. gRPC Connection Management

**Problem**: During testing, I initially experienced intermittent connection failures when the Greeter Service tried to call the Time Service immediately after startup.

**Root Cause**: Although Docker Compose's `depends_on` ensures that containers start in order, it does not wait for a service to be fully ready to accept connections. The Time Service gRPC server needs a few milliseconds to fully initialize and start listening on the port.

**Solution**: While I implemented basic retry logic in the code, the more robust solution would involve implementing health checks in the Docker Compose configuration. For this practical, the simple retry mechanism was sufficient, but I documented this as an area for improvement in a production environment.

**Lesson Learned**: Container orchestration tools start containers but do not guarantee application readiness. Production systems should implement health checks and retry logic to handle timing issues gracefully. This becomes even more critical in distributed systems where network latency and startup times can vary.

## Evidence and Screenshots

This section provides visual evidence of the successful implementation and testing of the microservices.

### 1. Docker Compose Build Process
The following screenshots show the successful build and deployment of both microservices using Docker Compose.

#### Initial Build Process
![Docker Compose Build - Part 1](./assets/compose1.png)
*Screenshot showing the Docker Compose build process starting, including image pulling and dependency resolution.*

#### Build Completion and Service Startup
![Docker Compose Build - Part 2](./assets/compose2.png)
*Screenshot showing the successful completion of the build process and both services starting up with their respective logs.*

### 2. Running Containers Verification
![Docker Container Details](./assets/docker.png)
*Screenshot showing both microservices running successfully in Docker containers with proper port mappings and status.*

### 3. gRPC Service Testing
![gRPC Call Response](./assets/message.png)

*Screenshot demonstrating successful gRPC communication between services, showing the greeter service calling the time service and returning a combined response with greeting and current timestamp.*

## Testing Results

### Successful gRPC Communication Test
```bash
$ grpcurl -plaintext -import-path ./proto -proto greeter.proto \
  -d '{"name": "WEB303 Student"}' localhost:50051 greeter.GreeterService/SayHello

Response:
{
  "message": "Hello WEB303 Student! The current time is 2025-10-04T03:30:03Z"
}
```

### Service Status Verification
```bash
$ docker-compose ps
NAME                              SERVICE           STATUS    PORTS
practical-one-greeter-service-1   greeter-service   Up        0.0.0.0:50051->50051/tcp
practical-one-time-service-1      time-service      Up        50052/tcp
```

## Key Learning Outcomes Achieved

### Learning Outcome 2: Microservices with gRPC
Successfully designed and implemented two microservices using gRPC and Protocol Buffers
- Efficient binary serialization
- Strong typing with generated code
- Bi-directional streaming capabilities (foundation set)

### Learning Outcome 6: Container Orchestration
Deployed microservices using Docker Compose as foundation for Kubernetes
- Multi-container application management
- Service discovery and networking
- Dependency management between services

### Learning Outcome 1: Microservices Concepts
Demonstrated fundamental microservices principles
- Service separation and independence
- Inter-service communication protocols
- Container-based deployment strategies

## What I Learned

This practical provided hands-on experience with several critical technologies and concepts that form the foundation of modern microservices architecture. Beyond simply completing the technical requirements, I gained deeper insights into how distributed systems work and the challenges involved in building them.

### Understanding gRPC and Protocol Buffers

Before this practical, I had only theoretical knowledge of gRPC. Working with Protocol Buffers taught me the value of contract-first development. By defining the service interface before writing any implementation code, I was forced to think carefully about what data each service would need and what it would return. This approach prevents many integration issues that arise when services are developed without clear contracts.

The code generation aspect of Protocol Buffers was particularly enlightening. Instead of manually writing serialization code, the protoc compiler generates type-safe code in Go. This eliminates entire classes of bugs related to data serialization and ensures that both the client and server are always in sync regarding message structure. I learned that this strong typing across service boundaries is one of the key advantages of gRPC over traditional REST APIs with JSON.

### Microservices Architecture Principles

Implementing two services that communicate with each other demonstrated several important architectural principles. The Time Service is intentionally simple and focused on a single responsibility. This taught me that microservices should be designed to do one thing well rather than trying to handle multiple concerns.

The Greeter Service, while also simple, illustrates how services in a microservices architecture often depend on other services. Learning to manage this dependency through gRPC calls showed me how service orchestration works in practice. I also learned that service boundaries should be designed carefully because inter-service communication has performance implications compared to in-process function calls.

### Docker and Containerization

This practical deepened my understanding of Docker beyond just running pre-built containers. Creating multi-stage Dockerfiles taught me about optimizing container images for production. The builder stage includes everything needed to compile the application, while the runtime stage contains only the minimal components needed to run it. This reduces the final image size from hundreds of megabytes to tens of megabytes and significantly improves security by reducing the attack surface.

I learned that containerization is not just about packaging applications but about creating consistent, reproducible environments. The fact that the same Docker image runs identically on my development machine and any production server eliminates the common problem of "it works on my machine" bugs.

### Docker Compose and Service Orchestration

Working with Docker Compose gave me practical experience with container orchestration. I learned that orchestration is about more than just starting containers; it involves managing networks, dependencies, and communication between services. The automatic DNS-based service discovery that Docker Compose provides simplified inter-service communication significantly.

The `depends_on` directive taught me about service startup ordering, though I also learned its limitations. Just because a container has started does not mean the application inside is ready to accept connections. This insight is crucial for building robust distributed systems and points toward more advanced orchestration tools like Kubernetes that provide health checks and readiness probes.

### Go Module System

Dealing with the module path resolution challenges taught me a great deal about Go's module system. I learned that Go modules are more than just dependency management; they define how code is organized and imported. The replace directive is a powerful tool for local development and monorepo setups, allowing you to override import paths to use local code instead of downloaded dependencies.

I also learned about the importance of go.sum for security and reproducibility. This file contains cryptographic checksums of dependencies, ensuring that the same code is used across different builds and preventing supply chain attacks where a dependency might be replaced with malicious code.

### Debugging Containerized Applications

Troubleshooting issues in containerized environments required developing new debugging skills. I learned to use docker logs to view container output, docker exec to inspect running containers, and grpcurl to test gRPC services. These tools are essential for diagnosing issues in environments where traditional debugging approaches do not work.

The networking challenges taught me to think differently about how applications connect to each other. Understanding that localhost means different things in different contexts (the host machine versus inside a container) was crucial for getting the services to communicate properly.

### Error Handling and Resilience

Although not explicitly required, implementing basic error handling taught me about the importance of resilience in distributed systems. Network calls can fail, services can be unavailable, and timing issues can cause intermittent problems. Learning to handle these scenarios gracefully is essential for building production-ready microservices.

### Development Workflow and Tooling

This practical exposed me to a complete development workflow for microservices. From defining contracts with Protocol Buffers, to implementing services in Go, to containerizing with Docker, to orchestrating with Docker Compose, I experienced the entire pipeline. I also learned to use tools like grpcurl for testing, which is essential for working with gRPC services that cannot be tested with simple HTTP requests.

### Version Compatibility and Dependency Management

The Go version compatibility issue taught me an important lesson about managing dependencies across different environments. Development tools, build systems, and runtime environments all need to be aligned on compatible versions. This experience highlighted the importance of clear documentation and careful version management in projects.

## Conclusion

This practical successfully achieved all stated objectives and provided a solid foundation in microservices architecture, gRPC communication, and containerized deployment. The implementation demonstrates a functional distributed system where two independent services communicate efficiently using industry-standard protocols and run in isolated, reproducible container environments.

### Objectives Achieved

**Primary Objective - Microservices Implementation**: Successfully implemented two microservices in Go that communicate using gRPC. The Time Service provides timestamp functionality, and the Greeter Service orchestrates calls to create personalized greetings with timestamps. This demonstrates the fundamental pattern of service composition in microservices architectures.

**Learning Outcome 2 - gRPC and Protocol Buffers**: The practical fully achieved this objective by implementing type-safe service contracts using Protocol Buffers and efficient binary communication using gRPC. The code generation workflow and client-server implementation demonstrate practical mastery of these technologies.

**Learning Outcome 6 - Container Orchestration**: Successfully containerized both services using Docker and orchestrated them using Docker Compose. The implementation includes proper networking, service discovery, dependency management, and port mapping. This provides the foundational knowledge needed to progress to more advanced orchestration platforms like Kubernetes.

**Learning Outcome 1 - Microservices Concepts**: The practical demonstrates core microservices principles including service independence, inter-service communication, and container-based deployment. The architecture clearly shows separation of concerns and the benefits of having focused, single-purpose services.

### Technical Achievements

The final implementation successfully demonstrates several technical achievements beyond the basic requirements. The multi-stage Docker builds produce optimized container images suitable for production deployment. The unified Go module structure provides a maintainable codebase that can scale to accommodate additional services. The networking configuration properly isolates internal services while exposing only the necessary public endpoints.

All services build successfully from source, deploy reliably using Docker Compose, and communicate effectively through gRPC. The implementation has been thoroughly tested using grpcurl, confirming that the end-to-end flow works correctly from external request through service orchestration to internal service calls.

### Challenges Overcome

The practical involved overcoming several significant technical challenges, each of which contributed to a deeper understanding of the technologies involved. The Go version compatibility issue reinforced the importance of version management across build pipelines. The module path resolution challenges provided practical experience with Go's module system and monorepo patterns. The Docker build context issues taught valuable lessons about containerization strategies for projects with shared code.

Each challenge required research, experimentation, and systematic problem-solving. The solutions implemented are not just workarounds but represent best practices for structuring microservices projects. These experiences have prepared me to handle similar issues in more complex projects.

### Skills Developed

This practical developed both technical and problem-solving skills. On the technical side, I gained practical experience with Go programming, gRPC implementation, Protocol Buffers, Docker containerization, and Docker Compose orchestration. I learned to use tools like protoc for code generation and grpcurl for gRPC testing.

More importantly, I developed problem-solving approaches for distributed systems. Learning to debug containerized applications, understand networking in Docker environments, and manage dependencies across services are skills that transfer to any microservices project. The experience of working through build failures and configuration issues built resilience and systematic debugging skills.

### Foundation for Future Work

This practical establishes a strong foundation for more advanced topics in cloud-native development. The concepts learned here scale directly to production systems using Kubernetes for orchestration, service meshes for advanced networking, and observability tools for monitoring distributed systems.

The experience with Docker Compose will make it easier to understand Kubernetes concepts since both deal with container orchestration, networking, and service discovery, though at different scales. The gRPC skills are directly applicable to building high-performance microservices in production environments.

### Reflection on the Learning Process

The practical reinforced that learning distributed systems requires hands-on experience. Reading about microservices and gRPC provides theoretical knowledge, but actually encountering and solving real problems like service discovery, module path resolution, and container networking creates lasting understanding.

The iterative process of encountering problems, researching solutions, implementing fixes, and testing results mirrors real-world software development. Each challenge improved my ability to diagnose issues, search for solutions effectively, and apply fixes systematically.

### Final Thoughts

This practical was a comprehensive introduction to modern microservices development. It combined multiple technologies into a cohesive project that demonstrates how distributed systems are built in practice. The skills and knowledge gained will be directly applicable to future projects involving microservices, containerization, and cloud-native development.

The successful completion of this practical confirms readiness to progress to more advanced topics like Kubernetes orchestration, service mesh implementations, and production-grade microservices patterns. The foundation established here in contract-first development, containerization, and inter-service communication will support continued growth in cloud-native software engineering.

## Dependencies and Technologies Used

- **Go 1.24.5**: Primary programming language
- **gRPC**: Inter-service communication framework
- **Protocol Buffers**: Interface definition and serialization
- **Docker & Docker Compose**: Containerization and orchestration
- **Alpine Linux**: Minimal container base images
- **grpcurl**: Testing and debugging gRPC services

## Usage Instructions

### Prerequisites

- Docker and Docker Compose installed
- Go 1.24+ (for local development)
- grpcurl (for testing)

### Running the Application

1. Clone the repository
2. Navigate to project directory
3. Build and start services:
   ```bash
   docker-compose up --build
   ```

### Testing the Services

```bash
# Test Greeter Service (includes Time Service call)
grpcurl -plaintext -import-path ./proto -proto greeter.proto \
  -d '{"name": "Your Name"}' localhost:50051 greeter.GreeterService/SayHello
```

## Repository Links

- **GitHub Repository**: https://github.com/Rynorbu/AS2025_WEB303_02230297_Practical1
- **Practical Requirements**: https://github.com/douglasswmcst/ss2025_web303/blob/master/practicals/practical1.md

---

**Note**: This implementation successfully fulfills all requirements of WEB303 Practical 1, demonstrating foundational microservices concepts, gRPC communication, and containerized deployment strategies.
