# PROJECT CHARTER — Permanent Design Rules
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Authority**: Project Owner
> **Status**: PERMANENT — These rules govern every design decision in this project.
> **Last Updated**: 2026-07-10
> **Override Policy**: Only the project owner can explicitly change these rules.

---

## ⚠️ Non-Negotiable Project Identity

This platform is:
- ✅ A **production-grade, enterprise SaaS platform**
- ✅ An **AI-powered market intelligence system**
- ✅ A **multi-tenant, subscription-based business**

This platform is NOT:
- ❌ A simple stock screener
- ❌ A trading bot
- ❌ A signal generator
- ❌ An MVP or prototype

---

## 📏 Permanent Scale Requirements

| Dimension | Requirement |
|---|---|
| **Registered Users** | 100,000+ |
| **Concurrent Sessions** | 5,000+ |
| **Symbols Analyzed** | 2,000+ NSE/BSE in real-time |
| **Signal Latency** | < 3 seconds end-to-end |
| **Alert Delivery** | < 10 seconds |
| **Uptime SLA** | 99.9% during market hours |
| **Data Retention** | 10 years historical minimum |

---

## 🏗️ Permanent Architectural Requirements

Every module, schema, API, and service MUST be designed to support:

### 1. Multi-Tenant SaaS Architecture
- All data is tenant-scoped with `tenant_id` on every table
- PostgreSQL Row-Level Security (RLS) enforced at DB layer
- Tenant-aware caching (Redis key namespacing)
- Tenant-aware Kafka consumers
- No cross-tenant data leakage — ever

### 2. Role-Based Access Control (RBAC)
Four permanent user roles:

| Role | Permissions |
|---|---|
| **Admin** | Full platform access, tenant management, user management, billing |
| **Analyst** | All market data, all signals, backtesting, paper trading, AI insights |
| **Premium User** | Market data, signals, portfolio, alerts, paper trading, backtesting |
| **Free User** | Limited market data, limited signals (quota-enforced), basic alerts |

- Every API endpoint must enforce RBAC
- RBAC enforced at API Gateway + Service layer (defense in depth)
- Roles stored in JWT claims — validated on every request

### 3. Subscription Plans
- **Free Plan**: Limited features, daily quotas, watermarked reports
- **Trial Plan**: Full features, time-limited (14 days), no credit card required
- **Pro / Premium Plan**: Full features, higher quotas
- **Enterprise Plan**: Unlimited, custom branding, dedicated support, API access
- Every feature must be gated by subscription tier
- Feature flags per plan stored in tenant configuration

### 4. Trial Users
- Trial users are full-feature, time-limited
- Grace period handling (expired trial → graceful degradation, not hard block)
- Trial-to-paid conversion tracking in analytics

### 5. Payment Gateway Integration
- **Razorpay** (primary — INR, India-first)
- **Stripe** (secondary — global expansion)
- Webhook-driven subscription lifecycle (created, renewed, failed, cancelled)
- Invoice generation and GST compliance (Indian tax)
- Proration handling for plan upgrades/downgrades
- Dunning management (failed payment retries)

### 6. User Portfolios
- Multiple portfolios per user (personal, family, HUF, etc.)
- Manual entry + future broker sync
- FIFO cost basis (SEBI standard for India)
- STCG / LTCG tax computation
- Real-time P&L with benchmark comparison
- Portfolio health score (AI-computed)

### 7. User Watchlists
- Multiple watchlists per user
- Watchlist-scoped alerts
- Watchlist-scoped signal feed
- Collaborative watchlists (Enterprise — future)
- Max watchlist size enforced by subscription tier

### 8. User-Specific AI Alerts
- Per-user alert rule configuration
- Per-symbol, per-indicator, per-signal alert triggers
- Alert delivery preferences per channel (Telegram, WhatsApp, Email, Push)
- Alert quota enforcement per subscription tier
- Alert deduplication scoped per user
- Alert history per user (minimum 90-day retention)

### 9. AI Explainability (XAI) — Non-Negotiable
- **Every AI prediction MUST carry a SHAP-based explanation**
- Human-readable explanation text required on every signal
- Confidence interval required on every prediction
- Top 5 contributing features displayed to user
- No "black box" signals — ever
- XAI is a core product differentiator, not an optional feature

