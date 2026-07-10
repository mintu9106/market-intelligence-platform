# Architecture Version 1.0 — AI-Powered Indian Stock Market Intelligence Platform

> **Phase 2: Enterprise System Architecture**
> Status: **Pending User Approval**
> Authored by: Chief Enterprise Architect
> Date: 2026-07-10

---

## Executive Summary

This document defines the complete enterprise architecture for an AI-Powered Indian Stock Market Intelligence Platform — a production-grade, multi-tenant SaaS system designed to serve 50,000+ registered users with 5,000+ concurrent sessions, delivering real-time market intelligence, explainable AI predictions, portfolio insights, smart alerts, paper trading, and backtesting capabilities.

The architecture is built around four foundational principles:

| Principle | Decision |
|---|---|
| **Scalability** | Horizontally scalable microservices, event-driven |
| **Reliability** | No single point of failure, fault-tolerant by design |
| **Explainability** | Every AI signal carries a human-readable rationale |
| **Extensibility** | Broker integrations and ML upgrades without downtime |

---

## Architectural Style

| Concern | Pattern Chosen | Rationale |
|---|---|---|
| Service decomposition | **Microservices** | Independent deploy, scale, and fault isolation per domain |
| Data flow | **Event-Driven Architecture (EDA)** | Decouples producers from consumers; enables real-time pipelines |
| Read/Write separation | **CQRS** | Read-heavy market queries separated from write-heavy signal generation |
| Audit & history | **Event Sourcing** | Full replay of market events; regulatory compliance |
| Multi-tenancy | **Shared infrastructure, isolated data** | Cost-efficient at scale; tenant-aware row-level security |
| AI serving | **Online + Offline dual pipeline** | Real-time inference + batch model training |

---

## Module Catalogue

### Module 1 — API Gateway & Edge Layer

**Purpose**
The single, unified entry point for all client traffic (web, mobile, external API consumers). Acts as the platform's front door.

**Responsibilities**
- TLS termination and SSL enforcement
- JWT/OAuth 2.0 token validation
- Rate limiting per tenant and per user tier (Free: 100 req/min, Pro: 1,000 req/min, Enterprise: unlimited)
- Request routing to downstream microservices
- Request/response transformation
- WebSocket upgrade for real-time feeds
- API versioning (`/v1`, `/v2`)
- DDoS protection and WAF (Web Application Firewall)
- Tenant identification from JWT claims

**Inputs**
- Raw HTTP/HTTPS/WebSocket requests from Web Dashboard, Mobile App, and third-party API clients

**Outputs**
- Authenticated, rate-limited, tenant-tagged requests routed to appropriate microservices

**Dependencies**
- Auth Service (for token introspection)
- Redis (for rate limit counters)

**Why It Exists**
Without a gateway, every microservice must individually handle auth, rate limiting, and routing. This creates duplication and security gaps. The gateway centralizes these cross-cutting concerns.

---

### Module 2 — Identity & Authentication Service

**Purpose**
Manages all aspects of user identity, authentication, and access control across all tenants.

**Responsibilities**
- User registration, login, password management
- OAuth 2.0 / OpenID Connect flows
- JWT issuance and refresh token rotation
- Multi-Factor Authentication (MFA/TOTP)
- Role-Based Access Control (RBAC): Admin, Analyst, Trader, Viewer
- Tenant-scoped permissions
- Session management and device tracking
- API key management for programmatic access

**Inputs**
- Login credentials, OAuth tokens, API keys

**Outputs**
- Signed JWT tokens with claims: `user_id`, `tenant_id`, `roles`, `subscription_tier`

**Dependencies**
- PostgreSQL (user store)
- Redis (session and token blacklist cache)
- Email Service (OTP delivery)

**Why It Exists**
Authentication is a security-critical, non-negotiable cross-cutting concern. It must be its own isolated service so security patches are deployed independently without touching business logic.

---

### Module 3 — Tenant & Subscription Management Service

**Purpose**
Manages the SaaS multi-tenancy model — onboarding organizations, managing their subscription plans, and enforcing feature access.

**Responsibilities**
- Tenant onboarding and provisioning
- Subscription plan management (Free, Pro, Enterprise)
- Feature flag enforcement per subscription tier
- Billing integration (Razorpay/Stripe)
- Usage metering (API calls, alert counts, watchlists)
- Tenant configuration (branding, custom domains — Enterprise)
- Tenant suspension and offboarding

**Inputs**
- Registration requests, payment webhooks, admin commands

**Outputs**
- Tenant context objects consumed by all other services
- Feature access decisions (allowed/denied)

**Dependencies**
- PostgreSQL (tenant data)
- Payment Gateway (Razorpay for INR, Stripe for global)
- Auth Service

**Why It Exists**
Tenancy rules must not bleed into domain services. A dedicated module keeps billing logic, feature gating, and tenant data isolated — preventing a billing bug from corrupting market data logic.

---

### Module 4 — Market Data Ingestion Service

**Purpose**
The platform's data nervous system. Connects to all external market data sources and ingests raw, real-time, and end-of-day data into the platform's internal event bus.

**Responsibilities**
- Connect to NSE/BSE real-time feeds (WebSocket or FIX protocol)
- Integration with data vendors: Zerodha Kite, Angel One SmartAPI, Upstox, NSE official data APIs, Dhan
- Ingest: OHLCV tick data, order book (Level 2), FII/DII data, derivatives data (F&O), index data (NIFTY 50, BANK NIFTY, etc.)
- Corporate action feeds (dividends, splits, bonus, rights)
- News feeds (Reuters, Bloomberg India, Economic Times API)
- Fundamental data feeds (quarterly results, shareholding pattern)
- Symbol master synchronization (new listings, delistings)
- Circuit breaker detection (upper/lower circuit)
- Data normalization — all sources emit in a unified canonical schema
- Dead-letter queue for failed ingestions

**Inputs**
- Raw tick streams from NSE/BSE/vendor WebSockets and REST APIs
- News RSS/API feeds

**Outputs**
- Normalized `MarketTickEvent` published to Kafka topic `raw.market.ticks`
- Normalized `NewsEvent` published to Kafka topic `raw.news`
- Normalized `FundamentalEvent` published to Kafka topic `raw.fundamentals`

