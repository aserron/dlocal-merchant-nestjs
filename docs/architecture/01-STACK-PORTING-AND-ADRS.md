# Stack porting matrix and architecture decision records

This document maps the 2018/05/17 specification in [docs/requirements.md](../requirements.md) to the target Node.js 22 LTS, TypeScript v7 (`tsgo`), and NestJS 11 stack, and records the architectural decisions (`ADR-01` through `ADR-06`) governing compiler setup, HTTP/microservice transport, persistence, rate-limited foreign exchange caching, and batch clearing.

---

## 1. Part-by-part stack porting matrix

| Specification Element (`docs/requirements.md`) | Original 2018 Premise | Target NestJS Stack (`dlocal-merchant-nestjs`) | Porting Rationale & Quantified Bounds |
| :--- | :--- | :--- | :--- |
| **Language & Compiler** | Java 8 (`javac`) | **Node.js 22 LTS + TypeScript v7 (`@typescript/native-preview` / `tsgo`) + SWC** | `tsgo` executes monorepo type-checking in native Go code (8x to 10x faster than JS `tsc`), while `@swc/core` compiles runtime JavaScript and decorator metadata in `<50 ms` per package. |
| **Application Framework** | Spring Framework 5 (IoC, MVC, DI) | **NestJS 11 (`@nestjs/common`, `@nestjs/core`, `@nestjs/microservices`)** | Provides modular dependency injection, controller routing, validation pipes, exception filters, and native microservice transport abstractions matching Spring's programming model. |
| **HTTP Server Engine** | Embedded Jetty | **Fastify (`@nestjs/platform-fastify`)** | Replaces the default Express adapter just as Embedded Jetty replaced default Tomcat in Spring, reducing HTTP dispatch overhead and accelerating JSON schema serialization (`fast-json-stringify`). |
| **Build & Multi-Project Packaging** | Maven Multi-Module + ZIP archive | **`pnpm` Workspaces Monorepo (`apps/merchant`, `apps/pm`, `packages/*`)** | Manages both applications, shared contracts, resiliency policies, seeder, load simulator, and tests under a single content-addressable lockfile (`pnpm-lock.yaml`). |
| **Application 1 (`Merchant` / `CHECK`)** | Spring MVC REST (`GET /check/{merchant_id}`) | **NestJS Hybrid App (`apps/merchant`)**: HTTP REST (`:8081`) + `@nestjs/microservices` TCP listener (`:8881`) | Exposes `GET /check/:merchant_id` (`200 OK` or `404 Not Found`) for external REST callers and `@MessagePattern({ cmd: 'merchant.check' })` over persistent TCP for internal `PM` checks. |
| **Application 2 (`PM` / `SALE`, `STATUS`, `BALANCE`)** | Spring MVC REST + `RestTemplate` | **NestJS Fastify App (`apps/pm`, `:8080`) + `ClientProxy` + `undici`** | Handles `POST /sale`, `GET /status/:id`, and `GET /balance` with strict `(merchant_id, transaction_id)` idempotency and `decimal.js` (`ROUND_HALF_UP`, scale 2) USD persistence. |
| **Database & Persistence** | MySQL 5.x + JDBC / JPA (Hibernate) | **MySQL 8.0 (InnoDB) + Drizzle ORM + `mysql2` pool** | Preserves MySQL relational semantics and ACID transactions while adding `FOR UPDATE SKIP LOCKED` support (introduced in MySQL 8.0) and zero-decorator TypeScript type inference compatible with `tsgo`. |
| **`fixer.io` 30-Second Rate Limit** | In-memory timer / `Guava Cache` | **Single-flight `inFlightPromise` + L1 in-memory TTL (`30,000 ms`) + L2 Redis lock/cache + MySQL snapshot** | Guarantees at most 1 outbound HTTP call to `fixer.io` every 30 seconds across concurrent bursts and multi-instance `PM` deployments. |
| **30-Second Batch Process (`70% PAID / 30% REJECTED`)** | Spring `@Scheduled(fixedRate = 30000)` | **`@nestjs/schedule` (`@Interval(30_000)`) + chunked `FOR UPDATE SKIP LOCKED`** | Runs every 30 seconds inside `apps/pm`, selecting `PENDING` transactions in batches of `1,000` rows without blocking concurrent `POST /sale` writes or double-processing across replicas. |
| **Automated Testing** | JUnit 4/5 + Spring `MockMvc` | **Vitest + `@nestjs/testing` + Supertest + Testcontainers** | Runs unit, concurrency, and E2E suites against both `apps/merchant` and `apps/pm` with deterministic PRNG injection for the 70/30 batch distribution. |
| **Startup Script Deliverable** | Shell script + Maven build | **`start-apps.ps1` (PowerShell) + `docker-compose.yml`** | Starts MySQL 8.0, Redis 7, `apps/merchant`, and `apps/pm` with health checks and schema migrations in a single command. |

