# Implementation, startup orchestration, and verification plan

This document specifies the `pnpm` monorepo structure, TypeScript v7 (`tsgo`) configuration, single-command startup orchestration (`docker-compose.yml` + `start-apps.ps1`), automated verification strategy, and Phase 2+ follow-up extensions roadmap.

---

## 1. Monorepo and directory structure

```text
dlocal-merchant-nestjs/
├── package.json                          # Root scripts (build, typecheck via tsgo, test, start:all)
├── pnpm-workspace.yaml                   # Workspaces: apps/*, packages/*
├── tsconfig.base.json                    # Shared TypeScript v7 (tsgo) strict compiler options
├── docker-compose.yml                    # MySQL 8.0, Redis 7, apps/merchant, apps/pm
├── start-apps.ps1                        # Single-command startup script (Deliverable requirement)
├── README.md                             # Entrypoint overview & links to docs/
├── apps/
│   ├── merchant/                         # Application 1: Merchant Service (:8081 HTTP / :8881 TCP)
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── main.ts                   # Fastify + TCP Hybrid microservice bootstrap
│   │   │   ├── merchant.module.ts
│   │   │   ├── merchant.controller.ts    # @Get('check/:merchant_id') & @MessagePattern({ cmd: 'merchant.check' })
│   │   │   ├── merchant.service.ts
│   │   │   └── db/
│   │   │       ├── drizzle.provider.ts
│   │   │       └── schema.ts             # merchants table definition
│   │   └── test/
│   │       ├── merchant.service.spec.ts
│   │       └── merchant.e2e-spec.ts
│   ├── pm/                               # Application 2: Payment Manager (:8080 HTTP)
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── main.ts                   # Fastify HTTP bootstrap + ValidationPipe
│   │   │   ├── pm.module.ts
│   │   │   ├── sale/
│   │   │   │   ├── sale.controller.ts    # POST /sale, GET /status/:id, GET /balance
│   │   │   │   ├── sale.service.ts
│   │   │   │   └── dto/
│   │   │   │       ├── create-sale.dto.ts
│   │   │   │       └── get-balance-query.dto.ts
│   │   │   ├── merchant-client/
│   │   │   │   └── merchant-check.client.ts # TCP ClientProxy + HTTP fallback + Cockatiel CB
│   │   │   ├── fixer/
│   │   │   │   └── fixer.service.ts      # 30s rate limiter, single-flight coalescing, Decimal math
│   │   │   ├── batch/
│   │   │   │   ├── batch-clearing.job.ts # @Interval(30_000) chunked FOR UPDATE SKIP LOCKED
│   │   │   │   └── random.provider.ts    # Injectable PRNG (crypto in prod, seeded in tests)
│   │   │   └── db/
│   │   │       ├── drizzle.provider.ts
│   │   │       └── schema.ts             # sales, idempotency_records, fx_rates_snapshot
│   │   └── test/
│   │       ├── fixer.service.spec.ts
│   │       ├── batch-clearing.job.spec.ts
│   │       ├── sale.concurrency.spec.ts
│   │       └── pm.e2e-spec.ts
│   └── admin-panel/                      # Phase 2+: React 19 Operations Control Plane (:3000)
├── packages/
│   ├── shared-contracts/                 # Shared RPC DTOs, error codes, Decimal helpers
│   ├── resiliency/                       # Phase 2+: Circuit breakers, bulkheads, L1/L2 caches
│   ├── client-sdk/                       # Phase 2+: Isomorphic SDK with RFC 8785 & HMAC-SHA256
│   ├── seeder/                           # Phase 2+: Deterministic Pareto & multi-currency seeder
│   └── load-simulator/                   # Phase 2+: 5-persona concurrent load & invariant runner
└── docs/
    ├── requirements.md                   # Baseline 2018/05/17 technical specification
    └── architecture/
        ├── 00-OVERVIEW.md
        ├── 01-STACK-PORTING-AND-ADRS.md
        ├── 02-SYSTEM-INTEGRATION-AND-FLOWS.md
        ├── 03-DATA-MODEL-AND-CONTRACTS.md
        └── 04-IMPLEMENTATION-AND-TEST-PLAN.md
```

### TypeScript v7 (`tsgo`) workspace configuration (`tsconfig.base.json`)

```json
{
  "compilerOptions": {
    "target": "ES2023",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noImplicitOverride": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "declaration": true,
    "sourceMap": true
  }
}
```
- `pnpm typecheck` executes `tsgo -b tsconfig.json --pretty` across all packages using `@typescript/native-preview`.
- `pnpm build` compiles runtime bundles via `@swc/core`.

---

## 2. Startup and orchestration plan (`start-apps.ps1` & `docker-compose.yml`)

`docs/requirements.md` requires a script to start both applications (`Un script para levantar las dos aplicaciones`).

