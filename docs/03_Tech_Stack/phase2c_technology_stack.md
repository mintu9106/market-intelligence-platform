# Phase 2C — Technology Stack Version 1.0
## AI-Powered Indian Stock Market Intelligence Platform

> **Status**: Pending User Approval
> **Version**: 1.0
> **Depends On**: Phase 2A (Architecture), Phase 2B (Data Flows) — Both Approved
> **Next Phase**: Phase 3 — Database Design

---

## Technology Selection Philosophy

Every technology in this stack was evaluated against six criteria:

| Criterion | Weight | Rationale |
|---|---|---|
| **Production Maturity** | High | No bleeding-edge experiments in a financial platform |
| **Scalability** | High | Must support 50,000+ users and 5,000+ concurrent sessions |
| **AI/ML Ecosystem** | High | Python-first ecosystem required for ML workloads |
| **Security Track Record** | High | Financial data demands battle-tested security |
| **Total Cost of Ownership** | Medium | SaaS economics must be viable at scale |
| **Talent Availability (India)** | Medium | Engineering team must be hireable in the Indian market |

> **Principle**: Choose boring, proven technology over exciting, novel technology. Innovation happens at the business logic layer — not the infrastructure layer.

---

## 1. Frontend

### 1A — Web Dashboard

**Selected: Next.js 14 (React — App Router)**

**Why Selected**
Next.js is the de-facto production standard for complex React applications. The App Router (introduced in Next.js 13) enables granular server-side rendering decisions at the component level — critical for a financial dashboard where some data must be server-rendered (for SEO and initial load speed) and some must be client-rendered (live charts, WebSocket feeds). TypeScript support is first-class, enabling strict type safety across the frontend.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **React (CRA/Vite)** | No SSR out of the box; SEO disadvantage; more boilerplate |
| **Angular** | Steeper learning curve; slower development velocity; overkill for this use case |
| **Vue.js / Nuxt** | Smaller ecosystem for financial UI libraries; fewer senior devs in India |
| **Remix** | Excellent framework but smaller ecosystem; fewer production reference deployments |
| **SvelteKit** | Excellent performance but very small ecosystem; hiring risk |

**Pros**
- Server-side rendering improves initial load and SEO for public-facing pages
- App Router enables Partial Pre-rendering (static + dynamic on same page)
- Excellent TypeScript support out of the box
- Huge ecosystem: shadcn/ui, Radix UI, React Query, Zustand all have Next.js-native support
- Vercel-optimized but deployable anywhere (Docker, Kubernetes)
- React Server Components reduce client-side JavaScript bundle
- Built-in image optimization, font optimization, and code splitting

**Cons**
- App Router has a learning curve vs Pages Router
- Server Components add complexity to state management patterns
- Cold start latency in serverless deployments (mitigated by self-hosting on Kubernetes)
- Opinionated directory structure

**Supporting Libraries**
| Library | Purpose | Why |
|---|---|---|
| **TradingView Lightweight Charts** | Financial charting | Industry-standard; free for SaaS; 60fps WebGL rendering |
| **Zustand** | Client state management | Lightweight (< 1KB); no boilerplate; scales well |
| **TanStack Query (React Query)** | Server state + caching | Best-in-class for REST data fetching, caching, background refresh |
| **shadcn/ui + Radix UI** | Component library | Accessible; headless; fully customizable; no vendor lock-in |
| **Socket.io-client** | WebSocket client | Fallback to polling; reconnection logic built-in |
| **Recharts / Nivo** | Analytics charts | Portfolio charts, heatmaps (lighter than TradingView) |
| **React Hook Form + Zod** | Form validation | Type-safe forms with schema validation |
| **Framer Motion** | Animations | Production-grade animation library for React |

---

### 1B — Mobile Application

**Selected: React Native (with Expo)**

**Why Selected**
React Native enables code and team sharing with the Next.js web dashboard — same TypeScript language, same component patterns, same state management. The Expo framework removes native build configuration complexity, enabling faster iteration. For a market intelligence platform, mobile is primarily a consumption interface (alerts, portfolio, signals) — not a trading terminal — so native performance is not the critical constraint.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Flutter (Dart)** | Different language from web stack; no code sharing; separate team required |
| **Native iOS (Swift) + Native Android (Kotlin)** | 2x development cost; 2x maintenance; slower feature parity |
| **Ionic (Angular/React)** | WebView-based; poor performance for real-time financial data |
| **PWA only** | Limited push notification support on iOS; no App Store presence |

**Pros**
- Shared codebase with web team (React components, business logic, TypeScript)
- Expo managed workflow eliminates native build complexity
- Large library of pre-built components
- Strong India developer talent pool (React Native expertise)
- Over-the-Air (OTA) updates via Expo EAS (no App Store review for minor updates)
- Strong integration with FCM (Android) and APNs (iOS)

**Cons**
- Performance ceiling below true native for CPU-heavy tasks (not applicable here)
- Large app bundle size vs native
- Expo managed workflow has occasional limitations requiring ejection
- Platform-specific bugs require React Native expertise to debug

---

## 2. Backend Services

**Selected: Python + FastAPI (primary), Node.js + NestJS (WebSocket/real-time service)**

### 2A — Python FastAPI (Primary Backend)

**Why Selected**
Python is non-negotiable for an AI-powered platform — the ML ecosystem (PyTorch, scikit-learn, pandas, SHAP, MLflow) is Python-first. FastAPI is the highest-performance Python web framework, built on ASGI (async), with automatic OpenAPI documentation, Pydantic-based request/response validation, and native type hint support. Running the AI Engine and business services in the same language eliminates serialization overhead and simplifies the team skill set.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Django + DRF** | Synchronous by default; slower; ORM is excellent but FastAPI async wins on performance |
| **Flask** | Too minimal; requires too much assembly; no async out of box |
| **Go (Gin/Echo)** | Excellent performance but no ML ecosystem; would require Python microservice for AI anyway |
| **Java (Spring Boot)** | Verbose; JVM startup overhead; Python ML integration awkward; hiring cost in India |
| **Node.js (Express)** | Poor for CPU-bound ML workloads; Python data science libraries unavailable |
| **Rust (Actix)** | Extreme performance but no ML ecosystem; hiring near-impossible |

**Pros**
- ASGI + async/await: handles high concurrency without threading complexity
- Pydantic: automatic request validation, serialization, and OpenAPI schema generation
- Native Python: direct integration with all ML/data libraries (no bridge needed)
- 3–5x faster than Flask/Django for API throughput (benchmarked)
- Automatic interactive API docs (`/docs`, `/redoc`)
- Dependency injection system (FastAPI `Depends`) for clean architecture
- Strong Indian developer community and talent availability

**Cons**
- Python GIL limits true CPU parallelism (mitigated by async I/O + multiprocessing for CPU tasks)
- Higher memory footprint per worker vs Go or Rust
- Not ideal for WebSocket at massive scale (handled by NestJS instead)

### 2B — Node.js + NestJS (WebSocket Real-Time Service)

**Why Selected**
Node.js's event loop architecture is specifically optimized for WebSocket connections — it can maintain 10,000+ concurrent WebSocket connections per instance efficiently. NestJS provides a structured, opinionated framework (dependency injection, decorators, modules) that scales teams well. This service handles only real-time data fan-out — not business logic — keeping it lean and single-purpose.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Python FastAPI WebSocket** | Works but event loop saturation under 5,000+ concurrent connections |
| **Go WebSocket server** | Better raw performance but adds a third language to the stack |
| **Elixir/Phoenix Channels** | World-class for WebSockets but near-zero hiring pool in India |
| **Erlang** | Same hiring problem as Elixir |