**Dependencies**
- Apache Kafka (event bus)
- TimescaleDB (raw tick persistence)
- External: NSE/BSE data APIs, vendor SDKs

**Why It Exists**
Multiple data sources with different formats, protocols, and reliability profiles must be abstracted away from all downstream consumers. If a data vendor changes their API, only this service changes — not the entire platform.

---

### Module 5 — Market Data Processing Service

**Purpose**
Transforms raw tick data into enriched, analytics-ready market data: OHLCV candles, technical indicators, derived metrics, and market breadth signals.

**Responsibilities**
- Tick aggregation into OHLCV candles (1m, 3m, 5m, 15m, 30m, 1H, 4H, 1D, 1W)
- Real-time technical indicator computation:
  - Trend: EMA (9, 21, 50, 200), SMA, VWAP, Supertrend, Ichimoku
  - Momentum: RSI, MACD, Stochastic, Williams %R, CCI, MFI
  - Volatility: Bollinger Bands, ATR, Keltner Channels, Historical Volatility
  - Volume: OBV, VWAP, Volume Profile, CMF
  - Support/Resistance detection (pivot points, swing highs/lows)
- Market breadth computation (advance/decline ratio, new highs/lows)
- Derivatives metrics: PCR (Put-Call Ratio), OI (Open Interest) analysis, IV (Implied Volatility)
- Multi-timeframe alignment signals
- Gap detection (opening gaps)
- Pattern detection: pre-computed candlestick patterns
- Sector rotation metrics
- Correlation matrix (rolling 30-day)

**Inputs**
- `MarketTickEvent` from Kafka topic `raw.market.ticks`

**Outputs**
- `EnrichedCandleEvent` to Kafka topic `processed.candles`
- `IndicatorSnapshotEvent` to Kafka topic `processed.indicators`
- `MarketBreadthEvent` to Kafka topic `processed.breadth`
- Persisted enriched data → TimescaleDB

**Dependencies**
- Kafka (input and output)
- TimescaleDB (persistence of processed data)
- Redis (rolling window state for streaming indicators)

**Why It Exists**
Indicator computation is CPU-intensive and shared across multiple consumers (AI engine, signal engine, dashboard). Computing indicators once and sharing via events prevents redundant computation across 5,000+ concurrent users.

---

### Module 6 — Strategy Engine

**Purpose**
Evaluates configurable trading strategies against real-time processed market data and generates raw trade signals.

**Responsibilities**
- Built-in strategy library:
  - Trend Following (EMA crossover, Supertrend)
  - Momentum (RSI divergence, MACD crossover)
  - Breakout (volume-confirmed breakouts)
  - Mean Reversion (Bollinger Band squeeze)
  - Options strategies (straddle, strangle entry conditions)
  - Swing trading setups
  - Intraday scalping setups
- User-configurable strategy parameters (strategy customization per user/tenant)
- Multi-timeframe strategy confirmation (e.g., signal on 15m confirmed by 1H)
- Strategy combination (logical AND/OR of multiple strategies)
- Scan engine: evaluate strategies across entire NSE universe (~2,000 symbols) in real-time
- Signal scoring: confidence percentage per signal
- Signal deduplication (prevent repeated signals on same setup)

**Inputs**
- `EnrichedCandleEvent` and `IndicatorSnapshotEvent` from Kafka

**Outputs**
- `RawSignalEvent` to Kafka topic `signals.raw`
  - Contains: symbol, timeframe, strategy name, direction (BUY/SELL), confidence score, trigger price, SL, target

**Dependencies**
- Kafka (input/output)
- Redis (state for multi-bar pattern tracking)
- PostgreSQL (strategy configurations per user/tenant)

**Why It Exists**
Strategies are the core business logic of the platform. Isolating them in a dedicated service allows adding new strategies, A/B testing strategies, and user customization — without touching the AI engine or alert system.

---

### Module 7 — AI / ML Engine

**Purpose**
The intelligence core of the platform. Provides machine learning-based predictions, pattern recognition, sentiment analysis, and explainable AI (XAI) outputs layered on top of rule-based signals.

**Sub-Components**

#### 7A — Feature Engineering Pipeline
- Builds the ML feature vector for each symbol from enriched indicators, fundamentals, sentiment, and market breadth
- Features: 150+ engineered features per symbol
- Runs as a Kafka Streams job

#### 7B — Prediction Models
- **Price Direction Model**: LSTM/Transformer — predicts next candle direction (BUY/SELL/NEUTRAL) with probability
- **Volatility Forecast Model**: GARCH + ML hybrid — forecasts expected volatility
- **Breakout Prediction Model**: Gradient Boosting — probability of breakout within N candles
- **Sentiment Model**: FinBERT fine-tuned on Indian market news — sentiment score per symbol
- **Anomaly Detection**: Isolation Forest — detects unusual price/volume behavior
- **Sector Rotation Predictor**: Detects early sector strength shifts

#### 7C — Explainability Layer (XAI)
- SHAP (SHapley Additive exPlanations) values for every prediction
- Human-readable explanation generation: *"RSI oversold condition (28.3) + MACD bullish crossover + high institutional buying in last 3 sessions contributed 73% to this BUY signal"*
- Confidence interval reporting
- Feature importance ranking per signal

#### 7D — Model Registry & Lifecycle
- MLflow for model versioning and experiment tracking
- A/B testing framework for model comparison
- Automated model retraining trigger (performance drift detection)
- Champion/Challenger model deployment strategy

**Inputs**
- `EnrichedCandleEvent`, `IndicatorSnapshotEvent`, `NewsEvent`, `FundamentalEvent` from Kafka
- Historical data from TimescaleDB (for batch training)

**Outputs**
- `AIPredictionEvent` to Kafka topic `ai.predictions`
  - Contains: symbol, model_name, prediction, probability, confidence, shap_values, explanation_text, model_version
- Model artifacts stored in S3/Object Storage

**Dependencies**
- Kafka (I/O)
- TimescaleDB (historical training data)
- Feature Store (Feast or Redis-based)
- MLflow (model registry)
- GPU compute for training (NVIDIA T4/A100 on cloud)
- S3 (model artifact storage)

