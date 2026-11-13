# dlocal-merchant-nestjs

![Node.js](https://img.shields.io/badge/Node.js-22%20LTS-339933?logo=nodedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-v7%20(tsgo)-3178C6?logo=typescript&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-11-E0234E?logo=nestjs&logoColor=white)
![Fastify](https://img.shields.io/badge/Fastify-HTTP%20Adapter-000000?logo=fastify&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8.0%20%2B%20Drizzle-4479A1?logo=mysql&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![pnpm](https://img.shields.io/badge/Monorepo-pnpm%20Workspaces-F69220?logo=pnpm&logoColor=white)

A **NestJS 11 + TypeScript v7 (`tsgo`) + React 19 Monorepo** implementing the **2018/05/17 Payment Gateway Technical Challenge**.

This repository ports the original Java 8, Spring 5, Embedded Jetty, Maven, and MySQL specification to **Node.js 22 LTS, TypeScript v7 (`tsgo`), NestJS 11 (Fastify + Microservices), and MySQL 8.0 (Drizzle ORM)**. It also defines five engineering extensions for synthetic seeding, load simulation, circuit breaking, cryptographic idempotency, and live operations.

---

## 1. Project premise and baseline specification

The baseline specification in [docs/requirements.md](./docs/requirements.md) defines two applications and a scheduled clearing job:

1. **Merchant Service (`apps/merchant`)**:
   - **`GET /check/:merchant_id`**: Verifies merchant existence (`200 OK` if found, `404 Not Found` otherwise) on HTTP port `8081` and `@nestjs/microservices` TCP port `8881`.
2. **Payment Manager (`apps/pm`)**:
   - **`POST /sale`**: Validates `merchant_id` against `apps/merchant`, converts `amount` to `USD` via `fixer.io` (maximum **1 external call per 30 seconds**), and persists the transaction in `PENDING` status with strict idempotency on `(merchant_id, transaction_id)`.
   - **`GET /status/:id`**: Returns transaction details (`id`, `date`, `transaction_id`, `merchant_id`, `amount_usd`, and `status`: `PENDING | PAID | REJECTED`).
   - **`GET /balance?merchant_id={id}&from={date}&to={date}`**: Returns accumulated USD totals (`total_paid`, `total_pending`, `total_rejected`) for a merchant and date range.
   - **30-Second Clearing Batch Job**: Runs every 30 seconds to transition `PENDING` transactions to `PAID` (70% probability) or `REJECTED` (30% probability).

---

## 2. Stack porting summary

| Concern | Original 2018 Premise | Target Monorepo Stack (`dlocal-merchant-nestjs`) |
| :--- | :--- | :--- |
| **Workspace & Build** | Maven Multi-Module | `pnpm` Workspaces Monorepo |
| **Runtime & Type Checker** | Java 8 (`javac`) | Node.js 22 LTS + TypeScript v7 (`@typescript/native-preview` / `tsgo`) + SWC |
| **Framework & Transport** | Spring Framework 5 | NestJS 11 (`@nestjs/platform-fastify` + `@nestjs/microservices` TCP/HTTP) |
| **Database & Persistence** | MySQL 5.x + JPA/Hibernate | MySQL 8.0 (InnoDB) + Drizzle ORM + Redis 7 (`UUIDv7`, `decimal.js`) |
| **Operations UI** | None | React 19 + Vite + Tailwind CSS (`apps/admin-panel`) |
| **Client Integration** | Raw REST calls | Isomorphic TypeScript SDK (`packages/client-sdk`) |
| **Testing & Simulation** | JUnit + MockMvc | Vitest + Supertest + 5-Persona Load Simulator (`packages/load-simulator`) |

### Workspace structure

```text
dlocal-merchant-nestjs/
├── apps/
│   ├── merchant/                 # Merchant Verification Hybrid Service (:8081 HTTP / :8881 TCP)
│   ├── pm/                       # Payment Manager Gateway & 30s Batch Engine (:8080 HTTP)
│   └── admin-panel/              # React 19 Operations Control Plane (:3000)
├── packages/
│   ├── shared-contracts/         # Shared DTOs, RPC message patterns & Decimal utilities
│   ├── seeder/                   # Deterministic Synthetic Data Generator (Pareto + FX)
│   ├── load-simulator/           # Multi-Persona Synthetic Usage & Fault Simulator
│   ├── resiliency/               # Circuit Breakers, Bulkheads, Timeouts & Cache Fallbacks
│   └── client-sdk/               # TypeScript Client SDK with RFC 8785 & HMAC-SHA256 Auth
└── docs/
    ├── requirements.md           # Baseline 2018/05/17 technical specification
    └── architecture/             # Complete architecture & stack porting suite
        ├── 00-OVERVIEW.md
        ├── 01-STACK-PORTING-AND-ADRS.md
        ├── 02-SYSTEM-INTEGRATION-AND-FLOWS.md
        ├── 03-DATA-MODEL-AND-CONTRACTS.md
        └── 04-IMPLEMENTATION-AND-TEST-PLAN.md
```

---

## 3. Phase 2+ engineering extensions

Beyond the Phase 1 baseline requirements, the architecture specifies five extensions:

1. **Synthetic Data Seeder (`packages/seeder`)**:
   - Seeds deterministic (`--seed 20180517`) Pareto (80/20) merchant tiers and multi-currency sales (`BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `EUR`, `USD`) using `5,000`-row streaming inserts.
2. **Synthetic Usage Simulator (`packages/load-simulator`)**:
   - Runs five concurrent actor personas (Checkout, Network Retry Burst, Status Tracker, Balance Auditor, and Payload Tampering) and verifies post-run invariants automatically.
3. **Resiliency and Circuit Breakers (`packages/resiliency`)**:
   - Enforces `800 ms` RPC timeouts, maximum 2 jittered retries (`100 ms` base backoff), `cockatiel` circuit breakers, L1/L2 read-through merchant caches, and `FOR UPDATE SKIP LOCKED` chunked batch clearing.
4. **Idempotency Engine, Client SDK and Auth (`packages/client-sdk`)**:
   - Implements `idempotency_records` with RFC 8785 SHA-256 payload fingerprinting (`HTTP 422` on mutated retry), Redis Pub/Sub in-flight lock resolution (`HTTP 409`), and HMAC-SHA256 request signing.
5. **React Admin Panel (`apps/admin-panel`)**:
   - Provides live circuit breaker controls, transaction and balance query explorers, 30-second batch job and `fixer.io` cache monitors, idempotency lease inspection, and load simulator controls.

---

## 4. Architecture documentation index

- **[Baseline Specification (`docs/requirements.md`)](./docs/requirements.md)**: Original 2018/05/17 technical challenge problem statement.
- **[00: Architecture Overview (`docs/architecture/00-OVERVIEW.md`)](./docs/architecture/00-OVERVIEW.md)**: Executive summary, requirements traceability matrix, system topology diagram, and ADR index.
- **[01: Stack Porting & ADRs (`docs/architecture/01-STACK-PORTING-AND-ADRS.md`)](./docs/architecture/01-STACK-PORTING-AND-ADRS.md)**: Part-by-part stack porting matrix and `ADR-01` through `ADR-06`.
- **[02: System Integration & Flows (`docs/architecture/02-SYSTEM-INTEGRATION-AND-FLOWS.md`)](./docs/architecture/02-SYSTEM-INTEGRATION-AND-FLOWS.md)**: End-to-end sequence diagrams for `SALE`, concurrent idempotency, 30s `fixer.io` caching, `STATUS` & `BALANCE`, and the 30s batch job.
- **[03: Data Model & Contracts (`docs/architecture/03-DATA-MODEL-AND-CONTRACTS.md`)](./docs/architecture/03-DATA-MODEL-AND-CONTRACTS.md)**: MySQL 8.0 DDL, Drizzle ORM schemas, covering indexes, REST contracts, and `@nestjs/microservices` message patterns.
- **[04: Implementation & Test Plan (`docs/architecture/04-IMPLEMENTATION-AND-TEST-PLAN.md`)](./docs/architecture/04-IMPLEMENTATION-AND-TEST-PLAN.md)**: Monorepo file tree, `tsgo` config, `start-apps.ps1` + `docker-compose.yml` orchestration, test matrix, and Phase 2+ roadmap.