**Pros**
- Node.js event loop is purpose-built for high-concurrency I/O (WebSocket)
- NestJS's module system mirrors Angular — familiar to enterprise developers
- Socket.io integration provides automatic fallback, reconnection handling, and namespaces
- Stateless WebSocket server horizontally scales easily behind sticky-session load balancer
- Redis adapter for Socket.io enables horizontal scaling across multiple Node.js pods

**Cons**
- Adds JavaScript/TypeScript to the backend stack (now Python + JS)
- NestJS can be over-engineered for simple use cases
- Node.js single-threaded; CPU-heavy tasks must be offloaded to worker threads

---

## 3. AI / Machine Learning Framework

**Selected: PyTorch (primary), XGBoost, scikit-learn**

### 3A — PyTorch

**Why Selected**
PyTorch is the leading deep learning framework in both academic research and production deployment, having surpassed TensorFlow in adoption since 2022. Its dynamic computation graph (define-by-run) makes debugging intuitive and model architecture iteration fast. PyTorch 2.x's `torch.compile()` provides significant inference speedups without model changes. LSTM and Transformer models for price direction prediction are best served by PyTorch.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **TensorFlow 2.x / Keras** | Still production-viable but losing ecosystem momentum vs PyTorch; TF 1.x legacy baggage |
| **JAX** | Cutting-edge but small ecosystem; fewer pre-trained financial models |
| **ONNX Runtime only** | Inference only; can't train new models |

**Pros**
- Most popular framework in ML research → production pipeline is mature
- Dynamic graphs: easier debugging, flexible architectures
- `torch.compile()`: up to 2x inference speedup (PyTorch 2.0+)
- Hugging Face Transformers deeply integrated with PyTorch (needed for FinBERT)
- TorchServe for model serving in production
- Strong GPU utilization (CUDA support)
- Largest community and hiring pool

**Cons**
- Higher memory usage vs TensorFlow for some model types
- Deployment to production requires TorchServe or ONNX export
- Not as beginner-friendly as Keras-based TensorFlow

### 3B — XGBoost (Gradient Boosting)

**Why Selected**
For tabular financial data (breakout prediction, signal scoring), gradient boosting consistently outperforms deep learning. XGBoost is the fastest, most memory-efficient gradient boosting library. It supports GPU acceleration, native handling of missing values, and integrates seamlessly with scikit-learn's API.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **LightGBM** | Marginally faster but less ecosystem support; very similar quality |
| **CatBoost** | Excellent for categorical features but slower; overkill for our feature set |
| **Random Forest** | Lower accuracy ceiling; slower inference |

**Pros**
- Best-in-class accuracy for tabular financial features
- GPU acceleration (`device='cuda'` parameter)
- Built-in cross-validation, early stopping, feature importance
- SHAP integration: XGBoost + SHAP is the gold standard combination for XAI
- Handles missing values natively (no imputation required)
- scikit-learn API compatible

**Cons**
- Not suitable for sequential/time-series modeling (use LSTM for that)
- Hyperparameter sensitivity: requires careful tuning

### 3C — scikit-learn

**Why Selected**
scikit-learn provides the preprocessing, feature engineering, model selection, and evaluation infrastructure that underpins all ML models. It is the universal utility layer of the Python ML ecosystem.

**Use Cases on This Platform**
- StandardScaler, MinMaxScaler (feature normalization)
- Pipeline API (preprocessing → model as single unit)
- Cross-validation, grid search (hyperparameter tuning)
- Isolation Forest (anomaly detection)
- PCA (dimensionality reduction for feature analysis)
- Classification metrics (accuracy, precision, recall, F1, AUC-ROC)

---

## 4. Python Libraries

**Core Libraries Specification**

| Library | Version (approx) | Purpose | Rationale |
|---|---|---|---|
| **pandas** | 2.x | Data manipulation, OHLCV processing | Industry standard; vectorized operations; TimescaleDB integration |
| **NumPy** | 1.26+ | Numerical computation | Foundation of all scientific Python; required by every other library |
| **TA-Lib** | 0.4.x | Technical indicator computation | C library (fast); 150+ indicators; battle-tested in production fintech |
| **pandas-ta** | Latest | Supplementary indicators | Pure Python fallback; some indicators not in TA-Lib |
| **SHAP** | 0.44+ | Explainable AI (XAI) | Industry standard; TreeExplainer + DeepExplainer; Plotly visualizations |
| **MLflow** | 2.x | ML experiment tracking + model registry | Open-source; self-hosted; experiment logging, model versioning, A/B |
| **Feast** | 0.38+ | Feature Store | Online (Redis) + Offline (BigQuery/S3) feature consistency |
| **Celery** | 5.x | Async task queue | Backtesting jobs, notification retries, model retraining |
| **SQLAlchemy** | 2.x | ORM + database abstraction | Async support; type-safe; Alembic migrations |
| **Alembic** | Latest | Database migrations | SQLAlchemy native; version-controlled schema changes |
| **Pydantic** | 2.x | Data validation + serialization | FastAPI native; 10-20x faster than v1; strict mode |
| **Transformers (HuggingFace)** | 4.x | NLP / FinBERT sentiment | Pre-trained FinBERT; fine-tuning pipeline; tokenizers |
| **httpx** | Latest | Async HTTP client | Async-native; replaces requests for FastAPI; connection pooling |
| **confluent-kafka** | Latest | Kafka Python client | Official Confluent client; librdkafka binding; production-grade |
| **redis-py** | 5.x | Redis client | Async support; cluster mode; Pub/Sub |
| **APScheduler** | Latest | In-process scheduling (supplementary) | Pre-market and post-market scheduled jobs |
| **scipy** | Latest | Statistical functions | VaR computation, correlation matrices, statistical tests |
| **pytest + pytest-asyncio** | Latest | Testing | Full async test support; fixtures; coverage |

---

## 5. Database

### 5A — PostgreSQL 16

**Selected as: Primary OLTP Database**

**Why Selected**
PostgreSQL is the gold standard for relational data in enterprise applications. For a multi-tenant SaaS platform, PostgreSQL's Row-Level Security (RLS) is a critical feature — it enforces tenant data isolation at the database layer (not just the application layer), providing defense-in-depth. JSONB support handles semi-structured data (strategy configurations, alert rules, AI explanation payloads). PostgreSQL 16 brings significant performance improvements for parallel query execution.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **MySQL / MariaDB** | No native RLS; JSONB less powerful; weaker for complex queries |
| **CockroachDB** | PostgreSQL-compatible but adds distributed complexity; overkill at 50k users |
| **PlanetScale (MySQL)** | No RLS; vendor lock-in; limited JSONB |
| **Supabase** | Good for MVPs but limited control at enterprise scale |

**Pros**
- Row-Level Security: tenant isolation enforced at DB layer
- JSONB: strategy configs, alert rules stored as semi-structured data efficiently
- Rich indexing: B-tree, GIN (JSONB), BRIN (time-ordered data), partial indexes
- Proven at massive scale (Instagram, GitHub use PostgreSQL)
- ACID compliance: critical for financial transaction records
- Read replicas: horizontal read scaling
- pgvector extension: vector similarity search (for future AI feature matching)
- Excellent managed options: AWS RDS, Supabase, Neon

**Cons**
- Not designed for time-series queries (handled by TimescaleDB)
- Vertical scaling ceiling (addressed by read replicas and caching)
- Schema migrations require careful planning (Alembic + zero-downtime migration patterns)

**What Lives in PostgreSQL**
- Users, tenants, subscriptions, billing records
- Portfolio holdings, transactions, orders (paper trading)
- Alert rules, alert history, notification delivery logs
- Signal history, backtest results (metadata)
- Strategy configurations per user
- Broker integration credentials (encrypted)

