# 🚗 Ride-Hailing Service - Uber Backend Architecture

A microservices-based backend proof-of-concept (PoC) built to understand the system design and core mechanics behind ride-hailing applications like Uber, Lyft, and Rapido. 

This project demonstrates how to handle real-time geospatial data, event-driven communication, and strict ride state management using modern backend technologies.

---

## 🏗️ Architecture & Services Overview

This system is decoupled into three core microservices, each handling a specific domain:


| Service              | Port   | Responsibility                                                           |
| -------------------- | ------ | ------------------------------------------------------------------------ |
| **Location Service** | `8082` | Tracks real-time driver locations using Redis Geospatial indexing.       |
| **Ride Service**     | `8083` | Manages the ride lifecycle and publishes state changes to Kafka.         |
| **Matching Service** | `8084` | Consumes ride events, scores nearby drivers, and assigns the best match. |




### 🔄 System Flow

```text
Driver Phone → Location Service → Redis (GEOADD)

Rider App → Ride Service → Kafka (ride.requested)
                                     ↓
                        Matching Service (consumer)
                                     ↓
                        Location Service (find nearby drivers)
                                     ↓
                        Matching Algorithm (score drivers)
                                     ↓
                        Kafka (ride.matched)
                                     ↓
                        Ride Service (update ride with driver)
```

---



## 🚀 Tech Stack

- **Java 17 & Spring Boot 3** (REST APIs, Data JPA, Validation)
- **Redis** (Geospatial querying, high-speed driver tracking)
- **Apache Kafka** (Asynchronous event-driven messaging)
- **MySQL** (Persistent storage for ride history and users)
- **Docker & Docker Compose** (Containerized infrastructure)

---



## 🛠️ How To Run



### Step 1: Start Infrastructure

Run the supporting services (Redis, MySQL, Zookeeper, Kafka) via Docker Compose.

```bash
docker-compose up -d
```

*Note: Wait about 30 seconds for Kafka to fully initialize before starting the Spring Boot apps.*

### Step 2: Start Microservices

Open three separate terminal windows and start each service from the root of their respective directories:

**Location Service:**

```bash
cd location-service
mvn spring-boot:run
```

**Ride Service:**

```bash
cd ride-service
mvn spring-boot:run
```

**Matching Service:**

```bash
cd matching-service
mvn spring-boot:run
```

---



## 🧪 Testing the End-to-End Flow

You can use tools like Postman, cURL, or Insomnia to simulate the application flow.

### 1. Update Driver Locations (Location Service)

Simulate drivers coming online in a city.

```http
POST http://localhost:8082/api/v1/locations/drivers/update
{
    "driverId": "driver:1",
    "latitude": 12.9716,
    "longitude": 77.5946
}
```

*(Repeat with* `driver:2` *and* `driver:3` *using different coordinates to populate the map).*

### 2. Request a Ride (Ride Service)

Simulate a user booking a ride.

```http
POST http://localhost:8083/api/v1/rides/request
{
    "riderId": "rider:1",
    "pickupLatitude": 12.9716,
    "pickupLongitude": 77.5946,
    "pickupAddress": "MG Road, Bangalore",
    "dropLatitude": 12.9352,
    "dropLongitude": 77.6245,
    "dropAddress": "Koramangala, Bangalore"
}
```

*Response will return a ride ID and status* `MATCHING`*.*

### 3. Check Ride Status

Verify that the Matching Service successfully assigned a driver via Kafka.

```http
GET http://localhost:8083/api/v1/rides/{rideId}
```

*Status should now be* `ACCEPTED` *with a valid* `driverId` *assigned.*

### 4. Complete the Ride Lifecycle

Start the ride:

```http
PUT http://localhost:8083/api/v1/rides/{rideId}/start
```

Complete the ride:

```http
PUT http://localhost:8083/api/v1/rides/{rideId}/complete
```

---



## 🔍 Verifying Real-Time Data (Redis CLI)

You can directly inspect the geospatial indexes inside the Redis Docker container to see exactly how driver coordinates are stored:

```bash
docker exec -it redis-geo redis-cli

# See all stored drivers
ZRANGE drivers:locations 0 -1

# Check a specific driver's exact position
GEOPOS drivers:locations "driver:1"

# Calculate distance between two drivers
GEODIST drivers:locations "driver:1" "driver:2" km
```

---



## 🧠 Key System Design Concepts Showcased

- **Redis Geospatial Data Structures:** Utilizing `GEOADD` and `GEORADIUS` for sub-millisecond location queries instead of slow SQL queries.
- **Event-Driven Architecture:** Decoupling services using Kafka Producers and Consumers to prevent bottlenecks.
- **State Machine Patterns:** Managing strict object lifecycles (`REQUESTED` → `MATCHING` → `ACCEPTED` → `STARTED` → `COMPLETED`).
- **Algorithmic Matching:** Implementing weighted scoring systems (e.g., proximity + rating) for optimized driver dispatching.
- **Inter-Service Communication:** Combining async messaging (Kafka) with synchronous REST calls.