---

## 2. Architecture decision records (ADRs)

### ADR-01: TypeScript v7 (`tsgo`), NestJS DI configuration, decimal math, and ID generation

#### Context
TypeScript v7 (`@typescript/native-preview`, invoked via `tsgo`) ports the TypeScript compiler and type checker from JavaScript to native Go. Traditional NestJS applications rely on legacy TypeScript `experimentalDecorators` and `emitDecoratorMetadata` (powered by `reflect-metadata`) to infer constructor parameter types at runtime. In addition, financial calculations in JavaScript (`Number` IEEE 754 double-precision floats) produce rounding errors (`0.1 + 0.2 === 0.30000000000000004`), and standard random UUIDv4 primary keys cause random B+ Tree page splits in MySQL InnoDB clustered indexes.

#### Options evaluated

| Dimension | Option A: `tsgo` Type-Check + SWC Build + Explicit `@Inject()` Tokens (Selected) | Option B: Legacy `tsc` Only | Option C: Babel Transpilation |
| :--- | :--- | :--- | :--- |
| **Type-checking speed** | `tsgo --noEmit` runs in `<300 ms` across all monorepo packages | `2,500 ms` to `6,000 ms` as monorepo grows | Requires separate `tsc` step anyway |
| **NestJS DI reliability** | Explicit `@Inject(TOKEN)` symbols never fail when interfaces or type-only imports are erased | Fails silently (`undefined` injection) if an interface is used without `@Inject()` | Requires manual plugin configuration |
| **Runtime build speed** | `@swc/core` compiles each service in `<50 ms` | Full `tsc` emit takes seconds | Slower than SWC |

#### Decision
1. **Compiler & DI strategy:** Configure `@typescript/native-preview` (`tsgo --noEmit`) as the workspace type checker in `package.json` (`pnpm typecheck`), and `@swc/core` via `@nestjs/cli` (`"builder": "swc"`) for runtime compilation. All NestJS providers use explicit Symbol injection tokens (`@Inject(MERCHANT_CLIENT)`, `@Inject(DRIZZLE_DB)`, `@Inject(RANDOM_PROVIDER)`) so dependency resolution does not depend on fragile runtime `design:paramtypes` reflection.
2. **Decimal arithmetic strategy:** Ban native JavaScript `number` arithmetic for monetary amounts. Use `decimal.js` configured with `precision: 20` and `rounding: Decimal.ROUND_HALF_UP`. All currency conversions (`amount / rate[currency] * rate['USD']`) compute at 8-decimal intermediate scale and round to 2 decimal places (`DECIMAL(18, 2)`) before database persistence. DTOs accept strings or numbers and normalize to canonical two-decimal strings (`"108.50"`).
3. **Platform-unique ID generation:** Generate RFC 9562 **UUIDv7** identifiers (`uuidv7` package). UUIDv7 embeds a 48-bit Unix millisecond timestamp prefix followed by 74 bits of cryptographic randomness. In Phase 1 baseline mode, IDs format as standard 36-character strings (`VARCHAR(36)`) to satisfy string JSON contracts directly; in Phase 2+ high-volume mode, MySQL stores UUIDv7 as `BINARY(16)` (`UUID_TO_BIN(id, 1)`), eliminating InnoDB clustered index fragmentation and cutting primary index size by 55%.

---

### ADR-02: HTTP server adapter and hybrid application strategy

#### Context
The original specification specifies Embedded Jetty instead of default Tomcat to keep the HTTP footprint small. In NestJS, the architect must choose between `@nestjs/platform-express` and `@nestjs/platform-fastify`, and decide whether `apps/merchant` runs as a pure HTTP server or a NestJS Hybrid Application (`app.connectMicroservice()`).

#### Options evaluated