**Why It Exists**
Rule-based strategies alone have diminishing returns in dynamic markets. The AI engine learns from historical patterns that rules cannot express. XAI is non-negotiable — users will not trust signals they cannot understand.

---

### Module 8 — Signal Aggregation & Scoring Service

**Purpose**
Merges raw rule-based signals from the Strategy Engine with AI predictions to produce final, high-confidence, actionable intelligence signals.

**Responsibilities**
- Consumes both `RawSignalEvent` and `AIPredictionEvent`
- Signal fusion logic: weighted combination of rule-based confidence + AI probability
- Signal quality scoring (A/B/C grade)
- Risk/reward calculation: automatic SL and target computation using ATR
- Signal deduplication and conflict resolution (e.g., BUY signal vs SELL prediction on same symbol)
- Signal enrichment with fundamental data context
- Signal persistence for audit and backtesting

**Inputs**
- `RawSignalEvent` from Kafka `signals.raw`
- `AIPredictionEvent` from Kafka `ai.predictions`

**Outputs**
- `FinalSignalEvent` to Kafka topic `signals.final`
  - Contains: symbol, direction, combined_confidence, grade (A/B/C), entry_price, sl, target, risk_reward_ratio, ai_explanation, strategy_names, timeframe

**Dependencies**
- Kafka
- PostgreSQL (signal history)
- TimescaleDB (price context)

**Why It Exists**
Neither pure rule-based nor pure AI is optimal. This service is the "judgement layer" that intelligently combines both — a fusion architecture proven to outperform either in isolation.

---

### Module 9 — Portfolio Intelligence Service

**Purpose**
Manages user portfolios and delivers AI-powered portfolio analytics, risk assessment, and rebalancing suggestions.

**Responsibilities**
- Portfolio position management (manual entry + broker import)
- Real-time P&L computation (unrealized/realized)
- Portfolio-level risk metrics: Beta, Sharpe Ratio, Sortino Ratio, Max Drawdown, VaR (Value at Risk)
- Sector and market-cap concentration analysis
- Correlation analysis within portfolio
- AI-powered rebalancing suggestions
- Portfolio health score (0–100)
- Dividend calendar and corporate action impact analysis
- Tax P&L report (FIFO/LIFO, STCG/LTCG computation — India-specific)
- Benchmark comparison (vs NIFTY 50, NIFTY 500)

**Inputs**
- User portfolio holdings (manual or via broker API)
- `FinalSignalEvent` (to highlight signals on held stocks)
- `EnrichedCandleEvent` (live price for P&L)

**Outputs**
- Portfolio snapshots stored in PostgreSQL
- Portfolio events for alerts: `PortfolioAlertEvent` (e.g., stock crosses SL, target hit)
- API responses to Dashboard and Mobile

**Dependencies**
- PostgreSQL (portfolio data)
- TimescaleDB (price history for metrics)
- Kafka (for live price subscriptions)
- Broker Integration Service (future)

**Why It Exists**
Signals alone are insufficient for users who hold positions. Portfolio intelligence contextualizes signals against existing holdings — preventing conflicting positions and quantifying actual portfolio-level risk.

---

### Module 10 — Alert Engine

**Purpose**
The real-time intelligence distribution core. Monitors all events and generates personalized, contextual alerts for users based on their watchlists, portfolio, and custom conditions.