---

### 5B — TimescaleDB

**Selected as: Time-Series Database**

**Why Selected**
TimescaleDB is a PostgreSQL extension, not a separate database — this means the development team uses the same SQL, same ORM (SQLAlchemy), same connection pool, and same tooling. It adds automatic hypertable partitioning (chunking data by time), columnar compression (90%+ storage reduction for historical OHLCV data), and time-series specific functions (`time_bucket`, `first`, `last`, `candlestick_agg`). It outperforms raw PostgreSQL for time-series queries by 10–100x.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **InfluxDB** | Purpose-built time-series but no SQL; separate query language (Flux); no join capability |
| **QuestDB** | Very fast but small ecosystem; limited managed offerings; hiring risk |
| **Apache Druid** | Excellent for analytics but complex operations; Java-based; separate team needed |
| **kdb+** | Industry-grade for HFT but extremely expensive licensing; complex query language (Q) |
| **MongoDB Time Series** | Less mature; weaker SQL compatibility; aggregations less powerful |
| **Raw PostgreSQL** | 10-100x slower for time-series queries; storage inefficient |

**Pros**
- PostgreSQL-native: same tools, same team, same ORM
- Automatic data tiering: hot (SSD) → warm (HDD) → cold (S3) based on age
- Columnar compression: OHLCV data compresses 10:1 or better
- `candlestick_agg()`: purpose-built OHLCV aggregation function
- Continuous aggregates: pre-computed materialized views for common timeframes
- Horizontal scaling via Timescale Multi-node (Hyperscale)
- Managed offering: Timescale Cloud (reduce ops burden)

**Cons**
- Hypertable requires careful chunk sizing configuration
- Multi-node Hyperscale adds operational complexity
- Timescale Cloud can be expensive at very high data volumes
- Continuous aggregates have refresh lag (acceptable for non-real-time queries)

**What Lives in TimescaleDB**
- Raw tick data (market ticks, indexed by symbol + timestamp)
- OHLCV candles (all timeframes, all symbols)
- Technical indicator snapshots
- Portfolio NAV history
- News articles with timestamps (for sentiment backtesting)

---

### 5C — ClickHouse

**Selected as: Analytics / OLAP Database**

**Why Selected**
ClickHouse is the fastest OLAP database for analytical queries over large datasets. A query that takes 30 seconds on PostgreSQL takes < 1 second on ClickHouse. For the Analytics & Reporting Service — which runs queries like "signal accuracy rate by strategy by sector over last 90 days across all users" — ClickHouse is the only viable solution at our data volumes.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Amazon Redshift** | Slower than ClickHouse for our query patterns; higher cost; vendor lock-in |
| **Google BigQuery** | Excellent but per-query cost model is unpredictable; latency for dashboards |
| **Apache Druid** | Complex to operate; Java; less SQL-friendly |
| **Snowflake** | Best managed DW option but very expensive; per-credit cost model |
| **DuckDB** | Excellent for embedded analytics but not a server database |

**Pros**
- Columnar storage: 100x+ faster than row-based DBs for analytical queries
- Vectorized query execution: SIMD instructions leverage modern CPU hardware
- Extremely high compression: 5-10x better than PostgreSQL for analytical data
- Linear horizontal scaling (sharding built-in)
- SQL-compatible with most BI tools
- Low operational cost: open-source, self-hosted on Kubernetes

**Cons**
- Not suitable for OLTP (no single-row updates/deletes efficiently)
- Limited JOIN performance vs columnar alternatives like Redshift
- Requires separate ETL pipeline to populate from Kafka
- Smaller talent pool than PostgreSQL

**What Lives in ClickHouse**
- Signal accuracy statistics (by strategy, sector, timeframe)
- Alert delivery analytics (success rate by channel by day)
- User engagement metrics (DAU, feature usage)
- Platform-wide aggregated KPIs
- Backtesting result aggregations
- Model prediction accuracy tracking over time

---

## 6. Cache

**Selected: Redis Cluster (7.x)**

**Why Selected**
Redis is the undisputed standard for in-memory caching and real-time data structures in production SaaS applications. For this platform, Redis serves five distinct roles simultaneously: API response cache, live price store, WebSocket Pub/Sub backbone, rate limit counter store, and alert deduplication state store. Cluster mode ensures no single point of failure and horizontal memory scaling.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Memcached** | Key-value only; no Pub/Sub; no sorted sets; no persistence; no clustering intelligence |
| **Hazelcast** | Distributed but complex; Java-heavy; overkill |
| **Apache Ignite** | Feature-rich but operationally complex; heavier than Redis |
| **DragonflyDB** | Redis-compatible, faster per-thread but immature for production; small community |
| **Valkey** | Redis fork by Linux Foundation; promising but too early |

**Pros**
- Sub-millisecond latency for all operations
- Rich data structures: Strings, Hashes, Lists, Sets, Sorted Sets, Streams, Pub/Sub
- Pub/Sub: powers WebSocket fan-out to users (live prices, signals, alerts)
- Atomic operations: rate limiting counters, deduplication logic
- Cluster mode: 6-node setup (3 primary + 3 replica) — zero downtime failover
- Persistence: RDB snapshots + AOF log for durability
- Redis Streams: lightweight Kafka alternative for simpler event flows
- AWS ElastiCache: managed, auto-failover, no ops burden
- Lua scripting: atomic multi-step operations

**Cons**
- In-memory: expensive per GB at large data volumes (mitigated by Redis-as-cache, not primary store)
- Cluster mode limits multi-key operations across slots
- Single-threaded command execution (mitigated by connection multiplexing and cluster)
- Not a replacement for a message broker (Kafka handles durability + replay)

**Redis Usage Map**

| Key Pattern | Data Type | TTL | Purpose |
|---|---|---|---|
| `price:{symbol_id}` | Hash | 5s | Live LTP, volume, OI |
| `indicator:{symbol_id}:{tf}` | Hash | 2m | Latest indicator snapshot |
| `signal:feed:{user_id}` | List | 1h | Recent signal feed (paginated) |
| `session:{token_hash}` | String | 15m | JWT session validation |
| `ratelimit:{user_id}:{window}` | String | 1m | API rate limit counter |
| `dedup:alert:{user_id}:{key}` | String | varies | Alert deduplication |
| `ws:sub:{user_id}` | Set | session | WebSocket subscriptions |
| `tenant:{id}:config` | Hash | 30m | Tenant configuration cache |

---

## 7. Task Queue

**Selected: Celery 5.x + Redis (broker) + Redis (result backend)**

**Why Selected**
Celery is the mature, production-proven distributed task queue for Python. Backtesting jobs are compute-intensive and must run asynchronously (a user should not wait for a 10-year backtest to complete). Notification retries, model retraining triggers, and report generation are all excellent Celery use cases. Redis as the broker provides simplicity and removes an additional infrastructure component (Kafka is already present but reserved for event streaming, not task queuing).

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Kafka (as task queue)** | Kafka is a log, not a task queue; no task result tracking; no priority queues; no retries |
| **RQ (Redis Queue)** | Simpler than Celery but less mature; limited monitoring; no canvas workflows |
| **Dramatiq** | Good alternative but smaller community; less documentation |
| **Huey** | Lightweight but lacks enterprise features |
| **Temporal** | Excellent workflow orchestration but very heavy for this use case; complex ops |
| **Airflow** | Good for batch pipelines (used in offline training) but not for real-time task queuing |

