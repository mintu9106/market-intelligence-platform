# Roadmap — AI-Powered Indian Stock Market Intelligence Platform

This roadmap defines the complete development trajectory from project initialization to production readiness.

> **Note**: Each phase must be approved before the next begins. Phases are sequential by design — skipping creates technical debt that is expensive to recover.

---

## Phase Overview

```
Phase 1  ██████████  Requirements Analysis          ✅ COMPLETE
Phase 2  ██████████  Enterprise Architecture        ✅ COMPLETE (2A + 2B + 2C)
Phase 3  ░░░░░░░░░░  Database Design                🔄 NEXT
Phase 4  ░░░░░░░░░░  API Design                     ⏳ PENDING
Phase 5  ░░░░░░░░░░  Backend Services               ⏳ PENDING
Phase 6  ░░░░░░░░░░  AI Engine                      ⏳ PENDING
Phase 7  ░░░░░░░░░░  Dashboard                      ⏳ PENDING
Phase 8  ░░░░░░░░░░  Backtesting Engine              ⏳ PENDING
Phase 9  ░░░░░░░░░░  Paper Trading                  ⏳ PENDING
Phase 10 ░░░░░░░░░░  Alert Engine (Advanced)         ⏳ PENDING
Phase 11 ░░░░░░░░░░  Deployment                     ⏳ PENDING
Phase 12 ░░░░░░░░░░  Production Readiness            ⏳ PENDING
```

---

## Phase 1 — Requirements Analysis ✅

**Status**: Complete
**Deliverables**: Platform vision, user personas, feature requirements, constraints

**Decisions Made**:
- Platform type: Enterprise SaaS (not a simple screener or bot)
- Target market: Indian retail investors, HNIs, and professional traders
- Scale: 50,000+ users, 5,000+ concurrent
- Compliance: SEBI, DPDPA alignment required

---

## Phase 2 — Enterprise Architecture ✅

**Status**: Complete (3 sub-phases)
**Deliverables**:

| Sub-Phase | Document | Status |
|---|---|---|
| 2A | Enterprise System Architecture v1.0 (18 modules) | ✅ Approved |
| 2B | Data Flow Architecture v1.0 (11 flows) | ✅ Approved |
| 2C | Technology Stack v1.0 (20 categories) | ✅ Approved |

**Key Architecture Decisions**:
- Event-Driven Architecture with Apache Kafka
- Microservices on Kubernetes (AWS EKS)
- Python FastAPI (backend) + Next.js (frontend) + React Native (mobile)
- PostgreSQL + TimescaleDB + ClickHouse (polyglot persistence)
- PyTorch + XGBoost + SHAP (AI/ML + XAI)
- Keycloak (multi-tenant IAM)

---

## Phase 3 — Database Design 🔄 NEXT

**Status**: Pending Approval to Begin
**Estimated Deliverables**:
- PostgreSQL schema (all tables, indexes, RLS policies)
- TimescaleDB hypertable design (OHLCV, indicators, ticks)
- Redis data structure specifications
- Kafka topic schema (Avro schemas)
- ClickHouse schema (analytics tables)
- Database migration strategy (Alembic)
- Data retention and archival policies

**Exit Criteria**: All schemas approved, migration scripts planned

---

## Phase 4 — API Design

**Status**: Pending Phase 3 Approval
**Estimated Deliverables**:
- OpenAPI 3.0 specification for all REST endpoints
- WebSocket event contract documentation
- API versioning strategy (`/v1/`, `/v2/`)
- Authentication and authorization flow specifications
- Rate limiting policies per endpoint per subscription tier
- Request/response schema definitions (Pydantic models)
- Error response standards

**Exit Criteria**: Full OpenAPI spec approved, mock server validated

---

## Phase 5 — Backend Services

**Status**: Pending Phase 4 Approval
**Services to Build** (in dependency order):

| Order | Service | Depends On |
|---|---|---|
| 1 | Auth Service | Keycloak, PostgreSQL |
| 2 | Tenant & Subscription Service | Auth, PostgreSQL, Razorpay |
| 3 | Market Data Ingestion Service | Kafka, TimescaleDB, Vendor APIs |
| 4 | Market Data Processing Service | Kafka, TimescaleDB, Redis |
| 5 | Strategy Engine | Kafka, Redis, PostgreSQL |
| 6 | Signal Aggregation Service | Kafka, PostgreSQL |
| 7 | Alert Engine | Kafka, Redis, PostgreSQL |
| 8 | Notification Delivery Service | Kafka, PostgreSQL, External APIs |
| 9 | Portfolio Intelligence Service | PostgreSQL, TimescaleDB, Redis |
| 10 | Analytics Service | Kafka, ClickHouse |

**Exit Criteria**: All services pass integration tests, >80% unit test coverage

---

## Phase 6 — AI Engine

**Status**: Pending Phase 5 Approval
**Components**:
- Feature Engineering Pipeline (Kafka Streams + pandas)
- Price Direction Model (LSTM/Transformer — PyTorch)
- Volatility Forecast Model (GARCH + XGBoost hybrid)
- Breakout Prediction Model (XGBoost)
- Sentiment Model (FinBERT fine-tuned on Indian financial corpus)
- Anomaly Detection (Isolation Forest)
- XAI Layer (SHAP — TreeExplainer + DeepExplainer)
- Human-readable explanation generator
- MLflow model registry setup
- Online inference service (FastAPI)
- Offline training pipeline (Celery + Airflow)
- Champion/Challenger A/B testing framework