**Responsibilities**
- Alert rule management: user-defined conditions (price alerts, indicator alerts, signal alerts, portfolio alerts)
- Real-time event evaluation against alert rules (CEP — Complex Event Processing)
- Alert deduplication (no spam — max 1 alert per symbol per condition per session)
- Alert priority scoring (Critical / High / Medium / Low)
- Alert personalization (only alerts for symbols in user's watchlist or portfolio)
- Tenant-level alert quota enforcement (per subscription tier)
- Alert history and audit log

**Inputs**
- `FinalSignalEvent` from Kafka
- `EnrichedCandleEvent` (price-level alerts)
- `PortfolioAlertEvent` from Portfolio Intelligence Service
- User alert rule configurations from PostgreSQL

**Outputs**
- `AlertEvent` to Kafka topic `alerts.outbound`
  - Contains: user_id, tenant_id, alert_type, priority, message, signal_data, delivery_channels

**Dependencies**
- Kafka
- Redis (active rule cache, deduplication state)
- PostgreSQL (alert rules, alert history)

**Why It Exists**
Without a dedicated alert engine, alert logic bleeds into every other service. This service is the single source of truth for "who gets alerted, when, and why" — enabling powerful customization without coupling.

---

### Module 11 — Notification Delivery Service

**Purpose**
Handles the reliable, multi-channel delivery of all outbound alerts and notifications to end users.

**Responsibilities**
- Channel routing: Telegram Bot, WhatsApp Business API, Email (SMTP/SendGrid), Push Notifications (FCM for Android, APNs for iOS), In-app Notifications
- Message templating and formatting per channel
- Retry logic with exponential backoff
- Delivery receipt tracking
- Rate limiting per channel (e.g., WhatsApp: 1,000 msgs/day per user limit compliance)
- Opt-in/opt-out preference management
- Notification digests (daily summary emails)
- Channel failover (if Telegram fails → fallback to Email)

**Inputs**
- `AlertEvent` from Kafka topic `alerts.outbound`

**Outputs**
- Delivered notifications to Telegram, WhatsApp, Email, Push
- Delivery status events back to PostgreSQL

**Dependencies**
- Kafka
- Telegram Bot API
- WhatsApp Business API (Meta Cloud API or provider like Interakt/Gupshup)
- SendGrid / AWS SES (Email)
- Firebase Cloud Messaging (Push)
- PostgreSQL (delivery logs, user preferences)
- Redis (rate limit counters per channel)

**Why It Exists**
Delivery logic is operationally complex (API rate limits, retries, receipts, user preferences). Isolating it prevents delivery failures from blocking the alert engine, and allows adding new channels (e.g., SMS, Discord) without touching business logic.

---

### Module 12 — Backtesting Engine

**Purpose**
Enables users to validate trading strategies and AI models against historical market data with institutional-grade accuracy.

**Responsibilities**
- Strategy backtesting on historical OHLCV data
- Simulation of order execution with realistic market conditions:
  - Slippage modeling
  - Impact cost
  - Transaction costs (brokerage, STT, GST, exchange fees)
  - NSE/BSE circuit breaker enforcement
- Performance reporting: CAGR, Sharpe, Sortino, Max Drawdown, Win Rate, Profit Factor, Expectancy
- Benchmark comparison (vs buy-and-hold NIFTY 50)
- Walk-forward analysis
- Monte Carlo simulation
- Multi-strategy comparison
- Report export (PDF/Excel)
- Async job execution (backtest jobs queued and processed asynchronously)

**Inputs**
- User backtest configuration (strategy, symbols, date range, capital, parameters)
- Historical OHLCV data from TimescaleDB

**Outputs**
- Backtest result reports stored in PostgreSQL/S3
- Streamed progress updates via WebSocket

**Dependencies**
- TimescaleDB (historical data)
- PostgreSQL (job queue, results)
- S3 (report PDFs)
- Celery + Redis (async job queue)
- WebSocket (real-time progress streaming)

**Why It Exists**
Users must be able to validate any strategy before deploying it — even in paper trading. Without backtesting, the platform encourages uninformed decisions. This module builds user trust and differentiates us from basic signal providers.

---

### Module 13 — Paper Trading Engine

**Purpose**
A simulated trading environment that mirrors real market conditions, allowing users to practice and validate strategies without real capital at risk.

**Responsibilities**
- Virtual portfolio management (INR-denominated virtual capital)
- Order simulation: Market, Limit, SL, SL-M order types
- Real-time order matching against live market prices
- Slippage simulation
- Brokerage and tax simulation (Zerodha-equivalent fee structure)
- Position management: open, modify, exit
- Paper trading P&L tracking (real-time and historical)
- Leaderboard (optional gamification)
- Paper trading → Live trading migration readiness (future)
- Multiple virtual portfolios per user

**Inputs**
- User order requests (via API)
- Live price feed from Market Data Processing Service

**Outputs**
- Virtual trade executions
- Paper portfolio state
- Performance analytics

**Dependencies**
- Kafka (live price feed subscription)
- PostgreSQL (virtual portfolio, orders, transactions)
- Redis (live price cache for order matching)

**Why It Exists**
Strategy validation in a live environment requires real capital risk. Paper trading eliminates that barrier, increases platform engagement, and serves as the testing ground before broker integration goes live.

---

### Module 14 — Broker Integration Service (Future-Ready)

**Purpose**
A pluggable integration layer for executing real trades through Indian brokers, designed for future activation.

**Responsibilities**
- Broker adapter pattern: one interface, multiple broker implementations
  - Zerodha Kite Connect
  - Angel One SmartAPI
  - Upstox API
  - Dhan API
  - Fyers API
- OAuth flow for broker account linking
- Order placement, modification, cancellation
- Live position and holdings sync
- Fund and margin data
- Real-time order status streaming (WebSocket)
- Multi-broker per user support

**Inputs**
- Order requests from Paper Trading Engine (initially), live trading module (future)

**Outputs**
- Order execution confirmations
- Live portfolio sync events

**Dependencies**
- PostgreSQL (broker credentials — encrypted)
- Kafka (order events)
- External: Broker APIs

**Why It Exists**
Built as an adapter pattern today so that when broker integration is activated, zero changes are needed in the Portfolio, Alert, or Paper Trading services. The interface is stable; implementations are swappable.

---

### Module 15 — Web Dashboard (Frontend)

**Purpose**
The primary user interface — a professional-grade, real-time financial intelligence dashboard.

**Responsibilities**
- Authentication flows (login, MFA, OAuth)
- Real-time market dashboard (indices, sector heatmap, market breadth)
- TradingView-grade charting with overlay indicators
- Signal feed with AI explanations
- Portfolio manager UI
- Alert configuration UI
- Backtesting interface
- Paper trading terminal
- Account settings, subscription management
- Admin panel (for tenant admins)
- Responsive design (desktop-first, tablet-compatible)
- WebSocket connection for live data feeds

**Inputs**
- REST API calls + WebSocket streams from the API Gateway

**Outputs**
- User interactions → API calls
- Visual rendering of all platform data

**Dependencies**
- API Gateway
- WebSocket Server (real-time feeds)

**Why It Exists**
The primary touchpoint for 90% of users. A poor dashboard invalidates excellent backend engineering. This must be a premium, performant, responsive interface.

---

### Module 16 — Mobile Application (Frontend)

**Purpose**
Native mobile access for alert monitoring, portfolio tracking, and signal review — optimized for the mobile-first Indian market.

**Responsibilities**
- Core feature parity with web for consumption features (signals, portfolio, alerts)
- Push notification integration (FCM/APNs)
- Biometric authentication
- Offline data caching (last known state)
- Simplified charting (mobile-optimized)
- Deep links from Telegram/WhatsApp alerts

**Inputs**
- REST API + WebSocket from API Gateway

**Outputs**
- User interactions → API calls

**Dependencies**
- API Gateway
- FCM / APNs (push notifications)

**Why It Exists**
Indian market participants are mobile-first. Telegram/WhatsApp alert delivery without a companion mobile app creates a broken experience. Mobile ensures users can act on signals immediately.

---

### Module 17 — Analytics & Reporting Service

**Purpose**
Platform-level and user-level analytics for business intelligence and reporting.

**Responsibilities**
- Signal performance tracking (win rate, accuracy by strategy, by sector, by timeframe)
- User engagement analytics
- Tenant-level usage reports
- Platform health KPIs (prediction accuracy, alert delivery rate, system latency)
- Custom report generation
- Data export (CSV, Excel, PDF)
- Admin dashboard feeds

**Inputs**
- All Kafka event streams (subscribed via analytics consumer group)
- Trade results from Paper Trading Engine
- Backtest results

**Outputs**
- Analytics data stored in ClickHouse (OLAP)
- Reports delivered via API and scheduled emails

**Dependencies**
- Kafka (event consumption)
- ClickHouse (OLAP analytics database)
- S3 (report files)
- PostgreSQL (metadata)

**Why It Exists**
Without tracking signal accuracy and platform performance, the team cannot improve models, the users cannot measure strategy effectiveness, and the business cannot measure SaaS health. Analytics is not optional at enterprise grade.

---

### Module 18 — Observability Stack

**Purpose**
Full-stack observability: centralized logging, distributed tracing, and real-time metrics monitoring across all services.

**Responsibilities**
- Structured JSON logging from all services
- Distributed tracing (trace_id propagated across service calls)
- Application metrics: latency, throughput, error rates, queue depths
- Infrastructure metrics: CPU, memory, disk, network per pod
- Custom business metrics: signal generation rate, alert delivery rate, active users
- Alerting on metric thresholds (PagerDuty integration)
- Log retention policy (30 days hot, 1 year cold in S3)
- Anomaly detection on platform metrics (automated incident detection)

**Components**
- **Logging**: Structured JSON → Filebeat → Elasticsearch → Kibana (ELK Stack)
- **Metrics**: Prometheus (scraping) → Grafana (dashboards)
- **Tracing**: OpenTelemetry → Jaeger / Tempo
- **Alerting**: Alertmanager → PagerDuty / Slack
- **Uptime**: Uptime Kuma or Grafana OnCall

**Why It Exists**
At 5,000+ concurrent users, debugging without observability is impossible. Distributed tracing is non-negotiable in a microservices architecture. This stack is the engineering team's command center.

---

## Complete Data Flows

### Flow 1 — Real-Time Market Data Flow

```
NSE/BSE/Vendor WebSocket
        │
        ▼
[Market Data Ingestion Service]
  - Normalize to canonical schema
  - Validate and clean
        │
        ▼ Kafka: raw.market.ticks
        │
[Market Data Processing Service]
  - Aggregate to OHLCV candles
  - Compute 50+ technical indicators
  - Detect patterns
        │
        ├──▶ Kafka: processed.candles
        ├──▶ Kafka: processed.indicators
        └──▶ TimescaleDB (persistence)
```

### Flow 2 — AI Decision Flow

```
Kafka: processed.candles + processed.indicators
        │
        ▼
[Feature Engineering Pipeline] (Kafka Streams)
  - Build 150+ feature vector per symbol
        │
        ├──▶ [Online Inference] ──▶ Real-time prediction
        │         │
        │         ▼
        │   [Prediction Models]
        │   - Price Direction (LSTM/Transformer)
        │   - Volatility Forecast
        │   - Breakout Prediction
        │   - Sentiment (FinBERT)
        │   - Anomaly Detection
        │         │
        │         ▼
        │   [XAI Layer]
        │   - SHAP values computation
        │   - Human-readable explanation
        │         │
        │         ▼
        │   Kafka: ai.predictions
        │
        └──▶ [Offline Training] (Nightly batch)
              - Retrain models on latest data
              - Evaluate vs champion model
              - Promote if better performance
              - Store in MLflow + S3
```

### Flow 3 — Signal & Alert Generation Flow

```
Kafka: processed.indicators
        │
        ▼
[Strategy Engine]
  - Evaluate 15+ built-in strategies
  - Scan 2,000+ NSE symbols
  - Score confidence per signal
        │
        ▼ Kafka: signals.raw
        │
[Signal Aggregation Service] ◀────── Kafka: ai.predictions
  - Fuse rule-based + AI signals
  - Compute final confidence grade
  - Calculate SL, Target, R:R
        │
        ▼ Kafka: signals.final
        │
[Alert Engine]
  - Match signals to user alert rules
  - Deduplication check
  - Priority scoring
        │
        ▼ Kafka: alerts.outbound
        │
[Notification Delivery Service]
  - Route to channels
  ├──▶ Telegram Bot API
  ├──▶ WhatsApp Business API
  ├──▶ Email (SendGrid)
  ├──▶ Push Notification (FCM/APNs)
  └──▶ In-App WebSocket
```

### Flow 4 — User Request Flow (Web/Mobile)

```
User (Browser / Mobile App)
        │
        ▼ HTTPS / WSS
[API Gateway]
  - TLS termination
  - JWT validation
  - Rate limiting
  - Tenant extraction
        │
        ├──▶ REST request ──▶ [Appropriate Microservice]
        │                            │
        │                            ▼
        │                     [PostgreSQL / TimescaleDB / Redis]
        │                            │
        │                            ▼
        │                     Response ──▶ API Gateway ──▶ Client
        │
        └──▶ WebSocket upgrade ──▶ [WebSocket Server]
                                         │
                                   Subscribe to Redis Pub/Sub
                                   channels per user/watchlist
                                         │
                                   Real-time push of:
                                   - Live prices
                                   - New signals
                                   - Portfolio P&L
                                   - Alerts
```

### Flow 5 — Backtesting Flow

```
User submits backtest job (strategy + symbols + date range)
        │
        ▼
[API Gateway] ──▶ [Backtesting Service]
  - Job queued in Celery (Redis broker)
  - Job ID returned to user immediately
        │
[Celery Worker picks up job]
  - Fetches historical data from TimescaleDB
  - Simulates strategy execution bar-by-bar
  - Applies realistic costs (slippage, brokerage, taxes)
        │
  - Streams progress % via WebSocket
        │
  - Job complete: results stored in PostgreSQL + S3
        │
[User views result in Dashboard]
  - Performance metrics, equity curve, trade log
  - PDF report available for download
```

---

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CLOUD PROVIDER (AWS / GCP)                          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    GLOBAL LOAD BALANCER / CDN                       │   │
│  │              (AWS CloudFront / Cloudflare)                          │   │
│  └──────────────────────────────┬──────────────────────────────────────┘   │
│                                 │                                           │
│  ┌──────────────────────────────▼──────────────────────────────────────┐   │
│  │                    API GATEWAY CLUSTER                               │   │
│  │              (Kong Gateway — 3+ instances, HPA)                     │   │
│  └──────┬───────────────────────────────────────────────────────┬──────┘   │
│         │                                                       │           │
│  ┌──────▼──────────────────────────────────────────────────────▼──────┐   │
│  │                  KUBERNETES CLUSTER (EKS / GKE)                    │   │
│  │                                                                     │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │ Auth Service │  │ Tenant Svc   │  │ Portfolio Svc│            │   │
│  │  │ (2-5 pods)   │  │ (2-3 pods)   │  │ (3-10 pods)  │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │ Strategy Eng │  │ Signal Agg   │  │ Alert Engine │            │   │
│  │  │ (5-20 pods)  │  │ (3-10 pods)  │  │ (5-20 pods)  │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │   │
│  │  │ Notification │  │ Backtesting  │  │ Paper Trading│            │   │
│  │  │ (3-10 pods)  │  │ (Celery 2-20)│  │ (3-10 pods)  │            │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘            │   │
│  │  ┌─────────────────────────────────────────────────┐             │   │
│  │  │              AI/ML Engine                       │             │   │
│  │  │  Online Inference: 3-10 CPU pods (fast)         │             │   │
│  │  │  Offline Training: GPU node pool (scheduled)    │             │   │
│  │  └─────────────────────────────────────────────────┘             │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────┐   ┌──────────────────────────────────────────┐   │
│  │   MESSAGE BUS        │   │          DATA LAYER                      │   │
│  │                      │   │                                          │   │
│  │  Apache Kafka        │   │  PostgreSQL (RDS Multi-AZ)               │   │
│  │  (3-broker cluster)  │   │  TimescaleDB (Managed/Self-hosted)       │   │
│  │  MSK on AWS          │   │  Redis Cluster (ElastiCache)             │   │
│  │                      │   │  ClickHouse (Analytics)                  │   │
│  └──────────────────────┘   │  S3 (Models, Reports, Backups)          │   │
│                              └──────────────────────────────────────────┘   │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    OBSERVABILITY STACK                               │   │
│  │  ELK Stack   |   Prometheus + Grafana   |   Jaeger (Tracing)        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## High-Level Architecture Diagram (ASCII)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║               AI-POWERED INDIAN STOCK MARKET INTELLIGENCE PLATFORM          ║
║                          HIGH-LEVEL ARCHITECTURE                             ║
╚══════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────┐
│                            CLIENT LAYER                                      │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────────────┐    │
│  │   Web Dashboard  │  │  Mobile App      │  │  External API Clients  │    │
│  │  (Next.js/React) │  │ (React Native)   │  │  (Broker, Partners)    │    │
│  └────────┬─────────┘  └────────┬─────────┘  └───────────┬────────────┘    │
└───────────┼────────────────────┼────────────────────────┼─────────────────┘
            └────────────────────┴────────────────────────┘
                                 │ HTTPS / WSS
┌────────────────────────────────▼────────────────────────────────────────────┐
│                    PLATFORM CORE — KUBERNETES                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  API GATEWAY (Kong) + AUTH SERVICE + TENANT SERVICE                  │  │
│  └─────────────────────────────┬────────────────────────────────────────┘  │
│                                 │                                            │
│  ┌──────────────────────────────┼────────────────────────────────────────┐  │
│  │                    BUSINESS SERVICES LAYER                            │  │
│  │  ┌─────────────┐  ┌────────────────┐  ┌────────────┐  ┌──────────┐  │  │
│  │  │  Strategy   │  │   Portfolio    │  │  Alert     │  │ Backtest │  │  │
│  │  │  Engine     │  │  Intelligence  │  │  Engine    │  │  Engine  │  │  │
│  │  └──────┬──────┘  └───────┬────────┘  └─────┬──────┘  └────┬─────┘  │  │
│  │         └─────────────────┼─────────────────┼─────────────┘         │  │
│  └─────────────────────────────┼───────────────┼──────────────────────┘  │
│                                 │               │                            │
│  ┌──────────────────────────────▼───────────────▼──────────────────────┐  │
│  │                  APACHE KAFKA (Event Bus)                            │  │
│  │  Topics: raw.ticks | processed.candles | signals.raw |              │  │
│  │          signals.final | ai.predictions | alerts.outbound           │  │
│  └──────┬────────────────────────────────────────────────────┬─────────┘  │
│         │                                                     │             │
│  ┌──────▼─────────────┐                         ┌────────────▼──────────┐  │
│  │  DATA PIPELINE     │                         │  AI / ML ENGINE       │  │
│  │                    │                         │                       │  │
│  │ Market Data        │                         │ Feature Engineering   │  │
│  │ Ingestion          │                         │ Prediction Models     │  │
│  │    │               │                         │ XAI / SHAP            │  │
│  │ Market Data        │                         │ Model Registry        │  │
│  │ Processing         │                         │ (MLflow)              │  │
│  └──────┬─────────────┘                         └────────────┬──────────┘  │
│         │                                                     │             │
│  ┌──────▼─────────────────────────────────────────────────────▼──────────┐  │
│  │                       DATA LAYER                                      │  │
│  │  PostgreSQL │ TimescaleDB │ Redis Cluster │ ClickHouse │ S3          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                   NOTIFICATION DELIVERY                               │  │
│  │  Telegram │ WhatsApp │ Email (SendGrid) │ Push (FCM/APNs) │ In-App  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │              OBSERVABILITY: ELK + Prometheus + Grafana + Jaeger       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Module Interaction Diagram

```
                        ┌─────────────────┐
                        │  External APIs   │
                        │ NSE/BSE/Vendors  │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │  Market Data    │
                        │  Ingestion      │
                        └────────┬────────┘
                                 │ Kafka: raw.market.ticks
                        ┌────────▼────────┐
                        │  Market Data    │◀──────────────────────┐
                        │  Processing     │                       │
                        └────┬──────┬─────┘                       │
          Kafka: candles │    │ Kafka: indicators              TimescaleDB
                         │    └──────────────────┐               (Read)
             ┌───────────┘                       │
             │                           ┌───────▼──────┐
    ┌────────▼───────┐                   │  AI / ML     │
    │  Strategy      │                   │  Engine      │
    │  Engine        │                   └───────┬──────┘
    └────────┬───────┘     Kafka: ai.predictions │
Kafka:       │             ┌─────────────────────┘
signals.raw  │             │
    ┌─────────▼─────────────▼───────────┐
    │   Signal Aggregation & Scoring    │
    └────────────────┬──────────────────┘
             Kafka: signals.final
    ┌────────────────▼──────────────────┐
    │         Alert Engine              │◀──── Portfolio Intelligence
    └────────────────┬──────────────────┘
             Kafka: alerts.outbound
    ┌────────────────▼──────────────────┐
    │    Notification Delivery          │
    └───────┬────────┬───────┬──────────┘
            │        │       │
         Telegram  Email  WhatsApp
                           Push
```

---

## Technology Stack Recommendation

### Core Infrastructure

| Category | Technology | Rationale |
|---|---|---|
| Container Orchestration | **Kubernetes (K8s)** | Industry standard; HPA for auto-scaling |
| Cloud Provider | **AWS (primary)** | MSK (Kafka), RDS, EKS, ElastiCache all managed |
| Message Broker | **Apache Kafka** | 10M+ msg/sec throughput; event replay; proven in fintech |
| API Gateway | **Kong Gateway** | Plugin ecosystem; rate limiting; JWT; WebSocket support |
| Service Mesh | **Istio** | mTLS between services; traffic management; observability |

### Data Layer

| Category | Technology | Rationale |
|---|---|---|
| Primary DB | **PostgreSQL 16** | ACID, JSONB, row-level security for multi-tenancy |
| Time-Series DB | **TimescaleDB** | PostgreSQL extension; optimized for OHLCV tick data; SQL-compatible |
| Cache / Pub-Sub | **Redis Cluster** | Sub-millisecond latency; Pub/Sub for WebSocket feeds |
| Analytics (OLAP) | **ClickHouse** | 100x faster than PostgreSQL for analytical queries |
| Object Storage | **AWS S3** | Model artifacts, backtest reports, log archives |
| Search | **Elasticsearch** | Log search + symbol search (full-text) |

### Backend Services

| Category | Technology | Rationale |
|---|---|---|
| AI/ML Services | **Python + FastAPI** | Ecosystem: PyTorch, scikit-learn, pandas, TA-Lib |
| Business Services | **Python + FastAPI** | Uniform stack; async support; type hints; Pydantic |
| Real-Time Service | **Node.js (NestJS)** | WebSocket performance for 5,000+ concurrent connections |
| Task Queue | **Celery + Redis** | Backtesting async jobs; notification retries |
| ML Framework | **PyTorch** | LSTM/Transformer models; dynamic graphs |
| XAI | **SHAP** | Industry standard for model explainability |
| Indicators | **TA-Lib + pandas-ta** | 150+ technical indicators, battle-tested |
| Model Registry | **MLflow** | Experiment tracking; model versioning; A/B testing |
| Feature Store | **Feast (Redis online)** | Consistent features between training and serving |

### Frontend

| Category | Technology | Rationale |
|---|---|---|
| Web Dashboard | **Next.js 14 (React)** | SSR/SSG; App Router; TypeScript |
| Mobile App | **React Native** | Code sharing with web; single team |
| Charting | **TradingView Lightweight Charts** | Professional financial charts; free for SaaS |
| State Management | **Zustand + React Query** | Lightweight; server-state optimized |
| UI Component Library | **Shadcn/UI + Radix** | Accessible; unstyled; customizable |

### Observability

| Category | Technology | Rationale |
|---|---|---|
| Logging | **ELK Stack** (Elasticsearch + Logstash + Kibana) | Industry standard; structured log search |
| Metrics | **Prometheus + Grafana** | Pull-based metrics; 500+ pre-built dashboards |
| Tracing | **OpenTelemetry + Jaeger** | Vendor-neutral; distributed trace visualization |
| Alerting | **Alertmanager + PagerDuty** | On-call escalation for production incidents |
| Uptime | **Grafana OnCall** | SLA monitoring and incident management |

### DevOps & CI/CD

| Category | Technology | Rationale |
|---|---|---|
| CI/CD | **GitHub Actions** | Native integration; matrix builds |
| Container Registry | **AWS ECR** | Private; integrated with EKS |
| IaC | **Terraform** | Declarative cloud infra; state management |
| Secret Management | **AWS Secrets Manager + Vault** | Encrypted credential storage; rotation |
| GitOps | **ArgoCD** | Kubernetes deployment from Git; rollback |

---

## Scalability Strategy

### Horizontal Scaling

| Service | Scaling Trigger | Min Pods | Max Pods |
|---|---|---|---|
| API Gateway | CPU > 60% or RPS > 1,000 | 3 | 20 |
| Market Data Ingestion | Market hours only (fixed) | 2 | 5 |
| Market Data Processing | Kafka consumer lag > 1,000 | 3 | 30 |
| Strategy Engine | Kafka consumer lag > 500 | 5 | 50 |
| AI Inference | Queue depth > 100 | 3 | 20 |
| Alert Engine | Kafka consumer lag > 200 | 5 | 30 |
| Notification Delivery | Queue depth > 500 | 3 | 20 |
| Backtesting Workers | Job queue depth > 5 | 0 | 50 |

### Database Scaling

- **PostgreSQL**: Read replicas for all read-heavy services. Write to primary only.
- **TimescaleDB**: Chunk-based partitioning by time; automatic data tiering (hot/warm/cold)
- **Redis**: Cluster mode with 6 nodes (3 primary + 3 replica)
- **Kafka**: 3-broker cluster; topic partitioning based on symbol hash (parallel processing per symbol)

### Caching Strategy

| Cache Layer | Technology | TTL | Purpose |
|---|---|---|---|
| Live Prices | Redis | 1 second | Order matching, dashboard |
| Indicator Snapshots | Redis | 1 minute | Strategy evaluation |
| User Profile + Auth | Redis | 15 minutes | Reduce DB hits |
| Signal Feed | Redis | 5 minutes | API response cache |
| Historical OHLCV | Redis (LRU) | 1 hour | Charting API |

---

## Fault Tolerance

| Failure Scenario | Mitigation |
|---|---|
| Market Data Vendor outage | Multi-vendor fallback; automatic failover to secondary vendor |
| Kafka broker failure | 3-broker cluster; replication factor 3; min ISR 2 |
| PostgreSQL primary failure | Multi-AZ RDS; automatic failover < 60 seconds |
| Service pod crash | Kubernetes restarts automatically; readiness probes prevent traffic to unhealthy pods |
| AI model inference failure | Fallback to rule-based signals only; no platform outage |
| Notification channel failure | Channel failover (Telegram → Email); retry with exponential backoff |
| Backtesting worker crash | Job is requeued; idempotent job design |
| Redis cache failure | Services degrade gracefully to DB; cache-aside pattern |
| Network partition | Circuit breaker pattern (Hystrix/Resilience4j) on all inter-service calls |

---

## Security Architecture

### Layers of Security

```
Layer 1 — Network:       VPC, Private subnets, Security Groups, WAF, DDoS protection
Layer 2 — Transport:     TLS 1.3 enforced everywhere; mTLS between internal services (Istio)
Layer 3 — Authentication: JWT (RS256), OAuth 2.0, MFA, API key rotation
Layer 4 — Authorization:  RBAC, Tenant isolation (row-level security in PostgreSQL)
Layer 5 — Data:          Encryption at rest (AES-256), Encryption in transit (TLS)
Layer 6 — Application:   Input validation (Pydantic), SQL injection prevention (ORM)
Layer 7 — Secrets:       AWS Secrets Manager; no secrets in code or environment variables
Layer 8 — Compliance:    SEBI data handling guidelines; DPDPA (India Digital Personal Data Protection Act)
```

### Tenant Isolation
- All database tables include `tenant_id` column
- PostgreSQL Row-Level Security (RLS) policies enforce tenant boundaries at the DB layer
- Separate Redis key namespaces per tenant: `tenant:{id}:*`
- Kafka topics are shared; consumer groups are tenant-aware

### Broker Credential Security
- Broker API keys stored encrypted (AES-256) in AWS Secrets Manager
- Keys are never logged, never included in API responses
- Separate IAM roles per service (principle of least privilege)

---

## Logging Architecture

```
All Services → Structured JSON Logs
                     │
              Filebeat (sidecar per pod)
                     │
              Logstash (enrichment: add trace_id, tenant_id, service_name)
                     │
              Elasticsearch (indexed storage)
                     │
              Kibana (search, dashboards, alerts)
                     │
              S3 (cold archive after 30 days)
```

### Log Schema (every log line)
```json
{
  "timestamp": "ISO8601",
  "level": "INFO|WARN|ERROR|DEBUG",
  "service": "service-name",
  "trace_id": "distributed-trace-uuid",
  "span_id": "span-uuid",
  "tenant_id": "uuid",
  "user_id": "uuid",
  "event": "human_readable_event_name",
  "duration_ms": 42,
  "data": {}
}
```

---

## Monitoring Architecture

### Metric Categories

| Category | Key Metrics | Alert Threshold |
|---|---|---|
| **Business** | Signals/hour, Alert delivery rate, Active users | < 90% delivery rate |
| **Latency** | API P99 latency, Signal generation lag | API P99 > 500ms |
| **Throughput** | Kafka msg/sec, DB queries/sec | Kafka lag > 10,000 |
| **Error Rate** | 5xx rate, Failed alerts, Model errors | 5xx > 1% |
| **Infrastructure** | CPU, Memory, Disk I/O, Network | CPU > 80% for 5min |
| **AI Model** | Prediction accuracy, Inference latency | Accuracy < 55% |

### Dashboards (Grafana)
1. **Platform Overview**: All services health, active users, signal volume
2. **Market Data Pipeline**: Tick ingestion rate, processing lag, indicator freshness
3. **AI Engine**: Model inference latency, prediction distribution, accuracy tracking
4. **Alert Engine**: Alert generation rate, delivery success rate per channel
5. **Infrastructure**: Kubernetes cluster health, database connections, Kafka consumer lag
6. **Business KPIs**: DAU/MAU, subscription revenue, churn indicators

---

## Development Order (Phased Execution)

### Phase 3 — Database Design
Design all schemas: PostgreSQL, TimescaleDB, Redis data structures, Kafka topic schemas

### Phase 4 — API Design
Design all REST + WebSocket APIs; OpenAPI 3.0 specification; versioning strategy

### Phase 5 — Backend Services (in order)
1. Auth Service + Tenant Service (foundational — everything depends on this)
2. Market Data Ingestion Service
3. Market Data Processing Service
4. Strategy Engine
5. Signal Aggregation Service
6. Alert Engine
7. Notification Delivery Service
8. Portfolio Intelligence Service

### Phase 6 — AI Engine
1. Feature Engineering Pipeline
2. Prediction Models (start with Gradient Boosting, then LSTM)
3. XAI Layer (SHAP integration)
4. Model Registry (MLflow setup)

### Phase 7 — Dashboard (Web Frontend)
1. Authentication flows
2. Market dashboard + real-time charts
3. Signal feed with AI explanations
4. Portfolio manager
5. Alert configuration UI

### Phase 8 — Backtesting Engine
Full historical simulation with realistic costs

### Phase 9 — Paper Trading Engine
Live price simulation with virtual capital

### Phase 10 — Alert Engine (Advanced)
Complex event processing, custom alert rules

### Phase 11 — Deployment
Kubernetes manifests, Terraform IaC, CI/CD pipelines

### Phase 12 — Production Readiness
Load testing, security audit, observability validation, runbooks

---

## Open Architecture Questions

> [!IMPORTANT]
> **Q1 — Market Data Vendor**: Which data vendors do you have access to or plan to subscribe to? (Zerodha Kite, Angel One, Upstox, NSE direct, Dhan, Fyers?) This determines the ingestion service design.

> [!IMPORTANT]
> **Q2 — Cloud Provider**: Confirmed AWS? Or do you prefer GCP or Azure? Or a hybrid/on-premise approach given SEBI data localization requirements?

> [!IMPORTANT]
> **Q3 — WhatsApp Integration**: WhatsApp Business API requires Meta Business Verification. Do you have this, or should we use a BSP (Business Solution Provider) like Interakt, Gupshup, or Wati?

> [!IMPORTANT]
> **Q4 — AI Model Depth**: Should the AI engine initially focus on technical analysis-based ML models, or do you also want NLP-based news sentiment models (FinBERT) from Phase 6?

> [!IMPORTANT]
> **Q5 — Broker Integration Timeline**: Should Phase 14 (Broker Integration) be built as a stub/interface now, or strictly deferred with only the adapter pattern defined?

> [!NOTE]
> **Q6 — Regulatory Compliance**: Are you registering as a SEBI Research Analyst (RA)? This determines how signals are presented — advice vs. information. This has legal and architectural implications for disclaimers and signal classification.

---

*Architecture Version 1.0 — Pending User Review and Approval*
*Next Phase: Phase 3 — Database Design (awaiting approval)*