**Pros**
- Native Python: integrates directly with FastAPI services
- Priority queues: CRITICAL (alerts) > HIGH (backtesting) > NORMAL (reports)
- Canvas: complex task workflows (chain, group, chord) for multi-step backtests
- Flower: real-time Celery monitoring UI
- Retry with exponential backoff: built-in for notification failures
- Rate limiting per task type: prevents external API overload
- Worker auto-scaling: Kubernetes HPA scales Celery workers by queue depth

**Cons**
- Redis broker lacks Kafka's durability guarantees (acceptable for tasks, not for events)
- No built-in long-term task history (Flower is ephemeral; use PostgreSQL for audit)
- Complex workflows (backtesting with multiple strategies in parallel) require careful task design

**Task Queue Design**

| Queue | Priority | Workers | Example Tasks |
|---|---|---|---|
| `critical` | Highest | Always-on | SL alert triggers, circuit breaker notifications |
| `high` | High | Auto-scaled | Backtesting jobs, model inference batches |
| `normal` | Normal | Auto-scaled | Report generation, email digests |
| `low` | Low | Scheduled | Data cleanup, cold archive, daily analytics |

---

## 8. Scheduler

**Selected: Celery Beat (primary) + APScheduler (in-process, supplementary)**

**Why Selected**
Celery Beat is the distributed scheduler that integrates natively with Celery, enabling cron-like scheduled tasks in a distributed, Kubernetes-friendly way. A single Beat instance manages all scheduled jobs and dispatches them to Celery workers. APScheduler provides lightweight in-process scheduling for fast, periodic tasks that don't need a distributed queue (e.g., Redis cache warming every 60 seconds within a single service).

**Scheduled Jobs Catalogue**

| Job | Schedule | System | Purpose |
|---|---|---|---|
| Market data health check | Every 30s during market hours | APScheduler | Detect stale data feeds |
| Symbol master refresh | 08:30 IST daily | Celery Beat | Sync new listings, delistings |
| Pre-market data warmup | 09:00 IST daily | Celery Beat | Pre-compute last session indicators |
| Options chain refresh | Every 5m (09:15–15:30) | APScheduler | F&O data update |
| Portfolio P&L snapshot | Every 5m during market hours | Celery Beat | Save portfolio state |
| Model retraining trigger | 20:00 IST daily | Celery Beat | Nightly offline model training |
| Daily alert digest email | 18:00 IST daily | Celery Beat | EOD summary email to users |
| Backtesting result cleanup | 02:00 IST daily | Celery Beat | Archive old backtest reports |
| User data export | On-demand (DPDPA compliance) | Celery task | User data download request |
| Database index maintenance | 03:00 IST Sunday | Celery Beat | VACUUM, ANALYZE, REINDEX |
| Log archive to S3 | 01:00 IST daily | Celery Beat | Move logs from hot to cold storage |

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Apache Airflow** | Excellent for complex DAG pipelines (used for ML training DAGs) but too heavy as a simple scheduler; separate deployment |
| **Kubernetes CronJobs** | Good for infrastructure-level jobs but not for application-level scheduling with Python context |
| **cron (OS-level)** | Not distributed; single point of failure; no retry logic |

---

## 9. Storage

**Selected: AWS S3 (primary object storage)**

**Why Selected**
AWS S3 is the global standard for object storage. It provides 11 nines (99.999999999%) durability, virtually unlimited capacity, versioning, lifecycle policies, and deep integration with every AWS service (EKS, Lambda, Athena, SageMaker). For this platform, S3 stores ML model artifacts, backtest PDF reports, user data exports, log cold archives, and database backups.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Google Cloud Storage** | Excellent but switching cloud provider; adds complexity if primary cloud is AWS |
| **Azure Blob Storage** | Same concern — multi-cloud complexity |
| **MinIO** | S3-compatible self-hosted; good for on-premise but adds operational burden |
| **Wasabi** | Cheaper but smaller ecosystem; fewer integrations |

**Pros**
- 11 nines durability; effectively zero data loss
- S3 Intelligent-Tiering: automatic cost optimization (hot → warm → cold → Glacier)
- Versioning: every model artifact version preserved
- Presigned URLs: secure, time-limited direct download links (for backtest PDFs)
- IAM integration: fine-grained access control per service (least privilege)
- S3 Lifecycle Rules: automatically archive logs after 30 days, delete after 1 year
- Athena integration: SQL queries directly on S3 data (log analysis without Elasticsearch)
- Global replication: S3 Cross-Region Replication for disaster recovery

**Cons**
- Per-request costs can accumulate with high-frequency small object access
- Latency (~100ms) means not suitable for hot path data (use Redis for that)
- Data egress costs (design to minimize cross-region data transfer)

**S3 Bucket Structure**

| Bucket | Purpose | Lifecycle |
|---|---|---|
| `platform-ml-models` | Model artifacts (PyTorch weights, XGBoost files) | Versioned; no expiry |
| `platform-backtest-reports` | PDF backtest reports | 90-day retention |
| `platform-user-exports` | DPDPA user data exports | 7-day retention; presigned URL |
| `platform-logs-cold` | Elasticsearch cold archive | 1-year retention |
| `platform-db-backups` | PostgreSQL and TimescaleDB backups | 30-day retention |
| `platform-news-archive` | Historical news corpus for model training | Indefinite |

---

## 10. Authentication

**Selected: Keycloak (self-hosted)**

**Why Selected**
Keycloak is the leading open-source Identity and Access Management (IAM) solution. It provides OAuth 2.0, OpenID Connect, SAML 2.0, MFA (TOTP), RBAC, social login, and enterprise federation (LDAP/Active Directory) — all out of the box. For a multi-tenant SaaS platform, Keycloak's realm concept maps perfectly to tenants: each tenant is a Keycloak realm with its own users, roles, and configurations. Self-hosting eliminates per-MAU (Monthly Active User) pricing that Auth0 and Cognito charge at scale.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Auth0** | Excellent developer experience but $240/month per 1,000 MAUs → $12,000/month at 50k users. Prohibitively expensive. |
| **AWS Cognito** | Good AWS integration but complex configuration; limited customization; cold start on token validation |
| **Firebase Authentication** | Simple but limited; no multi-tenancy model; Google dependency |
| **Supabase Auth** | Good for MVPs; limited enterprise features |
| **Custom JWT implementation** | Security risk: rolling your own auth is the most common source of vulnerabilities |

**Pros**
- Zero per-user licensing cost (self-hosted)
- Multi-tenant: each tenant = one Keycloak realm
- OAuth 2.0 + OIDC: industry-standard; all clients (Next.js, React Native, FastAPI) support it
- MFA: TOTP, WebAuthn, SMS (via SPI extension)
- RBAC: Admin, Analyst, Trader, Viewer roles with fine-grained permissions
- Social Login: Google, GitHub, LinkedIn (for easy onboarding)
- LDAP/AD Federation: Enterprise tenant integration
- Brute force protection, credential rotation, session management
- Token validation: services validate JWT locally (no roundtrip to Keycloak per request)

**Cons**
- Operational overhead: self-hosted means managing Keycloak HA cluster
- Java-based: higher memory footprint than lightweight alternatives
- Configuration complexity: realms, clients, flows require expertise
- Upgrading Keycloak versions requires testing (breaking changes between major versions)

**Mitigation**: Deploy Keycloak on Kubernetes with 3 instances behind a load balancer. Use managed PostgreSQL as Keycloak's database (not embedded H2). Treat Keycloak as infrastructure, not an application service.

---

## 11. Monitoring

**Selected: Prometheus + Grafana + Alertmanager**

