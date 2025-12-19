
# Spring-Idempotent-Kafka-Consumer-Persistence

## Overview

This project is a **multi-microservice demonstration** of **idempotent Kafka consumption** with **persistent deduplication** using a relational database (PostgreSQL). It ensures **exactly-once processing semantics** in the presence of retries, rebalances, or consumer crashes — a critical requirement in financial systems where duplicate actions can have severe consequences.

The core focus is on combining Kafka's at-least-once delivery with application-level idempotency to achieve safe, reliable processing.

## Real-World Scenario

In financial trading apps like **Robinhood** or brokerage platforms:
- Trade execution messages are delivered via Kafka (e.g., "execute buy order").
- Kafka guarantees **at-least-once** delivery — duplicates can occur due to retries or rebalances.
- Executing the same trade twice would result in incorrect positions and financial loss.
- Consumers must detect and skip duplicates using a **persistent store** (DB) to survive restarts.

This project implements a robust idempotency layer using a deduplication table, ensuring trades are executed **exactly once** even under failure conditions.

## Microservices Involved

| Service                     | Responsibility                                                                 | Port  |
|-----------------------------|--------------------------------------------------------------------------------|-------|
| **kafka-cluster**           | 3-broker Kafka cluster (Dockerized)                                            | 9092  |
| **zookeeper**               | Coordination for Kafka                                                         | 2181  |
| **postgres**                | Persistent store for deduplication keys and trade records                      | 5432  |
| **eureka-server**           | Service discovery (Netflix Eureka)                                             | 8761  |
| **order-producer-service**  | Simulates order placement, produces trade execution events                     | 8081  |
| **trading-engine-service**  | Idempotent consumer: checks DB before processing trades                        | 8082  |
| **duplicate-detector-service** | Shared persistence layer + health checks for deduplication                  | 8083  |
| **audit-service**           | Logs all processing attempts and outcomes for compliance                       | 8084  |

## Tech Stack

- Spring Boot 3.x
- Spring Kafka
- Spring Data JPA + Hibernate
- PostgreSQL (deduplication + business data)
- Apache Kafka 3.x (3-broker cluster)
- Spring Cloud Netflix Eureka
- Micrometer + Actuator (custom idempotency metrics)
- Lombok
- Maven (multi-module)
- Docker & Docker Compose

## Docker Containers

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka-1:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: trading
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  order-producer-service:
    build: ./order-producer-service
    depends_on:
      - kafka-1
      - eureka-server
    ports:
      - "8081:8081"

  trading-engine-service:
    build: ./trading-engine-service
    depends_on:
      - kafka-1
      - postgres
      - eureka-server
    ports:
      - "8082:8082"

  duplicate-detector-service:
    build: ./duplicate-detector-service
    depends_on:
      - postgres
      - eureka-server
    ports:
      - "8083:8083"

  audit-service:
    build: ./audit-service
    depends_on:
      - postgres
      - eureka-server
    ports:
      - "8084:8084"
```

Run with: `docker-compose up --build`

## Idempotency Strategy

| Component                     | Implementation Details                                                  |
|-------------------------------|-------------------------------------------------------------------------|
| **Deduplication Table**       | `processed_messages` (message_key PK, processed_at, source_topic)      |
| **Business Key**              | Trade/order ID as natural idempotency key                               |
| **Processing Flow**           | Consume → Check DB (SELECT) → If exists: skip → Else: process + INSERT  |
| **Transactional Safety**      | `@Transactional` on consumer listener — rollback on exception           |
| **Offset Commit**             | Manual ack only after successful DB insert                              |
| **Reprocessing Window**       | Optional TTL cleanup for old keys (e.g., retain 30 days)                |

## Key Features

- Application-level exactly-once processing
- Persistent deduplication surviving consumer restarts
- Transactional boundary: message processing + deduplication
- Custom metrics: duplicate attempts, processing latency
- Audit trail of all consumption attempts
- Graceful handling of poison messages
- Configurable deduplication strategy

## Expected Endpoints

### Order Producer Service (`http://localhost:8081`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| POST   | `/api/orders`                   | Place order → produce trade execution event      |
| POST   | `/api/orders/duplicates`        | Simulate duplicate messages                      |

### Trading Engine Service (`http://localhost:8082`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/trades/executed`          | List successfully executed trades                |
| GET    | `/actuator/metrics`             | View idempotency metrics (duplicates skipped)    |

### Duplicate Detector Service (`http://localhost:8083`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/dedup/stats`              | Deduplication statistics                         |
| GET    | `/api/dedup/keys`               | List processed message keys                      |

### Audit Service (`http://localhost:8084`)

| Method | Endpoint                        | Description                                      |
|--------|---------------------------------|--------------------------------------------------|
| GET    | `/api/audit/attempts`           | All processing attempts (success + duplicates)   |

## Architecture Overview

```
Order Placement
       ↓
Order Producer → Kafka Topic: trade.execution (key: orderId)
       ↓
Trading Engine Consumer
       ↓
Check PostgreSQL (processed_messages table)
   ├─→ Exists? → Skip + Ack
   └─→ New → Process Trade → INSERT dedup row → Commit offset
       ↓
Audit Service (logs outcome)
```

**Exactly-Once Flow**:
1. Message arrives (possibly duplicate)
2. Transaction begins
3. SELECT from dedup table
4. If not found → execute trade + INSERT dedup row
5. Commit transaction → manual ack
6. On exception → rollback → reconsume

## How to Run

1. Clone repository
2. Start Docker
3. `docker-compose up --build`
4. Place order: POST `/api/orders`
5. Send same order again → second processing skipped (check logs)
6. Restart trading-engine → no duplicate execution on reconsume

## Testing Idempotency

1. Send trade → executed once
2. Simulate duplicate (same orderId) → skipped, logged as duplicate
3. Kill consumer mid-processing → on restart, message reconsumed but safely skipped
4. Check DB → only one trade record

## Skills Demonstrated

- Achieving exactly-once semantics with at-least-once delivery
- Persistent idempotency patterns using relational DB
- Transactional consumer listeners in Spring Kafka
- Manual offset management for safety
- Deduplication strategies in financial systems
- Observability of duplicate detection
- Fault-tolerant consumer design

## Future Extensions

- Outbox pattern for transactional producer
- Distributed locks for high-contention keys
- Kafka Transactions (native EOS)
- Debezium for CDC-based deduplication
- Integration with saga/orchestration for compensation

