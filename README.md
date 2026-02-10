# Idempotent Event-Driven Order Processing System

## Overview

This project implements a robust, fault-tolerant **Event-Driven Architecture (EDA)** for processing high-throughput order events. It leverages **Apache Kafka** for asynchronous message streaming, **FastAPI** for high-performance ingestion, and **PostgreSQL** for persistent storage.

The core objective of this system is to guarantee **Exactly-Once Processing** (Idempotency). It ensures that even in the face of network failures, service crashes, or Kafka message redelivery, every order is processed and stored exactly once, maintaining strict data consistency.

### Key Features
* **Asynchronous Decoupling:** Producer and Consumer are completely isolated via Kafka.
* **Database-Level Idempotency:** robust handling of duplicate messages using transactional unique constraints.
* **Schema Validation:** Strict Pydantic enforcement prevents malformed data from entering the pipeline.
* **Resilience:** Dockerized architecture with health checks and automatic restarts.

---

## Technology Stack

* **Language:** Python 3.10+
* **Frameworks:** FastAPI, Pydantic, SQLAlchemy (Async)
* **Streaming:** Apache Kafka, Zookeeper, `aiokafka`
* **Database:** PostgreSQL (Async via `asyncpg`)
* **Infrastructure:** Docker & Docker Compose
* **Testing:** Pytest

---

## Setup & Installation

### Prerequisites
* [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) installed.

### 1. Clone the Repository
```bash
git clone <repository-url>
cd kafka-eda-order-system
```

## 2. Environment Configuration

The project includes a template file **`.env.example`** with safe default values.  
Copy this file to create your active environment configuration.

```bash
# Linux / macOS / PowerShell
cp .env.example .env

# Windows (CMD)
copy .env.example .env
```

> **Note:**  
> The default values in `.env` are configured to work **out-of-the-box** with the Docker network aliases.

---

## 3. Start Services

Run the entire stack (**Kafka, Zookeeper, Database, Producer, Consumer**) using a single command:

```bash
docker-compose up --build -d
```

### Command Options

- **`--build`**: Rebuilds the Python Docker images  
- **`-d`**: Runs containers in detached (background) mode  

Allow **30–60 seconds** for Kafka and PostgreSQL to fully initialize.

You can verify the running services with:

```bash
docker ps
```

Ensure all services show a **healthy** status before proceeding.

---

## API Documentation

### Ingest Order Event

Publishes a new **order event** to the Kafka topic.  
The request is **synchronously validated** but **processed asynchronously** by consumers.

---

### Endpoint
`POST /events`


### Service Port
`8000`


### Full URL
`http://localhost:8000/events`


---

### Request Body

The payload must **strictly adhere** to the defined schema.

```json
{
  "event_id": "123e4567-e89b-12d3-a456-426614174000",
  "order_id": "78901234-5678-90ab-cdef-1234567890ab",
  "user_id": "user_123",
  "total_amount": 99.99,
  "timestamp": "2023-10-27T10:00:00Z"
}
```

### Responses

| Status Code | Description                                              | Response Body                              |
|------------|----------------------------------------------------------|--------------------------------------------|
| **202 Accepted** | Event validated and queued for processing              | `{ "status": "accepted", "event_id": "..." }` |
| **422 Unprocessable Entity** | Invalid schema (e.g., negative amount, invalid UUID) | `{ "detail": [ ... ] }` |
| **500 Internal Server Error** | Kafka producer failure (e.g., broker down)          | `{ "detail": "Internal Server Error" }` |

---

## Health Checks

### Producer Service
`GET http://localhost:8000/health`

### Consumer Service
`GET http://localhost:8001/health`

---

## Testing

We follow a **Testing Pyramid** strategy, combining fast **Unit Tests** with comprehensive **End-to-End Integration Tests**.

---

## 1. Local Unit Tests

These tests **mock infrastructure components** (Kafka, Database) to quickly validate business logic in isolation.

### Setup Virtual Environment (Optional)

```bash
python -m venv venv
source venv/bin/activate   # Linux / macOS
# venv\Scripts\activate    # Windows

# Install Test Dependencies
pip install pytest pytest-asyncio httpx requests sqlalchemy asyncpg aiokafka

# Run Tests
pytest tests/test_producer_api.py tests/test_consumer_logic.py -v
```

## 2. Integration Tests (End-to-End)

These tests require the **Docker stack to be running**.  
They simulate real user behavior by sending requests through the API and verifying that data is persisted correctly in the database.

### Start Required Services

```bash
# Ensure Docker services are running
docker-compose up -d
```

## Idempotency Strategy

In distributed systems, **At-Least-Once delivery** is the standard guarantee provided by **Kafka**.  
This means a consumer may receive the **same message multiple times** (for example, if it crashes before committing the offset).