**Why Selected**
This is the de-facto standard observability stack for Kubernetes-based microservices. Prometheus scrapes metrics from all services (pull model), stores them in its time-series database, and evaluates alert rules. Grafana provides visualization. This stack has the largest library of pre-built dashboards (Grafana Dashboard repository has 3,000+ community dashboards) and integrates natively with every technology in this stack.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Datadog** | Excellent but $23/host/month; at 20+ Kubernetes nodes = $5,500+/month for monitoring alone |
| **New Relic** | Similar cost profile to Datadog; vendor lock-in |
| **CloudWatch** | AWS-native but poor for application-level metrics; limited visualization |
| **Dynatrace** | Best-in-class APM but very expensive; overkill for this phase |
| **VictoriaMetrics** | Prometheus-compatible and cheaper at scale; use as long-term storage replacement |

**Pros**
- Open-source: zero licensing cost
- Kubernetes-native: kube-prometheus-stack Helm chart deploys everything in minutes
- Pull model: services don't need to know about Prometheus; just expose `/metrics`
- 3,000+ pre-built Grafana dashboards (Kafka, PostgreSQL, Redis, Node.js all covered)
- Alertmanager: de-duplication, grouping, routing, silencing
- PromQL: powerful query language for metric analysis
- Horizontal scaling: Thanos or VictoriaMetrics for long-term storage

**Cons**
- No distributed tracing (handled by Jaeger/Tempo)
- No log aggregation (handled by ELK)
- Prometheus storage is local TSDB; long-term storage requires Thanos/VictoriaMetrics
- High-cardinality metrics can cause performance issues (needs careful label design)

**Monitoring Stack Components**

| Tool | Role |
|---|---|
| **Prometheus** | Metric collection and storage (30-day hot) |
| **Grafana** | Dashboards, visualization, alerting UI |
| **Alertmanager** | Alert routing → PagerDuty / Slack / Email |
| **VictoriaMetrics** | Long-term metric storage (1 year, cheaper than Thanos) |
| **kube-state-metrics** | Kubernetes resource metrics (pod, deployment, node) |
| **node-exporter** | OS-level metrics per Kubernetes node |
| **Grafana OnCall** | On-call scheduling and escalation |
| **PagerDuty** | Critical incident management and mobile alerts |

---

## 12. Logging

**Selected: ELK Stack — Elasticsearch + Logstash + Kibana + Filebeat**

**Why Selected**
The ELK Stack is the industry-standard centralized logging solution. Elasticsearch provides full-text search over billions of log lines in milliseconds. Kibana enables engineers to build log dashboards and investigate incidents. Filebeat (lightweight log shipper) runs as a DaemonSet on every Kubernetes node, collecting logs from all containers with zero application changes.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Grafana Loki** | Cheaper and simpler than ELK but no full-text search — can only filter by labels; limited for complex incident investigation |
| **Splunk** | Best-in-class but extremely expensive ($150+/GB/day ingest); enterprise licensing |
| **Datadog Logs** | Same cost concern as Datadog APM |
| **AWS CloudWatch Logs** | Functional but poor search UX; expensive at high volume; limited cross-service correlation |
| **OpenSearch (AWS)** | Elasticsearch fork; compatible but lags in features; use if Elastic licensing is a concern |

**Pros**
- Full-text search across all services' logs
- Kibana: powerful log visualization, dashboard building
- Index Lifecycle Management (ILM): automatic hot → warm → cold → delete
- APM integration: trace_id in logs links to Jaeger distributed traces
- SIEM capabilities: security event detection (Kibana Security) — needed for financial platforms
- Alerting from log patterns (error rate spike detection)
- Elastic Common Schema (ECS): standardized log format across all services

**Cons**
- Operationally complex: 3-node Elasticsearch cluster requires careful management
- Memory-intensive: Elasticsearch requires JVM heap sizing (typically 8-16GB per node)
- Elastic licensing: Elastic changed to SSPL license in 7.11; alternatives: OpenSearch (AWS fork, Apache 2.0) or Elasticsearch AGPL
- Cost at high log volume: need to tune retention policies and index sizing carefully

**Log Pipeline**
```
All Pods → stdout (structured JSON)
         → Filebeat DaemonSet (per node)
         → Logstash (enrich: PII scrub, add k8s metadata, trace_id)
         → Elasticsearch (index by service + date)
         → Kibana (search, dashboards)
         → S3 (cold archive via ILM after 30 days)
```

---

## 13. CI/CD Pipeline

**Selected: GitHub Actions (CI) + ArgoCD (CD)**

### GitHub Actions (Continuous Integration)

**Why Selected**
GitHub Actions is the natural CI choice when the codebase is hosted on GitHub. It provides deep integration (PRs, commits, releases trigger workflows automatically), a marketplace of 15,000+ pre-built actions, matrix builds (test across Python 3.11/3.12, Node.js 20/22), and cost-effective pricing (2,000 free minutes/month on free plan; affordable on team plans).

**Pipeline Stages**
1. **On every PR**: lint (ruff, black, eslint), type check (mypy, tsc), unit tests (pytest, jest)
2. **On merge to main**: integration tests, build Docker images, push to ECR, trigger ArgoCD sync
3. **On release tag**: full regression suite, security scan (Trivy), performance benchmarks, deploy to staging
4. **Nightly**: dependency vulnerability scan (Dependabot), ML model performance evaluation

### ArgoCD (Continuous Deployment — GitOps)

**Why Selected**
ArgoCD implements GitOps: Kubernetes manifests stored in Git are the single source of truth. ArgoCD continuously reconciles the cluster state with the Git state. If a deployment drifts from the Git definition (e.g., someone manually `kubectl apply`s), ArgoCD detects and corrects it. This provides full deployment audit trail and one-click rollback to any previous Git commit.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Jenkins** | Mature but complex; Groovy DSL; heavier operational burden |
| **GitLab CI/CD** | Excellent but requires GitLab hosting or self-hosting; GitHub is assumed |
| **CircleCI** | Good but higher cost at scale; moving away from Docker-native execution |
| **Flux CD** | GitOps alternative to ArgoCD; similar quality; ArgoCD has better UI |
| **Helm + manual kubectl** | No GitOps discipline; no automatic drift detection; rollback is manual |

**Pros of Full Pipeline**
- ArgoCD: automatic rollback on failed deployments (health check failure → revert)
- GitHub Actions: native PR integration, status checks block merges on test failure
- Full audit trail: every deployment is a Git commit with author, timestamp, diff
- Blue-green deployment support (zero-downtime deploys via ArgoCD ApplicationSet)
- Secrets never in Git (Sealed Secrets or External Secrets Operator)
- Multi-environment: `dev` / `staging` / `production` branches mapped to ArgoCD Applications

---

## 14. Cloud Provider

**Selected: AWS (Amazon Web Services)**

**Why Selected**
AWS is the market leader with the broadest service catalog. For this platform, the managed service ecosystem directly reduces engineering burden: MSK (managed Kafka), RDS (managed PostgreSQL), ElastiCache (managed Redis), EKS (managed Kubernetes), ECR (container registry), and S3 (object storage) are all AWS-native. India region (`ap-south-1` Mumbai) provides low latency to NSE/BSE data centers. AWS also meets SEBI data localization requirements (data must reside in India).

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **GCP (Google Cloud)** | Excellent ML tooling (Vertex AI) but smaller managed service ecosystem; fewer devs with GCP expertise in India |
| **Azure** | Strong enterprise integration but higher cost; less popular in Indian startup ecosystem |
| **Hetzner (bare metal)** | 5x cost savings but: no managed Kafka, no managed Kubernetes, no managed Redis; massive ops burden |
| **DigitalOcean / Linode** | Good for early stage but limited managed services at scale |

**Key AWS Services in Use**

