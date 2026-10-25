# Minimal Feature Flag Manager

[![Python Version](https://img.shields.io/badge/python-3.10%20%7C%203.11%20%7C%203.12-blue?style=flat-square&logo=python)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100%2B-009688?style=flat-square&logo=fastapi)](https://fastapi.tiangolo.com/)
[![CI Status](https://img.shields.io/badge/build-passing-brightgreen?style=flat-square&logo=github-actions)](https://github.com/)
[![Code Style: Black](https://img.shields.io/badge/code%20style-black-000000.svg?style=flat-square)](https://github.com/psf/black)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](https://opensource.org/licenses/MIT)

A lightweight, self-hosted web application and high-throughput evaluation engine for runtime feature toggles, canary releases, and dynamic user targeting. Minimal Feature Flag Manager eliminates external SaaS dependencies by offering a low-latency REST API for microservices paired with a server-rendered administrative UI.

---

## 1. Problem Statement & Target Audience

### The Problem
Decoupling continuous deployment from feature activation is critical to modern engineering workflows. However, existing commercial feature management platforms (e.g., LaunchDarkly, Split) introduce significant operational overhead:
* **Prohibitive Cost**: Per-seat and per-request pricing models scale poorly for internal utilities and mid-tier systems.
* **Third-Party Data Egress**: PII and context attributes must often leave your infrastructure for evaluation.
* **Network Latency & Reliability**: Hard dependencies on third-party cloud APIs introduce runtime failure modes during flag evaluation.

### Target Audience
* **Platform & Infrastructure Teams**: Looking to self-host toggle infrastructure within private VPCs.
* **Backend Engineers**: Requiring low-latency flag evaluation via local SQLite persistence without external SaaS contracts.
* **Small-to-Medium Engineering Teams**: Needing straightforward percentage rollouts, user targeting rules, and audit trails without unnecessary enterprise complexity.

---

## 2. Key Features

* **Environment-Aware Administrative Dashboard**: Inspect, configure, and flip kill switches across multiple environments (`production`, `staging`, `development`) through a server-rendered Jinja2 interface.
* **Deterministic Rule & Rollout Engine**: Compute consistent user buckets using Murmur3/SHA-256 hash modular arithmetic without persisting user tracking state. Supports custom attribute matching (e.g., `role == 'beta-tester'`, `country == 'US'`).
* **High-Performance REST API**: Low-latency endpoints supporting both single-flag lookups and batch evaluations for entire client application contexts in a single round-trip.
* **Immutable Audit Trail**: Automatically captures every toggle flip, rollout percentage adjustment, and environment configuration override with actor attribution and historical timestamps.

---

## 3. Architecture & System Design

The architecture optimizes for evaluation speed and simplicity. Flag definitions and rules are persisted via SQLite and queried using SQLAlchemy ORM. The evaluation engine computes flag states dynamically against incoming JSON context payloads.

### System Flow Diagram

```
+-------------------------------------------------------------------------+
|                              Admin Browser                              |
+-------------------------------------------------------------------------+
                                     |
                         HTTP GET/POST (Admin UI)
                                     v
+-------------------------------------------------------------------------+
|                              FastAPI Core                               |
|                                                                         |
|  +--------------------+   +---------------------+   +----------------+  |
|  | /admin/ (Jinja2)   |   | /api/v1/evaluate    |   | /api/v1/flags  |  |
|  | Dashboard Router   |   | Flag Eval Engine    |   | CRUD Endpoints |  |
|  +--------------------+   +---------------------+   +----------------+  |
|             |                        |                      |           |
|             v                        v                      v           |
|  +-------------------------------------------------------------------+  |
|  |                   Core Evaluation & Audit Engine                  |  |
|  |  - Deterministic SHA256 User Rollout Bucket Calculation           |  |
|  |  - Targeting Rule Predicate Matching                              |  |
|  |  - Append-Only Audit Trail Dispatcher                            |  |
|  +-------------------------------------------------------------------+  |
|                                      |                                  |
|                                      v                                  |
|  +-------------------------------------------------------------------+  |
|  |                    SQLAlchemy 2.0 ORM Engine                      |  |
|  |       [Environments] <---> [Flags] <---> [Rules]                  |  |
|  |                                  \                                |  |
|  |                                   +--> [AuditLogs]                |  |
|  +-------------------------------------------------------------------+  |
+-------------------------------------------------------------------------+
                                     |
                                  SQLite
                          (Single-file or In-Memory)
```

### Module Responsibilities

| Module | Location | Purpose |
| :--- | :--- | :--- |
| **Database** | `src/database.py` | Engine initialization, thread-safe connection pooling for SQLite (`check_same_thread=False`), and FastAPI `get_db` dependency provider. |
| **Models** | `src/models.py` | SQLAlchemy declarative database models (`Flag`, `Rule`, `Environment`, `AuditLog`) and Pydantic schemas for data validation and serialization. |
| **Evaluation Engine** | `src/core.py` | Deterministic percentage hashing, rule condition parsing, and audit event recording. |
| **API Endpoints** | `src/api.py` | REST routes for CRUD operations, single/batch evaluation endpoints, and Jinja2 admin view rendering. |
| **Main Application** | `src/main.py` | Application bootstrapping, static asset mounting, schema migration triggers, and Uvicorn server execution. |

---

## 4. Technology Stack

* **Language**: [Python 3.10+](https://www.python.org/)
* **Web Framework**: [FastAPI](https://fastapi.tiangolo.com/) (Asynchronous, OpenAPI-native)
* **ASGI Server**: [Uvicorn](https://www.uvicorn.org/)
* **Data Layer**: [SQLAlchemy 2.0](https://www.sqlalchemy.org/) + [SQLite](https://www.sqlite.org/)
* **Validation & Schemas**: [Pydantic v2](https://docs.pydantic.dev/)
* **Templating & UI**: [Jinja2](https://jinja.palletsprojects.com/) + Vanilla JavaScript / Modern CSS
* **Test Suite**: [pytest](https://docs.pytest.org/), `pytest-asyncio`, `httpx`

---

## 5. Installation & Setup

### Prerequisites
* Python 3.10 or higher
* `git`

### Step 1: Clone the Repository
```bash
git clone https://github.com/your-org/minimal-feature-flag-manager.git
cd minimal-feature-flag-manager
```

### Step 2: Create and Activate a Virtual Environment
```bash
# Linux/macOS
python3 -m venv venv
source venv/bin/activate

# Windows (PowerShell)
python -m venv venv
.\venv\Scripts\Activate.ps1
```

### Step 3: Install Dependencies
```bash
pip install --upgrade pip
pip install -r requirements.txt
```

---

## 6. Environment Variables Configuration

Copy the example environment configuration file to create your active configuration:

```bash
cp .env.example .env
```

### Available Configuration Directives

```ini
# Application Runtime Configuration
APP_NAME="Minimal Feature Flag Manager"
APP_ENV=development
DEBUG=True
HOST=127.0.0.1
PORT=8000

# Storage
DATABASE_URL=sqlite:///./feature_flags.db

# Security & CORS
SECRET_KEY=change-this-to-a-secure-random-secret-key-in-production
ALLOWED_ORIGINS=["*"]

# Evaluation Defaults
DEFAULT_HASH_SALT="4f9c1b7e8d2a6e3c"
```

---

## 7. Usage & API Reference

### Running the Application

Launch the server using the entrypoint module:

```bash
python -m src.main
```
Or directly with Uvicorn:
```bash
uvicorn src.main:app --host 127.0.0.1 --port 8000 --reload
```

* **Administrative UI**: Access `http://127.0.0.1:8000/admin` in your browser.
* **Interactive API Documentation (Swagger UI)**: Access `http://127.0.0.1:8000/docs`.

---

### Core API Endpoints

#### 1. Create a Feature Flag

```bash
curl -X POST "http://127.0.0.1:8000/api/v1/flags" \
     -H "Content-Type: application/json" \
     -d '{
       "key": "new_checkout_flow",
       "description": "Enables the modern one-page checkout experience.",
       "environment": "production",
       "enabled": true,
       "rollout_percentage": 25,
       "author": "developer@company.internal"
     }'
```

#### 2. Evaluate a Single Flag for a Context

```bash
curl -X POST "http://127.0.0.1:8000/api/v1/evaluate" \
     -H "Content-Type: application/json" \
     -d '{
       "flag_key": "new_checkout_flow",
       "environment": "production",
       "entity_id": "usr_948202",
       "attributes": {
         "tier": "enterprise",
         "country": "US"
       }
     }'
```

**Response (200 OK):**
```json
{
  "flag_key": "new_checkout_flow",
  "enabled": true,
  "reason": "MATCH_PERCENTAGE_ROLLOUT",
  "evaluated_at": "2023-10-24T14:32:01.129384Z"
}
```

#### 3. Batch Flag Evaluation

Client applications can request evaluations for multiple flags concurrently in a single HTTP request:

```bash
curl -X POST "http://127.0.0.1:8000/api/v1/evaluate/batch" \
     -H "Content-Type: application/json" \
     -d '{
       "environment": "production",
       "entity_id": "usr_948202",
       "flag_keys": ["new_checkout_flow", "dark_mode_v2", "realtime_notifications"],
       "attributes": {
         "beta_tester": true,
         "app_version": "2.4.0"
       }
     }'
```

**Response (200 OK):**
```json
{
  "entity_id": "usr_948202",
  "environment": "production",
  "flags": {
    "new_checkout_flow": { "enabled": true, "reason": "MATCH_PERCENTAGE_ROLLOUT" },
    "dark_mode_v2": { "enabled": true, "reason": "TARGETING_RULE_MATCH" },
    "realtime_notifications": { "enabled": false, "reason": "GLOBAL_KILL_SWITCH" }
  }
}
```

#### 4. Fetch Flag Audit Log History

```bash
curl -X GET "http://127.0.0.1:8000/api/v1/flags/new_checkout_flow/audit"
```

---

## 8. Automated Testing

The suite uses `pytest` and an isolated in-memory SQLite database to ensure atomic, non-destructive test execution.

### Run All Unit and Integration Tests
```bash
pytest
```

### Run with Verbose Output and Coverage Reporting
```bash
pytest -v --cov=src --cov-report=term-missing
```

### Targeted Test Execution
```bash
# Run core evaluation logic tests only
pytest tests/test_core.py

# Run API endpoint integration tests only
pytest tests/test_api.py
```

---

## 9. Project Structure

```
.
├── .env.example              # Sample environment variable template
├── .gitignore                # Git ignore configuration
├── requirements.txt          # Python runtime dependencies
├── src/
│   ├── __init__.py           # Package marker
│   ├── api.py                # FastAPI routes, schemas, and template endpoints
│   ├── core.py               # Evaluation engine, hashing, and audit triggers
│   ├── database.py           # SQLAlchemy session and engine management
│   ├── main.py               # App entrypoint, ASGI initialization, and hooks
│   └── models.py             # ORM database entities and Pydantic validation schemas
└── tests/
    ├── __init__.py           # Test package marker
    ├── test_api.py           # Endpoint integration and contract tests
    └── test_core.py          # Unit tests for deterministic hashing and targeting
```

---

## 10. Roadmap

- [ ] **Redis Evaluation Cache**: In-memory write-through cache layer for sub-millisecond evaluation workloads.
- [ ] **Streaming Updates (SSE / WebSockets)**: Push instant flag updates to long-running client applications without polling.
- [ ] **Role-Based Access Control (RBAC)**: Fine-grained permissions (Viewer, Editor, Admin) enforced per environment.
- [ ] **Client SDKs**: Lightweight client packages for Node.js, Go, and Python with local caching and polling sync.
- [ ] **Flag Expiration Alerts**: Automatic notifications for stale flags that have reached 100% rollout or remained untouched.

---

## 11. License

This project is licensed under the terms of the **MIT License**. See the [LICENSE](LICENSE) file for complete details.