| Option | Throughput (JSON REST) | Schema Serialization | Microservice Co-Hosting |
| :--- | :--- | :--- | :--- |
| **Option 1: `@nestjs/platform-fastify` + Hybrid App (Selected)** | ~45,000 req/s per Node.js process | Native `fast-json-stringify` compilation | Exposes REST on `:8081` and `@nestjs/microservices` on `:8881` in one DI container |
| **Option 2: `@nestjs/platform-express` (Pure HTTP)** | ~18,000 req/s per Node.js process | `JSON.stringify` per response | HTTP only |
| **Option 3: Standalone Microservice Only (No HTTP on Merchant)** | High internal throughput | N/A | Violates external `GET /check/:merchant_id` REST requirement (`200 OK` / `404 Not Found`) |

#### Decision
Use **`@nestjs/platform-fastify`** for both `apps/merchant` and `apps/pm`, and bootstrap `apps/merchant` as a **NestJS Hybrid Application**:
- `apps/merchant` listens on HTTP port `8081` (`GET /check/:merchant_id`) to satisfy direct REST callers and external E2E tests (`200 OK` / `404 Not Found`).
- In the same `bootstrap()` function, `apps/merchant` registers `app.connectMicroservice<MicroserviceOptions>({ transport: Transport.TCP, options: { host: '0.0.0.0', port: 8881 } })` and starts both listeners via `await app.startAllMicroservices()`.
- Both entrypoints share the exact same `MerchantService` and database connection pool.

---

### ADR-03: Inter-service communication (`PM <-> Merchant CHECK`)

#### Context
Every new `POST /sale` request on `PM` must validate `merchant_id` against the `CHECK` service of the `Merchant` application. We must evaluate how `PM` communicates with `Merchant` and how failures or latency spikes are contained.

#### Options evaluated

| Transport Option | Latency (Local / Docker Bridge) | Infrastructure Footprint | Protocol Characteristics |
| :--- | :--- | :--- | :--- |
| **Option A: `@nestjs/microservices` TCP `ClientProxy` + HTTP Fallback (Selected)** | `0.3 ms` to `0.8 ms` over persistent TCP socket | Zero extra broker required for baseline | Multiplexed JSON-over-TCP request/response (`@MessagePattern({ cmd: 'merchant.check' })`) with optional HTTP REST (`GET /check/:merchant_id`) adapter switch |
| **Option B: Pure HTTP REST (`undici` `GET /check/:id`)** | `1.0 ms` to `2.5 ms` | Zero extra broker | Standard HTTP/1.1 keep-alive; slightly higher header parsing overhead |
| **Option C: Redis / NATS / RabbitMQ / Kafka Broker** | `1.5 ms` to `5.0 ms` (2 network hops via broker) | Requires running broker for every RPC check | Adds unnecessary broker hop for a synchronous point-to-point `merchant_id` existence lookup |
| **Option D: gRPC (`Transport.GRPC`)** | `0.4 ms` to `0.9 ms` | Requires `.proto` build pipeline | Strong typing, but TypeScript monorepo shared contracts (`@dlocal/shared-contracts`) already enforce compile-time types without `.proto` codegen |

#### Decision
Implement a **Strategy-wrapped `MerchantCheckClient`** in `apps/pm` that uses **`@nestjs/microservices` `ClientProxy` over `Transport.TCP` (`port 8881`)** as the primary internal transport and supports HTTP REST (`GET http://merchant:8081/check/:merchant_id` via `undici`) via configuration (`MERCHANT_TRANSPORT=tcp | http`):
1. **Message pattern:** `MerchantController` decorates the handler with both `@Get('check/:merchant_id')` (for HTTP REST) and `@MessagePattern({ cmd: 'merchant.check' })` (for `@nestjs/microservices` TCP RPC).
2. **Resiliency & caching bounds:** In `apps/pm`, `MerchantCheckClient` wraps the outbound call in a `cockatiel` policy enforcing an **`800 ms` hard timeout**, **maximum 2 retries with `100 ms` base exponential backoff** on transport errors (`ECONNREFUSED`, timeout), and a circuit breaker that opens after **5 consecutive transport failures** for a **`15,000 ms` cooldown**.
3. **Business status handling:** A `{ exists: false }` RPC response (or HTTP `404 Not Found`) is a valid business outcome that immediately throws `NotFoundException` (`HTTP 404`) to the caller and **never** increments the circuit breaker failure counter.
4. **Read-through cache fallback (Phase 2+):** Verified merchants are cached in an L1 in-memory LRU cache (`60,000 ms` TTL) and L2 Redis cache (`900,000 ms` TTL) so `PM` can validate known merchants when the circuit breaker is open.

---

### ADR-04: Database engine, ORM / query builder, and schema isolation