| AWS Service | Purpose |
|---|---|
| **EKS** | Managed Kubernetes (all microservices) |
| **MSK (Managed Streaming for Kafka)** | Managed Apache Kafka cluster |
| **RDS PostgreSQL (Multi-AZ)** | Managed primary database |
| **ElastiCache (Redis Cluster)** | Managed Redis |
| **S3** | Object storage |
| **ECR** | Private container registry |
| **CloudFront** | CDN for static assets + API edge caching |
| **Route 53** | DNS management |
| **ACM** | SSL/TLS certificate management |
| **Secrets Manager** | Encrypted secrets storage |
| **SES** | Transactional email (backup to SendGrid) |
| **WAF** | Web Application Firewall |
| **Shield Standard** | DDoS protection (included) |
| **CloudTrail** | API audit logging (compliance) |
| **Cost Explorer** | Cloud cost monitoring |

**Pros**
- Mumbai region (`ap-south-1`): low latency to NSE/BSE (<5ms to financial district)
- SEBI data localization: data stays in India
- Widest managed service portfolio
- Largest talent pool in India
- Comprehensive security and compliance certifications (ISO 27001, SOC 2)

**Cons**
- Higher cost than bare metal at equivalent compute
- Vendor lock-in risk (mitigated by Kubernetes abstraction — services can migrate to GCP/Azure)
- EKS is more expensive than self-managed Kubernetes (mitigated by reduced ops burden)

---

## 15. Containerization

**Selected: Docker (container runtime) + Kubernetes (orchestration) + EKS (managed K8s)**

**Why Selected**
Docker + Kubernetes is the undisputed standard for containerized microservices at scale. Kubernetes provides horizontal pod autoscaling (HPA), automatic self-healing, rolling deployments, resource quotas, network policies, and service discovery — all non-negotiable for a 5,000+ concurrent user platform. EKS removes the control plane management burden (version upgrades, etcd backups, control plane HA handled by AWS).

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Docker Swarm** | Simpler than Kubernetes but dead ecosystem; no HPA; no advanced scheduling |
| **AWS ECS / Fargate** | Easier to start but less portable; vendor lock-in; weaker multi-service networking |
| **Nomad (HashiCorp)** | Excellent scheduler but smaller ecosystem; Kubernetes is the industry standard |
| **Bare VMs (no containers)** | No portability; slow scaling; environment drift; deployment complexity |

**Key Kubernetes Features in Use**

| Feature | Use Case |
|---|---|
| **HPA (Horizontal Pod Autoscaler)** | Scale Strategy Engine from 5 to 50 pods under load |
| **VPA (Vertical Pod Autoscaler)** | Right-size memory/CPU requests for stable services |
| **Node Auto-provisioning (Karpenter)** | Automatically provision EC2 instances when cluster needs capacity |
| **Network Policies** | Restrict traffic: web pods cannot directly talk to database pods |
| **Pod Disruption Budgets** | Guarantee minimum replicas during rolling updates |
| **Resource Quotas** | Prevent one service from consuming entire cluster resources |
| **Namespaces** | Separate `production`, `staging`, `monitoring` workloads |
| **RBAC (K8s)** | Engineers have read access; only ArgoCD has deploy access |
| **Helm** | Package manager for Kubernetes manifests |
| **Karpenter** | EC2 instance provisioning (faster than Cluster Autoscaler) |

**GPU Node Pool**
- Separate node pool with NVIDIA GPU instances (p3.2xlarge or g4dn.xlarge)
- Only scheduled during nightly model training jobs (8 hours/night)
- Automatically terminated after training (cost optimization)
- GPU nodes have `NoSchedule` taint — only ML training pods tolerate them

---

## 16. Reverse Proxy / API Gateway

**Selected: Kong Gateway (API Gateway) + Nginx (internal reverse proxy)**

### Kong Gateway

**Why Selected**
Kong Gateway is an enterprise-grade API gateway built on top of Nginx+OpenResty. Unlike raw Nginx, Kong provides a plugin ecosystem for authentication (JWT validation), rate limiting (per user, per tier, per route), request transformation, logging, circuit breaking, and WebSocket proxying — all without application code changes. Kong's declarative configuration (deck) integrates with GitOps.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **AWS API Gateway** | Vendor lock-in; limited WebSocket support; per-request pricing at scale is expensive |
| **Traefik** | Excellent for service mesh but weaker plugin ecosystem than Kong for our use cases |
| **NGINX Ingress only** | Lacks built-in rate limiting per user, JWT validation, and analytics |
| **Envoy (Istio)** | Used as service mesh inside cluster; too complex as external-facing gateway |
| **Apigee (Google)** | Excellent but expensive; Google-tied |

**Kong Plugins in Use**

| Plugin | Purpose |
|---|---|
| **JWT** | Validate JWT tokens on every request |
| **Rate Limiting Advanced** | Sliding window rate limits per user tier |
| **Prometheus** | Export Kong metrics to Prometheus |
| **Correlation ID** | Inject trace_id into every request |
| **Request Transformer** | Add `X-Tenant-ID` header from JWT claims |
| **Response Transformer** | Add security headers (HSTS, CSP, X-Frame-Options) |
| **Bot Detection** | Block known bot user agents |
| **IP Restriction** | Allowlist/blocklist by IP (admin endpoints) |
| **WebSocket** | WebSocket upgrade and proxying |

### Nginx (Internal)
- Handles static file serving for the Next.js frontend in production
- Acts as a sidecar proxy for services that need local request buffering
- Terminates TLS for internal service-to-service communication (before Istio mTLS)

---

## 17. Message Broker

**Selected: Apache Kafka (via AWS MSK)**

**Why Selected**
Kafka is the definitive choice for a real-time financial data platform. It handles 10 million+ messages per second, provides ordered, partitioned, replayable event streams, and enables the decoupled event-driven architecture that this platform requires. All 13 event topics (from `raw.market.ticks` to `notifications.delivery_receipts`) are Kafka topics. AWS MSK eliminates broker management — no Kafka cluster version upgrades, no ZooKeeper management, no broker disk management.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **RabbitMQ** | Good task queue but not an event stream; no replay; limited to queuing semantics |
| **AWS SQS + SNS** | Good for AWS-native fan-out but no replay; 14-day max retention; no ordering guarantees at scale |
| **Pulsar (Apache)** | Excellent Kafka alternative with better multi-tenancy; but smaller ecosystem; Kafka has 10x more client libraries |
| **NATS / NATS JetStream** | Extremely fast and lightweight; good for simpler architectures; limited for complex event sourcing needs |
| **Redis Streams** | Good for simple event flows but not suitable for high-throughput financial tick streaming |

**Pros**
- Infinite replay: consumers can replay historical events from any offset (critical for new service onboarding and bug replay)
- High throughput: 100MB/s+ per broker with commodity hardware
- Ordering guaranteed within a partition (symbol-partitioned = per-symbol ordering guaranteed)
- Consumer groups: multiple services consume same topic independently at their own pace
- Retention: ticks stored for 7 days; signals for 14 days (configurable)
- AWS MSK: managed brokers, automatic storage scaling, encryption in transit/at rest
- Schema Registry (AWS Glue or Confluent): Avro schema enforcement on every topic

**Cons**
- Operationally complex without managed service (MSK solves this)
- Message ordering only within partition (multi-partition topics require careful partition key design)
- Not a task queue (Celery handles that separately)
- Minimum latency ~5ms (Kafka is not suitable for microsecond latency requirements — not our use case)

**Kafka Cluster Configuration**