**Exit Criteria**: All models achieve accuracy > baseline, XAI explanations validated by domain expert

---

## Phase 7 — Web Dashboard

**Status**: Pending Phase 6 Approval
**Features**:
- Authentication flows (login, MFA, OAuth)
- Real-time market dashboard (indices, heatmap, breadth)
- TradingView-grade charting with indicator overlays
- Signal feed with AI explanations (SHAP visualization)
- Portfolio manager (P&L, risk metrics, tax analytics)
- Alert configuration UI (rule builder)
- Backtesting interface (strategy selector, date range, results)
- Paper trading terminal (order entry, positions, history)
- Subscription management (upgrade/downgrade)
- Admin panel (tenant management)

**Exit Criteria**: All screens implemented, Lighthouse score > 90, Playwright E2E tests pass

---

## Phase 8 — Backtesting Engine

**Status**: Pending Phase 7 Approval
**Features**:
- Historical OHLCV simulation (bar-by-bar)
- Realistic cost modeling (STT, brokerage, GST, exchange fees, slippage)
- NSE/BSE circuit breaker enforcement in simulation
- Multi-strategy comparison
- Walk-forward analysis
- Monte Carlo simulation
- Performance report: CAGR, Sharpe, Sortino, Max Drawdown, Win Rate, Profit Factor
- Benchmark comparison (NIFTY 50 buy-and-hold)
- Async job execution with progress streaming
- PDF/Excel report generation and S3 storage

**Exit Criteria**: Backtesting results validated against known historical scenarios, <5% deviation from reference data

---

## Phase 9 — Paper Trading Engine

**Status**: Pending Phase 8 Approval
**Features**:
- Virtual INR portfolio initialization
- Market, Limit, SL, SL-M order types
- Real-time order matching against live prices
- Realistic slippage and brokerage simulation
- Multiple virtual portfolios per user
- Real-time P&L tracking
- Trade history and performance analytics
- Leaderboard (optional gamification layer)

**Exit Criteria**: Order matching accuracy validated, paper P&L consistent with real market prices

---

## Phase 10 — Advanced Alert Engine

**Status**: Pending Phase 9 Approval
**Features**:
- Complex Event Processing (CEP) with compound conditions
- Multi-condition alert rules (AND/OR logic)
- Time-bound alerts ("Alert only between 09:15 and 11:00 IST")
- Portfolio-correlated alerts (signal on held stock)
- Scheduled digest alerts (EOD summary)
- Alert performance tracking (how many acted upon?)
- Alert fatigue detection and automatic throttling

**Exit Criteria**: < 100ms alert generation latency, > 99% delivery success rate in load testing

---

## Phase 11 — Deployment

**Status**: Pending Phase 10 Approval
**Deliverables**:
- Kubernetes manifests for all 18 microservices
- Helm charts with environment-specific values
- Terraform IaC for all AWS infrastructure
- ArgoCD ApplicationSets (dev/staging/production)
- CI/CD pipelines (GitHub Actions) with security scans
- TLS certificate management (AWS ACM + cert-manager)
- Secrets management (External Secrets Operator + AWS Secrets Manager)
- Multi-AZ deployment for all stateful services
- Disaster recovery runbooks

**Exit Criteria**: Staging deployment mirrors production configuration, all services healthy in staging

---

## Phase 12 — Production Readiness

**Status**: Pending Phase 11 Approval
**Deliverables**:
- Load testing: 5,000 concurrent users sustained for 30 minutes
- Chaos engineering: service failure simulation (chaos-mesh)
- Security penetration testing (external firm recommended)
- OWASP Top 10 validation
- SEBI compliance review (signal disclaimers, data handling)
- DPDPA compliance audit (data localization, consent management)
- SLA definition and SLO/SLI documentation
- Runbooks for all operational scenarios
- On-call rotation setup (PagerDuty)
- Incident response playbooks
- Performance benchmarks and baselines documented

**Exit Criteria**: Load tests pass, security audit clear, legal review complete, SLAs defined

---

## Post-Launch Roadmap (v1.x)

| Feature | Target Release |
|---|---|
| Broker Integration (Zerodha Kite) | v1.1 |
| Broker Integration (Angel One, Upstox) | v1.2 |
| Live Trading (after SEBI RA registration) | v1.3 |
| Mobile App (React Native) — iOS & Android | v1.2 |
| Options Strategy Builder | v1.4 |
| Sector Rotation Intelligence | v1.2 |
| Fundamental Screener | v1.3 |
| Social Portfolio Sharing | v1.5 |
| Multi-language support (Hindi) | v1.4 |
| White-label (B2B tenant branding) | v1.6 |
| AI Model Marketplace | v2.0 |

---

## Version Milestones

| Version | Description | Target |
|---|---|---|
| v0.1.0 | Project initialization | ✅ 2026-07-10 |
| v0.3.0 | Database + API design complete | Q3 2026 |
| v0.5.0 | All backend services operational | Q4 2026 |
| v0.7.0 | AI Engine + Dashboard complete | Q1 2027 |
| v0.9.0 | Backtesting + Paper Trading + Alerts | Q2 2027 |
| v1.0.0 | Production-ready full launch | Q3 2027 |

---

*This roadmap is a living document. It is updated at the completion of each phase.*
*Last updated: 2026-07-10*