#### Context
The specification lists MySQL 5.x and requires storing merchants and sales, enforcing idempotency on `(merchant_id, transaction_id)`, aggregating USD balances by merchant and date range, and updating `PENDING` sales every 30 seconds. Furthermore, the ORM or query builder must work cleanly with TypeScript v7 (`tsgo`).

#### Options evaluated

| Persistence Stack | `tsgo` Compatibility | SQL Control (`SKIP LOCKED`, `SUM(CASE...)`) | Runtime Overhead |
| :--- | :--- | :--- | :--- |
| **Option 1: MySQL 8.0 (InnoDB) + Drizzle ORM + `mysql2` (Selected)** | 100% native TypeScript inference; zero `experimentalDecorators` required on entities | Direct, type-safe SQL builder for `FOR UPDATE SKIP LOCKED` and conditional `SUM` | Minimal wrapper over `mysql2/promise` prepared statements |
| **Option 2: MySQL 8.0 + TypeORM** | Relies heavily on class property decorators (`@Column()`, `@Entity()`) and `emitDecoratorMetadata` | Requires raw string queries for `SKIP LOCKED` | Higher object hydration overhead on large batch reads |
| **Option 3: MySQL 8.0 + Prisma** | Requires external `.prisma` codegen step before `tsgo` can run | No native query builder support for `FOR UPDATE SKIP LOCKED` without `$queryRaw` | Separate Rust query engine binary or heavy adapter layer |
| **Option 4: PostgreSQL / SQLite** | High | High in PG; no `SKIP LOCKED` in SQLite | Deviates from the MySQL engine specified in `docs/requirements.md` |

#### Decision
Select **MySQL 8.0 (InnoDB)** with **Drizzle ORM (`drizzle-orm/mysql2`)** and **`mysql2/promise`** connection pools:
1. **`tsgo` alignment:** Drizzle defines schemas as standard TypeScript constants (`mysqlTable(...)`). `tsgo` checks schema and query types directly without code generation or legacy property decorator metadata.
2. **Service data isolation:** Deploy a single MySQL 8.0 server in `docker-compose.yml` provisioned with two logical schemas (`merchant_db` and `pm_db`, or distinct credentials per app) so `apps/pm` cannot query the `merchants` table directly and must validate `merchant_id` via `apps/merchant` as required by `docs/requirements.md`.
3. **Connection pool bulkheads:** Configure `apps/merchant` with a pool limit of `20` connections and `apps/pm` with `30` connections (`queueLimit: 100`, `connectTimeout: 5000 ms`).

---

### ADR-05: `fixer.io` 30-second rate-limit and caching strategy

#### Context
`docs/requirements.md` states: *"El `amount` debe persistirse siempre en USD, utilizando el servicio de fixer.io para obtener la cotización de la moneda"* and *"El servicio de fixer.io no debe ser consultado más de una vez cada 30 segundos."* Under concurrent traffic (for example, 100 `POST /sale` requests arriving in millisecond 0 when the cache is cold or expired), a naive cache check (`if (!cache) fetch()`) will trigger 100 simultaneous requests to `fixer.io`, violating the 30-second constraint.

#### Options evaluated

| Caching & Rate-Limit Strategy | Single-Instance Concurrency Safe? | Multi-Replica Safe? | Outbound Calls when Idle? |
| :--- | :--- | :--- | :--- |
| **Option A: Single-Flight Coalescing + L1 Memory (`30s`) + L2 Redis Lock/TTL (`30s`) + DB Snapshot (Selected)** | Yes (`inFlightPromise` coalesces concurrent callers) | Yes (Redis `SET fx:lock NX PX 30000` coordinates replicas) | Zero calls when no sales arrive (lazy on-demand with 30s floor) |
| **Option B: Naive In-Memory Timestamp Check** | No (thundering herd on cache miss) | No (each replica calls `fixer.io` independently) | Zero when idle |
| **Option C: Unconditional 30s Background Cron Poller** | Yes | Requires leader election | Wastes `fixer.io` API quota (`2,880` calls/day) even when traffic is zero |

#### Decision
Implement a **Three-Tier Rate-Limited `FixerService`**:
1. **Tier 1 (In-Process Single-Flight + L1 TTL Cache):**
   - Stores `cachedRates: Record<string, string>`, `lastFetchedAtMs: number`, and `inFlightPromise: Promise<FxRatesSnapshot> | null`.
   - If `nowMs - lastFetchedAtMs < 30_000`, `FixerService` returns `cachedRates` in `<0.05 ms` with zero network I/O.
   - If `nowMs - lastFetchedAtMs >= 30_000` and `inFlightPromise !== null`, all concurrent callers await the existing `inFlightPromise`, guaranteeing **at most 1 outbound HTTP call per 30,000 ms window**.
