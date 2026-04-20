# Library Manager Project

A library management system built with **Java 21**, **Quarkus**, **Maven**, **gRPC**, and **PostgreSQL**.

---

## Requirements

### Functional Requirements

- Manage library resources: **Bookings**, **Users**, **Products** (books), and **Orders**
- Expose a RESTful CRUD API for each domain
- Expose a gRPC API for inter-service communication (e.g., booking status notifications)
- Support creating, reading, updating, and cancelling bookings
- Associate bookings with users and specific library products (books)
- Persist all data in a relational database (PostgreSQL)

### Non-Functional Requirements

- Java 21 (LTS)
- Quarkus framework for fast startup and low memory footprint
- Maven for dependency management and build lifecycle
- REST API following OpenAPI 3.x specification
- gRPC for efficient binary communication between services
- PostgreSQL as the primary relational database
- Containerizable via Docker / Podman
- Health check endpoints (`/q/health`)

---

## Description

This project is a **library management backend** created with [Quarkus](https://quarkus.io/) and Java 21. It integrates two main communication layers:

1. **REST API** — exposes a full CRUD interface over the core library domains: Bookings, Users, Products, and Orders. The first implemented service covers the **Bookings** domain, allowing clients to create, retrieve, update, and cancel book reservations.

2. **gRPC API** — provides a high-performance binary protocol for internal or service-to-service communication, initially used for booking management operations.

Both services share a **PostgreSQL** database whose schema is managed via versioned SQL scripts.

---

## Source Code

| Component | Repository |
|---|---|
| REST API (Quarkus) | https://github.com/gandresAOQ/library-manager |
| gRPC API | https://github.com/gandresAOQ/booking-manager |
| Database scripts | https://github.com/gandresAOQ/library-database-scripts |

---

## Definitions

| Term | Meaning |
|---|---|
| **Booking** | A reservation linking a User to a Product (book) for a date range |
| **Product** | A library item (book) available for borrowing |
| **User** | A registered library member |
| **Order** | A checkout record associated with one or more bookings |

---

## Architecture

```mermaid
flowchart TD
    C["Clients\n(Browser / Mobile / Services)"]

    subgraph REST["REST Layer"]
        LM["library-manager\n(Quarkus REST)"]
    end

    subgraph gRPC["gRPC Layer"]
        BM["booking-manager\n(Quarkus gRPC)"]
        OS["other-services\n(Quarkus gRPC)"]
    end

    DB[("PostgreSQL DB\nlibrary_db")]

    C -->|"HTTP/REST"| LM
    LM -->|"gRPC (proto3)"| BM
    LM -->|"gRPC (proto3)"| OS
    BM --> DB
    OS --> DB
```

### Key Components

- **REST API** (`library-manager`): Entry point for all clients. Exposes JAX-RS resources over HTTP/JSON and delegates to downstream gRPC services.
- **gRPC services** (`booking-manager`, others): Domain-specific services called by `library-manager` via gRPC. Each service owns its domain logic and persists data to PostgreSQL.
- **Database** (`library-database-scripts`): Versioned SQL migration scripts (Flyway or Liquibase) shared by all gRPC services.

---

## How to Build and Run the Services

### Prerequisites

- Java 21+
- Maven 3.9+
- Docker / Podman (for PostgreSQL and containerized services)
- `grpcurl` (optional, for gRPC testing)

### 1. Start PostgreSQL

```bash
docker run --name library-db \
  -e POSTGRES_USER=library \
  -e POSTGRES_PASSWORD=library \
  -e POSTGRES_DB=library_db \
  -p 5432:5432 -d postgres:16
```

### 2. Apply Database Scripts

```bash
# Clone the database scripts repo and run migrations
git clone https://github.com/gandresAOQ/library-database-scripts
cd library-database-scripts
# Follow the README in that repo to apply migrations
```

### 3. Run the REST API

```bash
git clone https://github.com/gandresAOQ/library-manager
cd library-manager
./mvnw quarkus:dev
# Available at http://localhost:8080
```

### 4. Run the gRPC Service

```bash
git clone https://github.com/gandresAOQ/booking-manager
cd booking-manager
./mvnw quarkus:dev
# gRPC available at localhost:9000
```

---

## Example REST Requests

### Create a Booking

```bash
curl -X POST http://localhost:8080/api/bookings \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "productId": 42,
    "startDate": "2026-05-01",
    "endDate": "2026-05-15"
  }'
```

### Get All Bookings

```bash
curl http://localhost:8080/api/bookings
```

### Get a Booking by ID

```bash
curl http://localhost:8080/api/bookings/1
```

### Update a Booking

```bash
curl -X PUT http://localhost:8080/api/bookings/1 \
  -H "Content-Type: application/json" \
  -d '{
    "endDate": "2026-05-20"
  }'
```

### Cancel (Delete) a Booking

```bash
curl -X DELETE http://localhost:8080/api/bookings/1
```

---

## Proto File

The gRPC service contract is defined in both projects:

```protobuf
syntax = "proto3";
option java_package = "com.booking.grpc";

service BookingService {
  rpc Get(BookingRequestId) returns (BookingResponse);
  rpc GetAll(BookingRequestPaged) returns (BookingResponse);
  rpc Post(BookingRequest) returns (BookingResponse);
  rpc Put(BookingRequest) returns (BookingResponse);
  rpc Delete(BookingRequestId) returns (BookingResponse);
}

message BookingRequestPaged {
  string uuid = 1;
  string page = 2;
}

message BookingRequestId {
  string uuid = 1;
  string id = 2;
}

message BookingRequest {
  string uuid = 1;
  string id = 2;
  string name = 3;
  string author = 4;
  string price = 5;
  string language = 6;
  string pages = 7;
  string format = 8;
}

message BookingResponse {
  int32 status = 1;
  string payload = 2;
}
```

### gRPC Example Calls (`grpcurl`)

```bash
# Create a booking
grpcurl -plaintext -d '{"user_id":1,"product_id":42,"start_date":"2026-05-01","end_date":"2026-05-15"}' \
  localhost:9000 booking.BookingService/CreateBooking

# Get a booking by ID
grpcurl -plaintext -d '{"booking_id":1}' \
  localhost:9000 booking.BookingService/GetBooking

# List all bookings
grpcurl -plaintext -d '{}' \
  localhost:9000 booking.BookingService/ListBookings

# Update a booking
grpcurl -plaintext -d '{"booking_id":1,"end_date":"2026-05-20","status":"CONFIRMED"}' \
  localhost:9000 booking.BookingService/UpdateBooking

# Cancel a booking
grpcurl -plaintext -d '{"booking_id":1}' \
  localhost:9000 booking.BookingService/CancelBooking
```

---

## Design Decisions and Trade-offs

| Decision | Rationale | Trade-off |
|---|---|---|
| **Quarkus over Spring Boot** | Faster startup, lower memory usage, native image support | Smaller ecosystem; less community documentation than Spring |
| **gRPC alongside REST** | Efficient binary protocol for internal calls; strongly typed contracts via proto files | Extra complexity; requires protobuf tooling |
| **PostgreSQL** | Strong ACID guarantees, rich JSON support, mature tooling | Heavier than embedded DBs; requires a running instance for dev |
| **Separate repos per service** | Clear ownership boundaries; independent deployment and versioning | More overhead managing multiple repos and coordinating releases |
| **Bookings as first iteration** | Core domain that exercises all layers (REST, DB, gRPC) end-to-end | Other domains (Users, Products, Orders) are initially stubs |