### 10. Paper Trading
- INR-denominated virtual capital
- Multiple paper portfolios per user
- Realistic order simulation (Market, Limit, SL, SL-M)
- Realistic cost simulation (brokerage, STT, GST, slippage)
- Real-time P&L against live prices
- Paper trading performance analytics

### 11. Backtesting
- Strategy backtesting on historical OHLCV (minimum 10 years)
- Realistic cost modeling (India-specific: STT, brokerage, exchange fees)
- Multi-strategy comparison
- Walk-forward analysis
- Monte Carlo simulation
- Async job execution (never block the user)
- PDF/Excel report export
- Results stored per user (accessible in dashboard)

### 12. Future Broker Integrations
- **Adapter pattern is mandatory** — all broker code behind a common interface
- Planned brokers: Zerodha Kite, Angel One SmartAPI, Upstox, Dhan, Fyers
- OAuth flow for broker account linking
- Broker credentials encrypted at rest (AES-256, AWS Secrets Manager)
- Multi-broker per user support
- No tight coupling between broker code and business logic — ever

### 13. Future Mobile Apps
- All APIs must be mobile-first compatible (paginated, compressed, delta updates)
- WebSocket events designed for mobile consumption
- Push notification infrastructure (FCM + APNs) always-on
- Deep link support from all notification channels
- Offline-capable data structures (last-known state)
- No desktop-only assumptions in any API response

### 14. Future ML Model Upgrades
- MLflow model registry is mandatory — all models versioned
- Champion/Challenger A/B testing framework must always exist
- Model interfaces (input/output schema) are stable contracts
- New models can be deployed without touching business logic
- Feature Store (Feast) ensures training/serving consistency
- Model performance drift detection always running

---

## 🚫 Permanent Design Anti-Patterns (NEVER DO THESE)

| Anti-Pattern | Why Forbidden |
|---|---|
| Hardcoded tenant IDs or user IDs | Breaks multi-tenancy |
| Secrets in code or environment variables | Security violation |
| Synchronous inter-service calls for data | Breaks event-driven architecture |
| Black-box AI signals without explanation | Breaks XAI requirement |
| Features without subscription tier gating | Breaks SaaS business model |
| Single-user assumptions in any data model | Breaks multi-tenancy |
| Skipping RBAC on any endpoint | Security violation |
| Magic numbers without named constants | Maintainability violation |
| Optimizing only for simplicity | Violates scalability requirement |
| Database schema without `tenant_id` | Breaks data isolation |
| Direct broker API calls without adapter layer | Breaks extensibility |

---

## ✅ Permanent Design Mandates (ALWAYS DO THESE)

| Mandate | Reason |
|---|---|
| Every table has `tenant_id` | Multi-tenancy |
| Every table has `created_at`, `updated_at` | Auditability |
| Every API validates JWT + RBAC | Security |
| Every AI signal has SHAP explanation | XAI requirement |
| Every external call has circuit breaker | Fault tolerance |
| Every service emits structured JSON logs | Observability |
| Every service exposes `/metrics` endpoint | Monitoring |
| Every Kafka message has `trace_id` | Distributed tracing |
| Every secret via AWS Secrets Manager | Security |
| Every schema change via Alembic migration | Maintainability |
| Every feature gated by subscription tier | SaaS business model |
| All broker code behind adapter interface | Extensibility |

---

## 🎯 Design Decision Framework

When making any design decision, evaluate in this order:

1. **Security** — Does this expose any risk?
2. **Multi-tenancy** — Is tenant isolation preserved?
3. **RBAC** — Are permissions correctly enforced?
4. **Scalability** — Does this work at 100,000 users?
5. **Observability** — Can we debug this in production?
6. **Maintainability** — Can another engineer understand this in 6 months?
7. **Cost** — Is the infrastructure cost justified?
8. **Simplicity** — Only considered LAST, never first.

---

## 📌 Reference Documents

| Document | Location |
|---|---|
| Architecture Modules (Phase 2A) | `docs/02_Architecture/phase2a_architecture_modules.md` |
| Data Flow Architecture (Phase 2B) | `docs/02_Architecture/phase2b_data_flow_architecture.md` |
| Technology Stack (Phase 2C) | `docs/03_Tech_Stack/phase2c_technology_stack.md` |
| Project Roadmap | `ROADMAP.md` |
| Project Status | `PROJECT_STATUS.md` |

---

*These rules are permanent project law.*
*They apply to every phase, every module, every line of design.*
*Violation of these rules must be explicitly justified and approved by the project owner.*