2. **Tier 2 (Distributed L2 Redis Coordination for Multi-Replica Deployments):**
   - Before calling `fixer.io`, `FixerService` checks Redis key `fx:rates:current` (TTL `30,000 ms`). On L2 miss, it acquires a short mutex (`SET fx:rates:fetch_lock <instanceId> NX PX 3000`), fetches `fixer.io` (`2,000 ms` timeout), writes `fx:rates:current` with `PX 30000`, and publishes the update.
3. **Tier 3 (Stale-While-Revalidate Fallback & Base Currency Normalization):**
   - Every successful `fixer.io` payload is persisted asynchronously to `fx_rates_snapshot` in MySQL. If `fixer.io` returns `5xx` or times out after the 30-second TTL expires, `FixerService` serves the last persisted snapshot (for up to `3,600,000 ms` max staleness) instead of dropping checkout traffic.
   - Because `fixer.io` default endpoints return rates relative to `EUR` (or `USD` on paid plans), `FixerService` normalizes any input currency `C` to `USD` via cross-rate division:
     $$\text{amount\_usd} = \text{round\_half\_up}\left(\text{amount} \times \frac{\text{rate}[\text{'USD'}]}{\text{rate}[C]}, 2\right)$$
   - If `currency === 'USD'` and a valid cached snapshot exists, conversion returns `amount` directly; if no snapshot has been fetched in the last 30 seconds or if `STRICT_FIXER_CHECK=true`, the service still warms the 30-second cache consistently.

---

### ADR-06: 30-second batch clearing engine

#### Context
`docs/requirements.md` requires a batch process that runs every 30 seconds, selects all transactions in `PENDING` status, transitions each transaction to `PAID` with 70% probability or `REJECTED` with 30% probability, and persists the updated status.

#### Options evaluated

| Batch Architecture | Setup Complexity | Concurrency & Lock Contention | Horizontal Scalability |
| :--- | :--- | :--- | :--- |
| **Option 1: `@nestjs/schedule` `@Interval(30_000)` + Chunked `FOR UPDATE SKIP LOCKED` + Bulk `CASE` Update (Selected)** | Built into `apps/pm` with zero extra worker processes required | Locks at most `1,000` rows per sub-batch (`<15 ms` lock hold); skips rows locked by other workers | Multiple `pm` instances can run the interval simultaneously without double-processing |
| **Option 2: Naive `SELECT * FROM sales WHERE status = 'PENDING'` + N individual `UPDATE` statements** | Simple | Loads unbounded rows into V8 heap; N round-trips cause multi-second table lock contention | Breaks under multiple instances (race conditions overwrite statuses) |
| **Option 3: BullMQ Delayed Jobs per Sale** | Requires Redis queue management for every sale | Moves scheduling out of SQL | Schedules per-item timers rather than a unified 30-second batch sweep as specified |

#### Decision
Implement `BatchClearingJob` in `apps/pm` using `@nestjs/schedule` (`@Interval(30_000)`):
1. **In-process overlap guard:** An atomic `isRunning` boolean flag prevents overlapping executions within the same Node.js process if a large backlog takes longer than 30 seconds.
2. **Chunked `FOR UPDATE SKIP LOCKED` sweep:**
   - Inside a database transaction, execute:
     ```sql
     SELECT id FROM sales
     WHERE status = 'PENDING'
     ORDER BY created_at ASC
     LIMIT 1000
     FOR UPDATE SKIP LOCKED;
     ```
   - Partition the selected IDs in memory using an injected `RandomProvider` (`nextFloat(): number` in $[0, 1)$):
     - If `randomProvider.nextFloat() < 0.70`, assign `id` to `paidIds`.
     - Otherwise (`>= 0.70`), assign `id` to `rejectedIds`.
   - Execute at most two set-based `UPDATE` statements per chunk (`UPDATE sales SET status = 'PAID', updated_at = NOW(3) WHERE id IN (...)` and `UPDATE sales SET status = 'REJECTED', updated_at = NOW(3) WHERE id IN (...)`), commit the transaction, and repeat until a chunk returns `< 1,000` rows.
3. **Deterministic testability:** In unit and E2E tests, override `RANDOM_PROVIDER` with a seeded Mulberry32 PRNG or fixed sequence generator to verify exact `70% PAID / 30% REJECTED` transitions deterministically.
