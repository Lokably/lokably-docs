# 🧱 `lokably-shared`

> The foundation crate. Every other crate in the workspace depends on this one.

---

## 📋 Table of Contents

- [Purpose](#-purpose)
- [Module Map](#-module-map)
- [Modules](#-modules)
  - [`config`](#-config)
  - [`errors`](#-errors)
  - [`db`](#-db)
  - [`redis`](#-redis)
  - [`telemetry`](#-telemetry)
  - [`metrics`](#-metrics)
  - [`queue`](#-queue)
- [Usage Patterns](#-usage-patterns)
- [Testing](#-testing)
- [Design Decisions](#-design-decisions)

---

## 🎯 Purpose

`lokably-shared` provides **infrastructure primitives** consumed by both `lokably-api` and `lokably-worker`. It contains:

- ❌ **No business logic**
- ❌ **No HTTP handlers**
- ❌ **No domain models** (leads, invoices, users)
- ✅ Configuration loading
- ✅ Error types
- ✅ DB / Redis pool setup
- ✅ Tracing + metrics setup
- ✅ Job queue primitives

> 💡 **Rule of thumb:** if it's specific to *what* Lokably does (CRM, finance, payments), it lives in `lokably-api` or `lokably-worker`. If it's about *how* the app runs (config, logging, errors), it lives here.

---

## 🗺️ Module Map

```
lokably-shared/
└── src/
    ├── lib.rs          # Re-exports
    ├── config.rs       # ① Typed env loader
    ├── errors.rs       # ② Unified error type
    ├── db.rs           # ③ Postgres pool
    ├── redis.rs        # ④ Redis pool + helpers
    ├── telemetry.rs    # ⑤ Tracing setup
    ├── metrics.rs      # ⑥ Prometheus registry
    └── queue.rs        # ⑦ Redis Streams jobs
```

---

## 📦 Modules

### ⚙️ `config`

Loads and validates all environment variables at startup into a typed `Config` struct.

#### What It Does

1. Reads `.env` file if present
2. Parses every env var into its correct type (`u16`, `i64`, enums)
3. Validates constraints (e.g., `JWT_SECRET` ≥ 32 chars)
4. **Panics with a clear message** if anything is missing or invalid

> ⚠️ **Why panic instead of returning a Result?**
> An app cannot run with invalid config. Failing fast at startup is far better than crashing later in a request handler with a missing-env-var error.

#### Public API

```rust
pub struct Config {
    pub app:      AppConfig,
    pub db:       DbConfig,
    pub redis:    RedisConfig,
    pub auth:     AuthConfig,
    pub ai:       AiConfig,
    pub obs:      ObsConfig,
    pub payments: PaymentsConfig,
    pub email:    EmailConfig,
}

impl Config {
    pub fn load() -> Self;           // Reads .env + parses env vars
    pub fn load_from_env() -> Self;  // Skips .env (for tests)
    pub fn is_production(&self) -> bool;
    pub fn is_development(&self) -> bool;
}
```

#### Enums

| Enum | Values | Env Var |
|------|--------|---------|
| `AppEnv` | `Development` \| `Production` | `APP_ENV` |
| `AiProvider` | `Ollama` \| `Claude` | `AI_PROVIDER` |
| `LogFormat` | `Pretty` \| `Json` | `LOG_FORMAT` |
| `EmailProvider` | `Mailpit` \| `Resend` | `EMAIL_PROVIDER` |

#### Required vs Optional Variables

| Required | Optional (with defaults) |
|----------|--------------------------|
| `APP_ENV` | `DATABASE_MAX_CONNECTIONS` → `20` |
| `SERVICE_NAME` | `JWT_EXPIRY_HOURS` → `24` |
| `SERVICE_PORT` | `LOG_LEVEL` → `info` |
| `FRONTEND_URL` | `LOG_FORMAT` → `pretty` |
| `DATABASE_URL` | `METRICS_ENABLED` → `true` |
| `REDIS_URL` | `OLLAMA_BASE_URL` → `http://localhost:11434` |
| `JWT_SECRET` (min 32 chars) | `OLLAMA_MODEL` → `llama3.2` |
| `AI_PROVIDER` | `EMAIL_PROVIDER` → `mailpit` |
| | `SMTP_PORT` → `1025` |

#### Example

```rust
use lokably_shared::Config;

#[tokio::main]
async fn main() {
    let config = Config::load(); // panics if invalid

    if config.is_production() {
        println!("Running in production mode");
    }

    println!("Listening on port {}", config.app.port);
}
```

---

### 🚨 `errors`

A unified error type used across the entire workspace. Every variant maps to an HTTP status and a structured JSON response.

#### Public API

```rust
pub type LokablyResult<T> = Result<T, LokablyError>;

#[derive(thiserror::Error, Debug)]
pub enum LokablyError {
    NotFound(String),       // → 404
    Unauthorized,           // → 401
    Forbidden,              // → 403
    Validation(String),     // → 422
    Conflict(String),       // → 409
    QuotaExceeded(String),  // → 429
    Database(sqlx::Error),  // → 500 (details hidden)
    Redis(redis::Error),    // → 500 (details hidden)
    AiError(String),        // → 500
    PaymentError(String),   // → 500
    Internal(String),       // → 500 (details hidden)
}
```

#### Response Shape

Every error returns the same JSON body:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "lead xyz not found"
  }
}
```

#### Status Code Mapping

| Variant | HTTP Status | Code | Details Leaked? |
|---------|-------------|------|-----------------|
| `NotFound` | 404 | `NOT_FOUND` | ✅ Yes |
| `Unauthorized` | 401 | `UNAUTHORIZED` | ✅ Yes |
| `Forbidden` | 403 | `FORBIDDEN` | ✅ Yes |
| `Validation` | 422 | `VALIDATION_ERROR` | ✅ Yes |
| `Conflict` | 409 | `CONFLICT` | ✅ Yes |
| `QuotaExceeded` | 429 | `QUOTA_EXCEEDED` | ✅ Yes |
| `Database` | 500 | `DATABASE_ERROR` | ❌ **Hidden** |
| `Redis` | 500 | `REDIS_ERROR` | ❌ **Hidden** |
| `Internal` | 500 | `INTERNAL_ERROR` | ❌ **Hidden** |
| `AiError` | 500 | `AI_ERROR` | ⚠️ Partial |
| `PaymentError` | 500 | `PAYMENT_ERROR` | ⚠️ Partial |

> 🛡️ **Security note:** Database, Redis, and Internal errors **never leak details to clients**. Real error messages go to logs only. Clients always see `"an internal error occurred"` for 500-class errors from these sources.

#### Logging Behavior

- **5xx errors** → logged at `ERROR` level with full details
- **4xx errors** → logged at `DEBUG` level (client problems, not server problems)

#### Example

```rust
use lokably_shared::{LokablyError, LokablyResult};

async fn find_lead(id: Uuid) -> LokablyResult<Lead> {
    let lead = sqlx::query_as!(Lead, "SELECT * FROM crm.leads WHERE id = $1", id)
        .fetch_optional(&pool)
        .await?  // sqlx::Error → LokablyError::Database via From impl
        .ok_or_else(|| LokablyError::NotFound(format!("lead {id}")))?;

    Ok(lead)
}
```

---

### 🗄️ `db`

PostgreSQL connection pool setup and migration runner.

#### Public API

```rust
pub type DbPool = sqlx::PgPool;

pub async fn create_pool(config: &DbConfig) -> DbPool;
pub async fn migrate(pool: &DbPool);
```

#### Connection Pool Settings

| Setting | Value | Why |
|---------|-------|-----|
| `max_connections` | configurable, default 20 | Tuned per workload |
| `acquire_timeout` | 10 seconds | Fail fast if pool exhausted |
| `idle_timeout` | 10 minutes | Free idle connections |

#### Migrations

Migrations live in **[`lokably-infra/migrations/`](https://github.com/lokably-platform/lokably-infra/tree/main/migrations)** and are run at startup via `sqlx::migrate!()` macro.

> 📍 The migration path in `db.rs` is **workspace-relative**: `../../../lokably-infra/migrations` — this works because:
> - The macro resolves at compile time relative to `Cargo.toml`
> - Both repos must be cloned side-by-side
> - Validates SQL at compile time

#### Example

```rust
use lokably_shared::{db, Config};

let config = Config::load();
let pool = db::create_pool(&config.db).await;
db::migrate(&pool).await;
```

---

### 📡 `redis`

Auto-reconnecting Redis connection + simple key/value helpers.

#### Public API

```rust
pub type RedisPool = redis::aio::ConnectionManager;

pub async fn create_redis_pool(config: &RedisConfig) -> RedisPool;

// Helpers
pub async fn get(pool: &RedisPool, key: &str) -> LokablyResult<Option<String>>;
pub async fn set(pool: &RedisPool, key: &str, value: &str, ttl_secs: u64) -> LokablyResult<()>;
pub async fn del(pool: &RedisPool, key: &str) -> LokablyResult<()>;
pub async fn exists(pool: &RedisPool, key: &str) -> LokablyResult<bool>;
```

#### Why `ConnectionManager`?

| Feature | Benefit |
|---------|---------|
| Auto-reconnect | Survives Redis restarts without app crashes |
| Cheap to clone | Wraps an `Arc` — share across async tasks freely |
| Async-native | Works directly with Tokio |

#### Example

```rust
use lokably_shared::redis;

// Cache a value for 5 minutes
redis::set(&pool, "session:abc123", &user_id.to_string(), 300).await?;

// Fetch later
if let Some(uid) = redis::get(&pool, "session:abc123").await? {
    println!("session belongs to {uid}");
}
```

---

### 📝 `telemetry`

Tracing initialization. Routes all `tracing::*` macros to stdout in either pretty (dev) or JSON (prod) format.

#### Public API

```rust
pub fn init(config: &ObsConfig);
```

> 🚨 **Must be called first in `main()`**, before anything else. Logs emitted before init are silently dropped.

#### Output Formats

##### Pretty Mode (`LOG_FORMAT=pretty`)

Used in dev. Colored, human-readable terminal output:

```
2026-05-16T10:23:11Z  INFO lokably_api::auth: User logged in
    user_id="abc-123" latency_ms=42
```

##### JSON Mode (`LOG_FORMAT=json`)

Used in production. One structured object per line — picked up by Promtail, shipped to Loki, queryable in Grafana:

```json
{
  "timestamp": "2026-05-16T10:23:11.234Z",
  "level": "INFO",
  "service": "lokably-api",
  "module": "auth",
  "trace_id": "abc-123",
  "user_id": "abc-123",
  "company_id": "xyz-789",
  "latency_ms": 42,
  "message": "User logged in"
}
```

#### Filtered Crates

These noisy dependencies are forced to `warn` level so they don't drown out application logs:

```
sqlx → warn
hyper → warn
tower → warn
tower_http → warn
h2 → warn
reqwest → warn
```

#### Example

```rust
use lokably_shared::{telemetry, Config};
use tracing::{info, warn};

let config = Config::load();
telemetry::init(&config.obs);

info!("server starting on port {}", config.app.port);
warn!(retries = 3, "failed to send email");
```

---

### 📊 `metrics`

Prometheus metrics registry. Exposes a `/metrics` endpoint via Axum.

#### Public API

```rust
pub fn init(config: &ObsConfig) -> PrometheusHandle;

// Helpers
pub fn inc_counter(name: &'static str, labels: &[(&'static str, String)]);
pub fn record_histogram(name: &'static str, value: f64, labels: &[(&'static str, String)]);
pub fn set_gauge(name: &'static str, value: f64, labels: &[(&'static str, String)]);
```

#### Registered Metrics

##### HTTP Metrics (set by middleware)

| Metric | Type | Labels |
|--------|------|--------|
| `http_requests_total` | Counter | `method`, `path`, `status` |
| `http_request_duration_seconds` | Histogram | `method`, `path` |

##### Business Metrics (set by handlers)

| Metric | Type | Labels |
|--------|------|--------|
| `lokably_ai_queries_total` | Counter | `provider` |
| `lokably_ai_query_duration_seconds` | Histogram | `provider` |
| `lokably_transactions_total` | Counter | `status` |
| `lokably_subscriptions_active` | Gauge | `plan` |
| `lokably_jobs_total` | Counter | `job_type`, `status` |
| `lokably_job_duration_seconds` | Histogram | `job_type` |
| `lokably_queue_depth` | Gauge | — |

#### Histogram Buckets

Tuned for HTTP latency (seconds):

```
0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0
```

#### Example

```rust
use lokably_shared::metrics;

// Initialize at startup
let handle = metrics::init(&config.obs);

// In a handler — record an AI query
metrics::inc_counter(
    metrics::AI_QUERIES_TOTAL,
    &[("provider", "claude".to_string())],
);

metrics::record_histogram(
    metrics::AI_QUERY_DURATION_SECONDS,
    0.42,
    &[("provider", "claude".to_string())],
);

// Expose /metrics endpoint in Axum
let app = Router::new()
    .route("/metrics", get(move || async move { handle.render() }));
```

---

### 📬 `queue`

Redis Streams–based job queue. The API publishes jobs; the worker consumes and processes them.

#### Public API

```rust
pub struct Queue { /* ... */ }

impl Queue {
    pub fn new(redis: RedisPool) -> Self;

    pub async fn publish(&self, job: &LokablyJob) -> LokablyResult<()>;
    pub async fn consume(&self) -> LokablyResult<Vec<(String, LokablyJob)>>;
    pub async fn ack(&self, message_id: &str) -> LokablyResult<()>;
    pub async fn depth(&self) -> LokablyResult<i64>;
    pub async fn ensure_group(&self) -> LokablyResult<()>;
}
```

#### Stream Layout

```
Stream:         lokably:jobs
Consumer Group: lokably-workers
```

#### Job Types

```rust
pub enum LokablyJob {
    SendEmail        { to, template },
    GenerateInsights { company_id },
    ScoreLeads       { company_id },
    DetectChurn      { company_id },
    FlagExpenses     { company_id },
    SplitTransaction { transaction_id },
    HandleSubscriptionWebhook { event_type, payload },
    HandlePaymentWebhook      { event_type, payload },
    ResetMonthlyUsage         { company_id },
    GenerateMonthlyReport     { company_id },
    SendInvoiceReminder       { invoice_id },
}
```

#### Email Templates

```rust
pub enum EmailTemplate {
    Welcome         { name },
    PasswordReset   { token },
    InvoiceReminder { invoice_id, amount },
    ChurnWarning    { company_name },
    PaymentFailed   { plan },
}
```

#### Consumer Semantics

| Operation | Behavior |
|-----------|----------|
| `publish` | Adds job to stream with auto-generated ID |
| `consume` | Blocks up to 1s, reads up to 10 messages |
| `ack` | Removes from pending-acks (call after successful processing) |
| `depth` | Returns total messages in the stream |
| `ensure_group` | Idempotent — creates group + stream if missing |

> 💡 **Why ack manually?**
> If processing fails, the message stays in the consumer group's pending list. On worker restart, it gets re-delivered. This guarantees at-least-once processing.

#### Example: Publishing a Job (API side)

```rust
use lokably_shared::{Queue, LokablyJob, EmailTemplate};

let queue = Queue::new(redis_pool.clone());

queue.publish(&LokablyJob::SendEmail {
    to: "alice@example.com".into(),
    template: EmailTemplate::Welcome {
        name: "Alice".into(),
    },
}).await?;
```

#### Example: Consuming Jobs (Worker side)

```rust
use lokably_shared::{Queue, LokablyJob};

let queue = Queue::new(redis_pool);
queue.ensure_group().await?;

loop {
    let jobs = queue.consume().await?;

    for (id, job) in jobs {
        match process_job(&job).await {
            Ok(_) => queue.ack(&id).await?,
            Err(e) => tracing::error!(error = %e, "job failed, will retry"),
        }
    }
}
```

---

## 🧩 Usage Patterns

### Standard Startup Sequence

Every binary in the workspace follows this exact pattern:

```rust
use lokably_shared::{config::Config, db, redis, telemetry, metrics};

#[tokio::main]
async fn main() {
    // 1️⃣ Load config FIRST — panics if invalid
    let config = Config::load();

    // 2️⃣ Initialize tracing SECOND — so all later steps are logged
    telemetry::init(&config.obs);

    // 3️⃣ Initialize metrics
    let metrics_handle = metrics::init(&config.obs);

    // 4️⃣ Connect to infrastructure
    let pool = db::create_pool(&config.db).await;
    db::migrate(&pool).await;
    let redis_pool = redis::create_redis_pool(&config.redis).await;

    // 5️⃣ Start the application
    // ... (server-specific logic)
}
```

### Convenience Re-exports

For ergonomic imports, common types are re-exported at the crate root:

```rust
// Instead of:
use lokably_shared::config::Config;
use lokably_shared::errors::{LokablyError, LokablyResult};
use lokably_shared::db::DbPool;

// You can write:
use lokably_shared::{Config, LokablyError, LokablyResult, DbPool};
```

---

## 🧪 Testing

### Test Files

| File | Tests | Requires |
|------|-------|----------|
| `tests/config_test.rs` | 7 tests | Nothing |
| `tests/errors_test.rs` | 7 tests | Nothing |
| `tests/redis_test.rs` | 4 tests | Redis |
| `tests/queue_test.rs` | 4 tests | Redis |

### Running

```bash
# All tests (requires Redis from lokably-infra)
cargo test -p lokably-shared

# Skip integration tests (unit-only)
SKIP_INTEGRATION_TESTS=1 cargo test -p lokably-shared

# Specific test file
cargo test -p lokably-shared --test config_test

# With output (useful for debugging)
cargo test -p lokably-shared -- --nocapture
```

### Test Patterns

#### Config Tests — Serial Execution

Config tests mutate process env vars. To prevent races, they use `serial_test`:

```rust
#[test]
#[serial]
fn loads_minimal_required_env() { /* ... */ }
```

The `ENV_LOCK` mutex is **poison-safe** — if a test panics while holding it (e.g., `should_panic` tests), subsequent tests still acquire normally.

#### Integration Tests — Skip Cleanly

Redis/Queue tests check `SKIP_INTEGRATION_TESTS` and return early when Redis isn't available, so CI without infrastructure still passes.

---

## 🏗️ Design Decisions

### Why Panic on Bad Config?

> **Q:** Why doesn't `Config::load()` return a `Result`?
>
> **A:** Because the app **cannot run** without valid config. Returning a `Result` would just mean every caller writes `.expect()` anyway. Panic at startup is the cleanest failure mode — clear stack trace, before any user requests hit the server.

### Why Hide Database/Internal Errors from Clients?

> **Q:** Why does the error response say `"an internal error occurred"` instead of the real error?
>
> **A:** Two reasons:
> 1. **Security** — DB errors often leak schema info, column names, or connection strings
> 2. **Stability** — Clients shouldn't depend on internal error wording that could change

> The full error is **always logged** with full detail. Operators see everything; users see nothing sensitive.

### Why Redis Streams Over a Real Queue?

> **Q:** Why not RabbitMQ / NATS / SQS?
>
> **A:** We already need Redis for caching + rate limiting + sessions. Adding a second piece of infra for a queue would be wasteful at this scale. Redis Streams give us:
> - At-least-once delivery
> - Consumer groups
> - Persistence
> - Replay capability
>
> When we outgrow it (probably never for early-stage), migrating to a dedicated queue is straightforward — `Queue` is already an abstraction.

### Why Two `Config::load*` Methods?

> **Q:** Why `load()` and `load_from_env()`?
>
> **A:** `load()` reads `.env` then parses env vars — perfect for `main()`. But in tests, `.env` would reload removed vars, breaking "missing var" tests. `load_from_env()` skips dotenvy entirely, giving tests full control of env state.

---

## 📚 See Also

- 🏗️ [Lokably API README](../../README.md)
- 🐳 [Infrastructure setup](https://github.com/lokably-platform/lokably-infra)
- 🤖 [AI prompts](https://github.com/lokably-platform/lokably-prompts)

---

<div align="center">

**🧱 Foundation crate · v0.1.0**

</div>