### 2.1 Container topology (`docker-compose.yml`)
1. **`mysql` (`mysql:8.0`):**
   - Port: `3306:3306`.
   - Mounts `./docker/mysql/init.sql` to create `merchant_db` and `pm_db`, apply DDL for `merchants` and `sales`, and seed baseline test merchants (`merch-01`, `merch-02`, `merch-03`).
   - Healthcheck: `mysqladmin ping -h localhost -u root -p$MYSQL_ROOT_PASSWORD` (interval `5s`, retries `10`).
2. **`redis` (`redis:7-alpine`):**
   - Port: `6379:6379`.
   - Used for distributed L2 caching and `FixerService` / `IdempotencyEngine` locks.
3. **`merchant` (`apps/merchant`):**
   - Ports: `8081:8081` (HTTP Fastify) and `8881:8881` (TCP `@nestjs/microservices`).
   - Depends on `mysql` (`service_healthy`).
4. **`pm` (`apps/pm`):**
   - Port: `8080:8080` (HTTP Fastify).
   - Environment: `MERCHANT_TCP_HOST=merchant`, `MERCHANT_TCP_PORT=8881`, `MERCHANT_HTTP_URL=http://merchant:8081`, `FIXER_API_URL`, `FIXER_API_KEY`, `FIXER_TTL_MS=30000`, `BATCH_INTERVAL_MS=30000`.
   - Depends on `mysql` (`service_healthy`), `redis`, and `merchant`.

### 2.2 Single-command startup script (`start-apps.ps1`)
Run `.\start-apps.ps1` from PowerShell to start the stack:
1. Checks whether Docker is running and starts the `mysql` and `redis` containers via `docker compose up -d mysql redis`.
2. Polls `mysql` readiness until port `3306` accepts connections (timeout after `30,000 ms`).
3. Runs `pnpm db:migrate` and `pnpm db:seed:baseline` to ensure `merchant_db.merchants` and `pm_db.sales` exist.
4. Starts `apps/merchant` (`:8081` / `:8881`) and `apps/pm` (`:8080`) in parallel via `pnpm --parallel --filter "@dlocal/merchant" --filter "@dlocal/pm" start:prod` (or `docker compose up --build -d` when `-Docker` is passed).
5. Probes `GET http://localhost:8081/check/merch-01` and `GET http://localhost:8080/balance?merchant_id=merch-01&from=2026-01-01&to=2026-12-31` and prints `HTTP 200` confirmation.

---

## 3. Verification and testing strategy

`docs/requirements.md` states: *"Se deben implementar tests para ambas aplicaciones."* All unit, integration, concurrency, and E2E suites run via **Vitest** and **Supertest**.

### 3.1 Test matrix across applications

| Test Suite | Target Component | Scenarios & Asserted Invariants |
| :--- | :--- | :--- |
| **`merchant.service.spec.ts`** (Unit) | `apps/merchant` `MerchantService` | Returns `{ exists: true }` for active merchant IDs and `{ exists: false }` for unknown IDs. |
| **`merchant.e2e-spec.ts`** (E2E HTTP + TCP) | `apps/merchant` Hybrid App (`:8081` & `:8881`) | 1. `GET /check/merch-01` returns `HTTP 200 OK`.\n2. `GET /check/unknown-99` returns `HTTP 404 Not Found`.\n3. TCP `ClientProxy.send({ cmd: 'merchant.check' }, { merchant_id: 'merch-01' })` returns `{ exists: true }`. |
| **`fixer.service.spec.ts`** (Unit + Fake Timers) | `apps/pm` `FixerService` | 1. **30-second rate-limit invariant:** Fires 100 calls across a `29,999 ms` window (`vi.advanceTimersByTime(29_999)`) and asserts `httpFetchSpy` was called **1 time**.\n2. Advances timer to `30,001 ms`, fires another burst of 50 calls, and asserts `httpFetchSpy` total call count is **2**.\n3. **Single-flight coalescing:** Simulates a `150 ms` network delay on `fixer.io` and fires 25 concurrent `convertToUsd()` calls at $t = 0\text{ ms}$; asserts all 25 resolve from 1 HTTP call.\n4. **USD precision:** Verifies `ROUND_HALF_UP` scale-2 conversion across `USD`, `EUR`, `BRL`, `ARS`, and `JPY`. |
| **`batch-clearing.job.spec.ts`** (Unit + Seeded PRNG) | `apps/pm` `BatchClearingJob` | 1. Injects a deterministic sequence generator (`[0.10, 0.69, 0.70, 0.95]`) for 4 `PENDING` rows; asserts the first 2 transition to `PAID` (`r < 0.70`) and the last 2 transition to `REJECTED` (`r >= 0.70`).\n2. Seeds 1,000 `PENDING` rows with Mulberry32 seed `20180517` and verifies the resulting distribution is within $70\% \pm 1.5\%$ `PAID` and $30\% \pm 1.5\%$ `REJECTED`.\n3. Verifies already-settled (`PAID` or `REJECTED`) rows are never modified on subsequent 30-second runs. |
| **`sale.concurrency.spec.ts`** (Concurrency E2E) | `apps/pm` `POST /sale` | Fires **50 simultaneous `POST /sale` requests** via `Promise.all` with the exact same `(merchant_id: "merch-01", transaction_id: "tx-race-01")`. Asserts:\n1. All 50 responses return the **exact same `id`**.\n2. `SELECT COUNT(*) FROM sales WHERE merchant_id='merch-01' AND transaction_id='tx-race-01'` equals **`1`**.\n3. `fixer.io` mock server received at most **1 request**. |
| **`pm.e2e-spec.ts`** (Full E2E Lifecycle) | `apps/pm` + `apps/merchant` + MySQL | 1. `POST /sale` with unknown `merchant_id` returns `HTTP 404`.\n2. `POST /sale` with valid `merchant_id` returns `HTTP 200 { id }`.\n3. Immediate `GET /status/:id` returns `status: "PENDING"` and converted `amount_usd`.\n4. `GET /balance` before batch run reports `total_pending == amount_usd`, `total_paid == 0`, `total_rejected == 0`.\n5. Triggers `BatchClearingJob`, then queries `GET /status/:id` and `GET /balance` to verify transition to `PAID` or `REJECTED` and matching USD balance totals. |

