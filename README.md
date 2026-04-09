# EAI Capstone Project — Enterprise Integration with Node-RED

> Production-ready **Saga orchestration** with **verified Docker deployment**, **compensation flows**, and **runtime health checks**.

## Overview

This project implements an **Enterprise Application Integration (EAI)** solution using **Node-RED** as the central orchestration engine.

The orchestrator coordinates four microservices:

* **Order Service**
* **Payment Service**
* **Inventory Service**
* **Notification Service**

The solution focuses on:

* centralized workflow orchestration
* Saga-based distributed consistency
* compensation (refund / release) scenarios
* retry and fault tolerance
* correlation-based traceability
* containerized deployment with reproducible startup

---

## Deployment Requirements (Important)

The project is packaged so that the **actual submitted `flows.json` is loaded into `/data/flows.json` inside the Node-RED container at startup**.

This guarantees that the required endpoints exist at runtime:

* `GET /health`
* `POST /order`
* `POST /admin/payment/fail-mode`
* `POST /admin/inventory/fail-mode`
* `POST /admin/reset`

Without this mapping, Node-RED starts with the default starter flow, which breaks rubric verification.

### Docker volume mapping

```yaml
services:
  nodered:
    image: nodered/node-red
    ports:
      - "1880:1880"
    volumes:
      - ./flows.json:/data/flows.json
```

---

## Architecture Decision

### Why Node-RED

Node-RED is used as the **Process Manager** and orchestration layer because it:

* visually represents Saga flow logic
* simplifies HTTP-based service coordination
* supports retries and branching logic
* provides fast debugging during demos
* clearly maps to Enterprise Integration Patterns

---

## Runtime Endpoints

### Public endpoints

| Endpoint  | Method | Purpose                     |
| --------- | ------ | --------------------------- |
| `/health` | GET    | runtime health verification |
| `/order`  | POST   | create order Saga           |

### Admin endpoints (required for testing)

| Endpoint                     | Method | Purpose                     |
| ---------------------------- | ------ | --------------------------- |
| `/admin/payment/fail-mode`   | POST   | simulate payment failures   |
| `/admin/inventory/fail-mode` | POST   | simulate inventory failures |
| `/admin/reset`               | POST   | reset all failure flags     |

---

## Business Flow

### Happy Path

1. Client sends `POST /order`
2. Node-RED creates `correlationId`
3. Order is created
4. Payment is authorized
5. Inventory is reserved
6. Notification is sent (best-effort)
7. Response returns `completed`

---

## Compensation Scenarios

### 1) Payment failure

* retry payment **3 times**
* if still failed → stop flow
* return `error`

### 2) Inventory failure (required compensation)

* payment succeeds
* inventory reservation fails
* retry inventory **3 times**
* trigger **payment refund compensation**
* return `compensated`

### 3) Notification failure

* best-effort retry
* business flow remains successful
* return `completed_with_warning`

---

## Enterprise Integration Patterns Used

| Pattern                  | Node-RED implementation   |
| ------------------------ | ------------------------- |
| Process Manager          | main orchestration flow   |
| Request-Reply            | HTTP request nodes        |
| Retry Pattern            | retry subflows            |
| Correlation Identifier   | `correlationId`           |
| Content-Based Router     | switch nodes              |
| Dead Letter Channel      | centralized error handler |
| Compensating Transaction | refund branch             |

> Only patterns that are **actually implemented in the flow** are listed here.

---

## Example API Usage

### Health check

```bash
curl http://localhost:1880/health
```

### Happy path

```bash
curl -X POST http://localhost:1880/order \
  -H "Content-Type: application/json" \
  -d '{"item":"book","amount":50}'
```

### Force inventory compensation

```bash
curl -X POST http://localhost:1880/admin/inventory/fail-mode \
  -H "Content-Type: application/json" \
  -d '{"mode":"always"}'

curl -X POST http://localhost:1880/order \
  -H "Content-Type: application/json" \
  -d '{"item":"book","amount":50}'
```

---

## Expected Response Example

```json
{
  "orderId": "123",
  "correlationId": "1742834567890-abc123",
  "status": "compensated",
  "trace": [
    { "step": "order", "status": "success" },
    { "step": "payment", "status": "success" },
    { "step": "inventory", "status": "failed" },
    { "step": "refund", "status": "success" }
  ]
}
```

---

## How to Run

```bash
docker compose up -d --build
```

Verify:

```bash
curl http://localhost:1880/health
```

Stop:

```bash
docker compose down
```

---

## What Was Improved After Feedback

Based on professor feedback, the following critical issues were fixed:

* actual `flows.json` now loads into container runtime
* default starter flow issue removed
* `/health` endpoint added
* admin failure simulation endpoints added
* README claims aligned only with implemented behavior
* removed leftover AI-generated chat fragments
* added explicit end-to-end compensation test cases

---

System Architecture
<img width="974" height="1382" alt="image" src="https://github.com/user-attachments/assets/7eca7c1b-2069-4060-81bd-96c69afee91b" />
<img width="972" height="296" alt="image" src="https://github.com/user-attachments/assets/896d4d50-4e03-4ce7-8121-7c8a188283b7" />