| Setting | Value | Rationale |
|---|---|---|
| Brokers | 3 | Fault tolerance: survive 1 broker failure |
| Replication Factor | 3 | All topics replicated to all brokers |
| Min ISR | 2 | Write succeeds only if 2/3 brokers acknowledge |
| Partition Strategy | By symbol_id hash | Ensures per-symbol ordering |
| Retention | 7 days (market data), 14 days (signals) | Balance between replay ability and storage cost |
| Compression | LZ4 | Best ratio for financial data; fast compression/decompression |

---

## 18. Notification Services

**Multi-Channel Notification Stack**

### 18A — Telegram

**Selected: Telegram Bot API (direct)**

**Why Selected**
Telegram Bot API is free, well-documented, and has no per-message cost. Telegram is widely adopted by Indian retail investors and traders. Bots support rich text (Markdown), buttons, and inline keyboards — enabling interactive alerts ("Tap to view full signal analysis").

**Key Considerations**
- Rate limit: 30 messages/second globally per bot; 20 messages/minute per chat — must implement token bucket
- Message formatting: MarkdownV2 for rich formatting
- Webhook vs polling: Webhook mode for production (lower latency, no polling overhead)
- Bot token stored in AWS Secrets Manager

---

### 18B — WhatsApp

**Selected: Meta WhatsApp Cloud API (direct) with BSP fallback**

**Why Selected**
WhatsApp has 500+ million Indian users — the highest penetration of any messaging platform in India. The Meta WhatsApp Cloud API (direct) is free for the first 1,000 conversations/month, then per-conversation pricing. For scale, a BSP (Business Solution Provider) like **Interakt** or **Gupshup** provides managed WhatsApp Business infrastructure, higher throughput, and template management — recommended once volume justifies it.

**Key Considerations**
- Meta Business Verification required before API access
- Message templates must be pre-approved by Meta (24–48 hour review)
- Two conversation types: user-initiated (free) vs business-initiated (₹0.50–₹1.00 per conversation India pricing)
- Phone number must be WhatsApp Business registered (cannot also be used on WhatsApp personal)

**BSP vs Direct**

| | Meta Cloud API (Direct) | BSP (Interakt/Gupshup) |
|---|---|---|
| Cost | Cheaper at low volume | Higher per-message but managed |
| Setup | Complex; self-managed | Managed; faster to market |
| Throughput | Limited | Higher |
| Template management | Manual | Dashboard UI |
| Recommendation | Phase 1 (< 10k users) | Phase 2 (10k+ users) |

---

### 18C — Email

**Selected: SendGrid (primary) + AWS SES (backup)**