### 3.2 Exact CLI verification commands

```powershell
# 1. Run TypeScript v7 (tsgo) type checking across all workspaces
pnpm typecheck

# 2. Run unit test suites for Merchant and PM
pnpm test

# 3. Run E2E and concurrency suites against MySQL & Redis
pnpm test:e2e
```

---

## 4. Phase 2+ follow-up extensions roadmap

Once the Phase 1 baseline satisfies every requirement in `docs/requirements.md`, the architecture expands across five modular packages without breaking the baseline REST contracts:

| Phase & Package | Architectural Scope & Quantified Bounds |
| :--- | :--- |
| **Extension 1: Synthetic Data Seeder (`packages/seeder`)** | Deterministic `@faker-js/faker` (`--seed 20180517`) bulk generator implementing Pareto merchant volume tiers (`5%` Enterprise generating `65%` of volume, `20%` Mid-Market generating `25%`, `75%` SMB generating `10%`), emerging-market FX corridors (`BRL`, `MXN`, `ARS`, `CLP`, `COP`, `UYU`, `EUR`, `USD`), and streaming `5,000`-row multi-row `INSERT` chunks with `<150 MB` V8 heap usage. |
| **Extension 2: Multi-Persona Load Simulator (`packages/load-simulator`)** | Worker-thread traffic generator running 5 concurrent personas: Checkout Actor (`65%`), Network Retry Burst Actor (`15%`, firing `3` to `10` duplicate `(merchant_id, transaction_id)` calls within `20 ms`), Status Poller (`12%`, polling every `5,000 ms`), Balance Auditor (`5%`), and Fault/Tampering Actor (`3%`). Automatically verifies post-run invariants (zero duplicate rows, `fixer.io` call count $\le \lceil T/30 \rceil$, and $70\% \pm 1.5\%$ batch settlement ratio). |
| **Extension 3: Resiliency & Circuit Breakers (`packages/resiliency`)** | `cockatiel` policy wrappers around `MerchantCheckClient` and `FixerService`: `800 ms` RPC timeout, max 2 jittered retries (`100 ms` base backoff), circuit breaker (`5` consecutive failures -> `15,000 ms` open state), separate `undici` connection pool bulkheads, L1 (`60 s`) / L2 (`15 m`) read-through merchant cache fallback, and `fx_rates_snapshot` stale-while-revalidate fallback. |
| **Extension 4: Idempotency Engine, Client SDK & Auth (`packages/client-sdk`)** | Server-side `idempotency_records` state machine with RFC 8785 canonical SHA-256 payload fingerprinting (`HTTP 422` on mutated retry), Redis `SET NX` + Pub/Sub in-flight lock resolution (`1,500 ms` wait window before `HTTP 409` + `Retry-After: 1`), HMAC-SHA256 request signing (`X-Merchant-Key`, `X-Timestamp` `300 s` anti-replay window, `X-Signature`), and zero-dependency TypeScript client SDK. |
| **Extension 5: Operations Admin Panel (`apps/admin-panel`)** | React 19 operator control plane built in strict compliance with `AGENTS.md` Section 4 (`ENERGY 1 / RHYTHM 2 / MOTION 1`, WCAG 2.2 AA `4.5:1` text and `3:1` UI contrast, explicit Empty/Loading/Error states, `44x44px` tap targets, and live SSE telemetry). Provides interactive circuit breaker overrides, live 30-second batch job and `fixer.io` TTL controls, idempotency lease inspection, and load simulator execution. |