To prevent data corruption (e.g., charging a user twice), this system implements **Transactional Deduplication at the database layer**.

---

### The Algorithm

#### 1. Unique Identifier
- Every event carries a generic UUID: `event_id`

#### 2. Database Constraint
- The `processed_events` table in PostgreSQL defines:
  - `event_id` as a **PRIMARY KEY**

#### 3. Atomic Insert Logic

1. The consumer attempts to **INSERT** the event payload
2. **If successful**
   - The event is new
   - Business logic executes
   - Kafka offset is committed
3. **If failed (IntegrityError)**
   - The event has already been processed
   - The consumer logs a warning: `"Duplicate skipped"`
   - Kafka offset is committed to mark the message as handled

---

### Why This Approach Is Superior

This strategy is **fully atomic**.

It avoids race conditions inherent in **Check-Then-Act** patterns (`SELECT` → `INSERT`), where multiple consumers could see “no record” and insert duplicates simultaneously under high concurrency.

By relying on **database-enforced uniqueness**, correctness is guaranteed even with parallel consumers.

---

## Architectural Decisions & Trade-offs

### 1. PostgreSQL vs MongoDB
**Decision:** PostgreSQL  

**Rationale:**  
While NoSQL databases are common for event storage, this system requires **strict ACID guarantees** to enforce idempotency via unique constraints. PostgreSQL enforces uniqueness natively and deterministically, making deduplication simpler and safer than eventual consistency models.

---

### 2. Async Python (FastAPI + AIOKafka)
**Decision:** Asynchronous I/O  

**Rationale:**  
Event processing is primarily **I/O-bound** (Kafka network I/O, database writes).  
Using Python’s `asyncio` allows a single consumer process to handle high concurrency without blocking, reducing **consumer lag** during database spikes.

---

### 3. Pydantic for Schema Validation
**Decision:** Strict schema validation at ingestion  

**Rationale:**  
> “Garbage in, garbage out.”

By validating events at the **Producer API level** using Pydantic, malformed or invalid events are prevented from ever reaching Kafka. This protects consumers from crashes caused by missing fields or incorrect data types.

---

### 4. Docker Health Checks
**Decision:** `depends_on` with `service_healthy`  

**Rationale:**  
A common Docker Compose failure mode is services starting before dependencies are ready.  
Health checks were implemented for **Kafka and PostgreSQL**, ensuring that Producer and Consumer services start only after all dependencies are fully operational.

---

## Technical Implementation Details

### 1. API & Data Validation
| Feature | Implementation Detail |
| :--- | :--- |
| **RESTful Event Ingestion** | Implemented a high-performance `POST /events` endpoint using **FastAPI**. |
| **Strict Schema Enforcement** | Utilizes **Pydantic** models to validate data types and constraints before ingestion. Invalid payloads are rejected immediately (422). |
| **Asynchronous Response** | API returns `202 Accepted` to acknowledge receipt without blocking the client, adhering to async design patterns. |

### 2. Message Streaming (Kafka)
| Feature | Implementation Detail |
| :--- | :--- |
| **Non-blocking Producer** | Uses `aiokafka` to publish events to the `order_events` topic asynchronously, ensuring low latency. |
| **Scalable Consumer** | Implements a continuous, non-blocking consumer loop that processes messages using Python's `asyncio` event loop. |

### 3. Idempotency & Persistence
| Feature | Implementation Detail |
| :--- | :--- |
| **Exactly-Once Processing** | Achieved via **Database-Level Idempotency**. The Consumer catches `IntegrityError` (Unique Constraint Violation) to safely ignore duplicate messages from Kafka. |
| **ACID Persistence** | State is persisted to **PostgreSQL** using `SQLAlchemy` (Async) for reliable transactional integrity. |
| **Optimized Schema** | The database schema enforces the `event_id` as the `PRIMARY KEY`, guaranteeing physical data uniqueness. |

### 4. Reliability & Fault Tolerance
| Feature | Implementation Detail |
| :--- | :--- |
| **Graceful Error Handling** | The consumer loop is protected by robust `try...except` blocks to log errors and prevent crash loops. |
| **Retry Strategy** | Implements basic retry logic (wait-and-retry) for transient failures (e.g., temporary DB connection loss). |
| **Health Monitoring** | Both services expose `/health` endpoints. The Consumer runs a background HTTP server specifically for orchestration checks. |

### 5. DevOps & Infrastructure
| Feature | Implementation Detail |
| :--- | :--- |
| **Containerization** | Custom `Dockerfile` builds for Producer and Consumer services. |
| **Orchestration** | `docker-compose.yml` manages the full lifecycle of Zookeeper, Kafka, Postgres, and application services. |
| **Configuration Management** | Strict separation of config from code using `.env` files and `pydantic-settings`. No hardcoded secrets. |