**Why Selected**
SendGrid is the industry standard for transactional email. It provides delivery analytics, bounce management, unsubscribe handling, and template management out of the box. AWS SES is 10x cheaper ($0.10 per 1,000 emails vs SendGrid's $0.0015/email at scale) and serves as the automatic failover when SendGrid rate limits are approached.

**Email Types**
- Transactional: OTP, password reset, account verification (SendGrid)
- Alert digests: Daily EOD market summary (SendGrid)
- Reports: Backtest PDF delivery, tax P&L reports (SendGrid + S3 presigned attachment)
- Marketing: Onboarding drip campaigns (SendGrid Marketing Campaigns)

---

### 18D — Push Notifications

**Selected: Firebase Cloud Messaging (FCM) for Android + Apple Push Notification Service (APNs) for iOS**

**Why Selected**
FCM and APNs are the only official channels for push notifications on Android and iOS respectively. FCM additionally supports web push (for Chrome/Edge browser notifications on the web dashboard). Both are free for standard use.

**React Native Integration**: `@react-native-firebase/messaging` for FCM + APNs unified API.

**Notification Types**
- `data-only` messages: Silent background updates (portfolio P&L refresh)
- `notification` messages: Visible alert banners (GRADE-A signal alert)
- Notification channels (Android): Separate channels for Critical vs Normal alerts (user can mute one)

---

## 19. Search Engine

**Selected: Elasticsearch (as part of ELK Stack)**

**Why Selected**
Elasticsearch already exists in the stack for log aggregation. Repurposing it for application-level search (symbol search, news search, user search) adds zero additional infrastructure. Elasticsearch provides full-text search with relevance scoring, fuzzy matching (handles typos in stock names), and instant autocomplete — all needed for the "search any symbol" functionality in the dashboard.

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Algolia** | Excellent developer experience but $1/1000 searches; at high usage, expensive; vendor lock-in |
| **Typesense** | Fast and open-source; good alternative but adds another service when ES is already present |
| **PostgreSQL Full-Text Search** | Adequate for basic search but slower; no fuzzy matching; no relevance tuning |
| **Meilisearch** | Fast; modern API; but another service to maintain |

**Search Indices in Use**

| Index | Data | Use Case |
|---|---|---|
| `symbols` | Symbol name, ticker, sector, market cap, exchange | "Search RELIANCE" → instant autocomplete |
| `news` | Headline, body excerpt, tags | Full-text news search by keyword or symbol |
| `signals` | Signal description, strategy name | Signal history search |
| `logs-*` | All platform logs | Engineering incident investigation |

---

## 20. Secrets Management

**Selected: AWS Secrets Manager (primary) + HashiCorp Vault (advanced workflows)**

### AWS Secrets Manager

**Why Selected**
AWS Secrets Manager provides encrypted secret storage with automatic rotation for RDS database credentials, API keys, and service passwords. Native EKS integration via External Secrets Operator means secrets are automatically synchronized to Kubernetes Secrets without ever being stored in Git or environment variables.

**What It Stores**
- Database credentials (PostgreSQL, TimescaleDB, Redis passwords)
- External API keys (Telegram Bot token, WhatsApp API key, SendGrid API key, data vendor API keys)
- JWT signing keys (RSA private keys)
- Broker API credentials (encrypted, per user — future)
- Internal service-to-service API keys

**Key Features**
- Automatic rotation: RDS credentials rotated every 30 days automatically
- IAM-based access: each service's IAM role only has permission to read its specific secrets
- Audit logging: every secret access logged in CloudTrail
- Versioning: previous secret versions retained for rollback

### HashiCorp Vault (Supplementary)

Used for advanced secrets workflows not covered by AWS Secrets Manager:
- **Dynamic secrets**: Vault generates short-lived, just-in-time database credentials for CI/CD pipelines (no long-lived credentials in pipelines)
- **PKI (Certificate Authority)**: Issues internal TLS certificates for mTLS between services (Istio integration)
- **Transit encryption**: Encrypt/decrypt broker API keys at the application layer (double encryption — Vault + Secrets Manager)

**Alternatives Considered**

| Alternative | Why Rejected |
|---|---|
| **Environment Variables** | Plain text in Kubernetes manifests or shell; leaked in logs; no rotation; NEVER acceptable |
| **Kubernetes Secrets (base64)** | Not encrypted at rest by default; stored in etcd; no rotation; only use as the delivery mechanism |
| **Doppler** | Good developer experience but vendor lock-in; per-secret pricing |
| **Parameter Store (AWS SSM)** | Good alternative but less feature-rich than Secrets Manager for rotation |

---

## Technology Stack Version 1.0 — Consolidated Reference

```
╔══════════════════════════════════════════════════════════════════════════════╗
║          TECHNOLOGY STACK VERSION 1.0 — CONSOLIDATED REFERENCE              ║
╚══════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│  #   CATEGORY              SELECTED TECHNOLOGY                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  1   Frontend — Web        Next.js 14 (React, TypeScript, App Router)        │
│      Frontend — Mobile     React Native + Expo                               │
│      Frontend — Charts     TradingView Lightweight Charts                    │
│      Frontend — State      Zustand + TanStack Query                          │
│      Frontend — UI         shadcn/ui + Radix UI + Framer Motion              │
├─────────────────────────────────────────────────────────────────────────────┤
│  2   Backend — Primary     Python 3.12 + FastAPI (ASGI)                     │
│      Backend — Real-Time   Node.js 20 + NestJS + Socket.io                  │
│      Backend — Validation  Pydantic v2                                       │
│      Backend — ORM         SQLAlchemy 2.x + Alembic                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  3   AI Framework          PyTorch 2.x (LSTM, Transformer, FinBERT)         │
│      ML — Tabular          XGBoost (Breakout prediction, scoring)            │
│      ML — Classical        scikit-learn (preprocessing, evaluation)          │
│      ML — Operations       MLflow (experiment tracking, model registry)      │
│      Feature Store         Feast (online: Redis, offline: S3/Parquet)       │
├─────────────────────────────────────────────────────────────────────────────┤
│  4   Python Libraries      pandas, NumPy, TA-Lib, pandas-ta, SHAP           │
│                            Transformers (HuggingFace), Celery, httpx         │
│                            confluent-kafka, redis-py, scipy, pytest          │
├─────────────────────────────────────────────────────────────────────────────┤
│  5   Database — OLTP       PostgreSQL 16 (AWS RDS Multi-AZ)                 │
│      Database — TimeSeries TimescaleDB (Timescale Cloud or self-hosted)      │
│      Database — OLAP       ClickHouse (self-hosted on Kubernetes)            │
├─────────────────────────────────────────────────────────────────────────────┤
│  6   Cache                 Redis 7.x Cluster (AWS ElastiCache)               │
├─────────────────────────────────────────────────────────────────────────────┤
│  7   Task Queue            Celery 5.x + Redis broker + Redis result backend  │
├─────────────────────────────────────────────────────────────────────────────┤
│  8   Scheduler             Celery Beat (distributed) + APScheduler           │
├─────────────────────────────────────────────────────────────────────────────┤
│  9   Object Storage        AWS S3 (with Intelligent-Tiering lifecycle)       │
├─────────────────────────────────────────────────────────────────────────────┤
│  10  Authentication        Keycloak (self-hosted, HA cluster)                │
│                            OAuth 2.0 / OIDC / JWT (RS256)                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  11  Monitoring            Prometheus + Grafana + Alertmanager               │
│                            VictoriaMetrics (long-term storage)               │
│                            Grafana OnCall + PagerDuty (incident management)  │
├─────────────────────────────────────────────────────────────────────────────┤
│  12  Logging               ELK Stack (Elasticsearch + Logstash + Kibana)    │
│                            Filebeat DaemonSet + S3 cold archive              │
│                            OpenTelemetry + Jaeger (distributed tracing)      │
├─────────────────────────────────────────────────────────────────────────────┤
│  13  CI/CD                 GitHub Actions (CI) + ArgoCD (GitOps CD)         │
│                            Trivy (container security scanning)               │
│                            External Secrets Operator + Sealed Secrets        │
├─────────────────────────────────────────────────────────────────────────────┤
│  14  Cloud Provider        AWS (ap-south-1 Mumbai primary region)            │
│                            ap-south-2 Hyderabad (DR / failover)              │
├─────────────────────────────────────────────────────────────────────────────┤
│  15  Containerization      Docker (runtime) + Kubernetes EKS                │
│                            Helm (package management) + Karpenter (autoscale) │
│                            Istio (service mesh, mTLS) + Kiali (mesh UI)      │
├─────────────────────────────────────────────────────────────────────────────┤
│  16  Reverse Proxy         Kong Gateway (external API gateway)               │
│                            Nginx (internal static serving + ingress)         │
├─────────────────────────────────────────────────────────────────────────────┤
│  17  Message Broker        Apache Kafka (AWS MSK — 3 broker cluster)        │
│                            Confluent Schema Registry (Avro enforcement)      │
│                            AKHQ / Redpanda Console (Kafka management UI)     │
├─────────────────────────────────────────────────────────────────────────────┤
│  18  Notifications         Telegram Bot API (direct)                         │
│                            Meta WhatsApp Cloud API + BSP (Interakt Phase 2) │
│                            SendGrid (primary email) + AWS SES (backup)       │
│                            FCM (Android push) + APNs (iOS push)             │
├─────────────────────────────────────────────────────────────────────────────┤
│  19  Search Engine         Elasticsearch 8.x (reused from ELK stack)        │
├─────────────────────────────────────────────────────────────────────────────┤
│  20  Secrets Management    AWS Secrets Manager (primary)                     │
│                            HashiCorp Vault (dynamic secrets, PKI, transit)   │
│                            External Secrets Operator (K8s → Secrets Manager) │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Estimated Monthly Infrastructure Cost (AWS ap-south-1)

*Estimate at 5,000 concurrent users / 50,000 registered users. Actual costs depend on usage patterns.*

| Component | Service | Est. Monthly Cost (USD) |
|---|---|---|
| Kubernetes Control Plane | EKS | $73 |
| Worker Nodes (CPU) | EC2 (6x c6i.2xlarge) | $600 |
| GPU Node (training, nightly) | EC2 g4dn.xlarge × 8h/day | $120 |
| Database (primary) | RDS PostgreSQL Multi-AZ db.r6g.large | $240 |
| Time-Series DB | Timescale Cloud (200GB) | $350 |
| Analytics DB | ClickHouse (self-hosted on workers) | $0 (included) |
| Cache | ElastiCache Redis r6g.large cluster | $280 |
| Message Broker | MSK (3x kafka.m5.large) | $600 |
| Object Storage | S3 (2TB + requests) | $80 |
| CDN | CloudFront | $50 |
| Elasticsearch | Self-hosted on workers (3-node) | $0 (included) |
| Keycloak | Self-hosted on workers | $0 (included) |
| Data Transfer | Egress + inter-AZ | $100 |
| Email | SendGrid Essentials | $20 |
| Push | FCM/APNs | $0 (free) |
| Telegram | Bot API | $0 (free) |
| WhatsApp | Meta Cloud API (est. 10k msg/month) | $50 |
| DNS + SSL | Route 53 + ACM | $5 |
| Monitoring | Prometheus + Grafana (self-hosted) | $0 (included) |
| PagerDuty | Team plan | $30 |
| **Total Estimate** | | **~$2,600 / month** |

> *This is a Phase 1 estimate. At 50,000 active users with heavy concurrent use, expect $6,000–$10,000/month. Revenue from SaaS subscriptions should comfortably cover this at commercial pricing.*

---

## Technology Decisions Requiring Confirmation

> [!IMPORTANT]
> **D1 — TimescaleDB Hosting**: Self-hosted on Kubernetes (lower cost, higher ops) vs Timescale Cloud (higher cost, managed). Decision impacts Phase 3 database design and operational budget.

> [!IMPORTANT]
> **D2 — Keycloak vs SaaS Auth**: Confirmed self-hosted Keycloak? If team is small (< 5 engineers), Auth0's developer experience advantage may outweigh cost — until 10,000+ MAUs where cost becomes prohibitive. Need clarity on initial team size.

> [!NOTE]
> **D3 — WhatsApp BSP vs Direct**: Start with Meta Cloud API direct (Phase 1) and evaluate BSP migration at 10,000 users? Or go BSP-first for simpler template management?

> [!NOTE]
> **D4 — ClickHouse Hosting**: ClickHouse Cloud (managed, ~$400/month) vs self-hosted on existing Kubernetes workers (free but ops overhead). Same question as TimescaleDB.

---

*Phase 2C — Technology Stack Version 1.0*
*Status: Pending User Approval*
*Next: Phase 3 — Database Design (awaiting approval)*
