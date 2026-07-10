# Phase 2B — Complete Data Flow Architecture
## AI-Powered Indian Stock Market Intelligence Platform

> **Status**: Pending User Approval
> **Version**: 1.0
> **Depends On**: Phase 2A — Architecture Modules v1.0 (Approved)
> **Next Phase**: Phase 3 — Database Design

---

## Guiding Principles for Data Flow Design

| Principle | Implementation |
|---|---|
| **Single Source of Truth** | Each data entity has exactly one authoritative source |
| **Immutability** | Raw ingested data is never mutated; transformations create new records |
| **Traceability** | Every data element carries a `trace_id` from ingestion to delivery |
| **Fail-Fast Validation** | Invalid data is rejected at the source boundary, never propagates downstream |
| **Schema Contracts** | All Kafka events are schema-validated (Avro + Schema Registry) |
| **Idempotency** | All consumers handle duplicate events without side effects |
| **Latency Budget** | Each flow has a defined maximum acceptable latency (see SLA table) |

---

## Latency Budget by Flow

| Flow | Maximum Acceptable Latency | Priority |
|---|---|---|
| Market Data → Kafka | < 50ms | Critical |
| Indicator Computation | < 500ms per candle | Critical |
| AI Prediction | < 2 seconds | High |
| Signal Generation | < 3 seconds end-to-end | High |
| Alert Generation | < 5 seconds from signal | High |
| Notification Delivery | < 10 seconds from alert | High |
| Dashboard Real-time Feed | < 1 second | High |
| Portfolio P&L Update | < 2 seconds | Medium |
| News Processing | < 60 seconds | Medium |
| Logging Pipeline | < 5 seconds (async) | Low |
| Monitoring Scrape | 15-second interval | Low |

---

## Flow 1 — Market Data Flow

### Overview
The foundational flow. All intelligence on the platform begins here. Raw price and volume data is ingested from external market feeds, normalized, validated, and published as the single canonical stream consumed by every downstream service.

### Source
- **Primary**: NSE real-time WebSocket feed (tick-by-tick data during market hours 09:00–15:30 IST)
- **Secondary**: BSE real-time feed (parallel ingestion for cross-validation)
- **Vendor APIs**: Zerodha Kite Connect, Angel One SmartAPI, Upstox, Dhan (as fallback or supplementary sources)
- **Data types ingested**:
  - Tick data: LTP (Last Traded Price), LTQ (Last Traded Quantity), ATP (Average Traded Price), Open Interest
  - Order book: Bid/Ask (Level 1 and Level 2 depth)
  - OHLC snapshots (every minute from exchange)
  - Index data: NIFTY 50, BANK NIFTY, NIFTY IT, NIFTY Pharma, etc.
  - Derivatives: Futures price, Options chain (all strikes, all expiries)
  - Circuit breaker status (Upper/Lower circuit hits)
  - Pre-open session data (09:00–09:15 IST)

### Processing
1. **Connection Management**: Persistent WebSocket connections maintained with automatic reconnection (exponential backoff). Connection health monitored via heartbeat ping every 5 seconds.
2. **Vendor Abstraction**: Each vendor's proprietary format is translated by a vendor-specific adapter into a unified internal schema (Canonical Data Model). The rest of the system never sees vendor-specific formats.
3. **Deduplication**: Tick events with identical `(symbol, timestamp, ltp, ltq)` tuples within a 100ms window are discarded. Sequence numbers tracked per vendor to detect gaps.
4. **Gap Detection**: Missing sequence numbers trigger an alert to the engineering team and a backfill request to the secondary vendor. No gaps are silently swallowed.
5. **Symbol Normalization**: Vendor-specific ticker symbols mapped to internal canonical symbol IDs (e.g., `NSE:RELIANCE-EQ` → `SYM_0001`). Symbol master refreshed daily before market open.
6. **Timestamp Normalization**: All timestamps converted to UTC and tagged with IST offset. Exchange timestamp preserved separately from ingestion timestamp.
7. **Corporate Action Adjustment**: Prices adjusted for splits, bonuses, and dividends in real-time when corporate action events are received.

### Validation
- **Schema Validation**: Every tick validated against Avro schema registered in Schema Registry. Malformed events routed to Dead Letter Queue (DLQ).
- **Price Sanity Check**: LTP must be within ±20% of previous close (circuit breaker bounds). Ticks outside this range flagged and held for manual review.
- **Timestamp Staleness**: Ticks with exchange timestamps older than 10 seconds discarded (stale data from reconnection buffer).
- **Symbol Existence Check**: Ticks for unknown symbols routed to DLQ with alert triggered.
- **Sequence Continuity**: Gaps in sequence numbers > 5 trigger vendor failover evaluation.

### Output
- **Kafka Topic**: `raw.market.ticks`
  - Partition Key: `symbol_id` (ensures all ticks for one symbol land on same partition → ordered processing)
  - Retention: 7 days
  - Replication Factor: 3
- **Event Schema**:
  ```
  MarketTickEvent {
    trace_id, symbol_id, exchange, vendor_source,
    ltp, ltq, atp, volume, oi,
    bid_price, ask_price, bid_qty, ask_qty,
    exchange_timestamp, ingestion_timestamp,
    is_circuit_hit, circuit_type (UPPER/LOWER/NONE)
  }
  ```
- **Dead Letter Queue**: `raw.market.ticks.dlq` — all rejected ticks land here for investigation

### Destination
- **Market Data Processing Service** (Kafka consumer) — primary consumer
- **TimescaleDB** — raw tick persistence (via separate archival consumer)
- **Paper Trading Engine** — live price feed for order matching

---

## Flow 2 — News Data Flow

### Overview
News and corporate announcements are a non-price signal source that feeds the AI sentiment model and provides contextual enrichment to signals. This flow is asynchronous and non-blocking — news latency is measured in seconds to minutes, not milliseconds.

### Source
- **Wire Services**: Reuters India API, Bloomberg India (if licensed), PTI feed
- **Financial Portals**: Economic Times Markets API, Moneycontrol RSS, Livemint RSS
- **Exchange Announcements**: BSE Corporate Filings API (real-time regulatory announcements), NSE announcements feed
- **SEBI**: SEBI enforcement orders and circulars (RSS)
- **Regulatory**: RBI monetary policy announcements, MPC meeting outcomes
- **Social Signals** (Phase 2+): Curated financial Twitter/X accounts (for future incorporation)

### Processing
1. **Feed Polling / Webhook**: News sources polled every 30–60 seconds (RSS) or received via webhook (if API supports push). BSE announcements received via streaming API.
2. **Content Extraction**: HTML/RSS content parsed; article body, headline, source, publication timestamp extracted.
3. **Symbol Extraction (NER)**: Named Entity Recognition model identifies company names, ticker mentions, and sector references within the article. Maps entities to canonical symbol IDs.
4. **Deduplication**: Article content hashed (MD5 of headline + first 200 chars). Duplicate hashes within 24 hours discarded.
5. **Relevance Scoring**: Articles scored for relevance to Indian equity markets (0.0–1.0). Articles below threshold 0.3 discarded before further processing.
6. **Category Classification**: News categorized as: Earnings, Merger/Acquisition, Regulatory, Macro, Sector, Management Change, Dividend, Buyback, Others.
7. **Urgency Tagging**: Breaking news (market hours, high relevance) tagged URGENT for faster pipeline priority.

### Validation
- **Source Credibility**: Only whitelisted news sources accepted. Unknown sources routed to review queue.
- **Timestamp Validation**: Articles older than 24 hours flagged as historical (not used for real-time alerts).
- **Language Check**: Non-English articles routed to translation queue (Hindi financial news — Phase 2+).
- **Symbol Association Confidence**: Articles with symbol confidence < 0.5 are processed as macro/sector news, not symbol-specific.

### Output
- **Kafka Topic**: `raw.news`
  - Partition Key: `primary_symbol_id` (if symbol-specific) or `category`
  - Retention: 30 days
- **Event Schema**:
  ```
  NewsEvent {
    trace_id, news_id, source, headline, body_excerpt,
    url, published_at, ingested_at,
    associated_symbols[ ], association_confidence,
    category, urgency_flag, relevance_score
  }
  ```

### Destination
- **AI/ML Engine** — FinBERT sentiment model consumes `raw.news`
- **Analytics Service** — tracks news volume per symbol
- **Dashboard** — news feed widget with symbol-linked headlines
- **TimescaleDB** — news archive for backtesting sentiment

---

## Flow 3 — Technical Indicator Flow

### Overview
This flow transforms raw tick streams into analytics-ready OHLCV candles and computes a comprehensive library of technical indicators across multiple timeframes. This is the most computationally intensive flow and the primary input to both the Strategy Engine and the AI Engine.

### Source
- **Input**: `raw.market.ticks` from Kafka (Kafka Streams application)
- **Supplementary**: Previous session's closing data from TimescaleDB (needed for indicators requiring historical bars, e.g., EMA 200)

### Processing

#### Stage 1 — Candle Aggregation
- **Tick Aggregation**: Ticks are grouped by `(symbol, timeframe)` windows using Kafka Streams tumbling windows.
- **Timeframes computed**: 1m, 3m, 5m, 15m, 30m, 1H, 4H, 1D
- **OHLCV Construction**: For each window — Open (first tick), High (max LTP), Low (min LTP), Close (last tick LTP), Volume (sum LTQ)
- **VWAP Integration**: Volume-Weighted Average Price computed within each candle window
- **State Management**: Incomplete (live) candle maintained in Redis with atomic updates per tick. Finalized at window close.

#### Stage 2 — Indicator Computation
For each finalized candle, the full indicator suite is computed. Computation uses TA-Lib (C library via Python bindings) for performance.

**Trend Indicators**:
- EMA (9, 21, 50, 200), SMA (20, 50, 200)
- VWAP (intraday reset at 09:15 IST)
- Supertrend (ATR multiplier: 3.0, Period: 10)
- Ichimoku Cloud (Tenkan: 9, Kijun: 26, Senkou B: 52)
- Parabolic SAR

**Momentum Indicators**:
- RSI (14), RSI (7) — dual RSI divergence detection
- MACD (12, 26, 9) — histogram + signal line
- Stochastic (14, 3, 3)
- Williams %R (14)
- CCI (20), Rate of Change (ROC)
- Money Flow Index (MFI, 14)
- Chande Momentum Oscillator

**Volatility Indicators**:
- Bollinger Bands (20, 2σ), Band Width, %B
- ATR (14), Normalized ATR
- Keltner Channels (20, 2)
- Historical Volatility (20-day)
- Donchian Channels (20)

**Volume Indicators**:
- OBV (On-Balance Volume)
- VWAP Deviation
- Volume Profile (price levels with highest volume)
- CMF (Chaikin Money Flow, 20)
- Volume Ratio (current vs 20-period average)

**Derived / Composite**:
- Support/Resistance levels (swing high/low detection over last 20 bars)
- Pivot Points (Standard, Fibonacci, Woodie's, Camarilla)
- Candlestick pattern flags (Doji, Engulfing, Hammer, Shooting Star, Morning/Evening Star) — 20+ patterns
- Multi-timeframe alignment score (e.g., 15m + 1H + 1D all in uptrend)

#### Stage 3 — Market Breadth (Index-level)
- Advance/Decline ratio (NSE 500 universe)
- New 52-week Highs / Lows count
- % of stocks above 200 EMA
- Sector-wise strength ranking
- India VIX processing

### Validation
- **Minimum Bars Check**: Indicators requiring N bars (e.g., EMA 200 requires 200 bars) only computed when sufficient history exists. Flags emitted when insufficient data.
- **NaN/Infinity Check**: Any NaN or Infinity in computed indicator values rejects the entire snapshot and logs an alert.
- **Price Spike Filter**: Candles with >10% single-bar movement flagged as potentially erroneous and held for review before indicator computation.
- **Volume Zero Guard**: Candles with zero volume (no trades in window) excluded from indicator computation to prevent misleading signals.

### Output
- **Kafka Topic**: `processed.candles`
  - Partition Key: `symbol_id`
  - Payload: Full OHLCV candle with all computed indicators
- **Kafka Topic**: `processed.market_breadth`
  - Published every minute with index-level breadth metrics
- **TimescaleDB**: All finalized candles and indicators persisted for historical access and model training
- **Redis**: Latest indicator snapshot per symbol per timeframe cached (TTL: 2 minutes)

### Destination
- **Strategy Engine** (primary consumer of `processed.candles`)
- **AI/ML Engine** (feature engineering pipeline)
- **Web Dashboard** (charting service reads from Redis cache)
- **Portfolio Intelligence Service** (beta, correlation computation)
- **Backtesting Engine** (reads from TimescaleDB for historical runs)

---

## Flow 4 — AI Prediction Flow

### Overview
The intelligence layer. Consumes enriched market data and news sentiment to produce probabilistic directional predictions with full explainability (SHAP values and human-readable rationale) for every signal.

### Source
- **Kafka**: `processed.candles` — structured indicator snapshots
- **Kafka**: `raw.news` — news events (sentiment input)
- **TimescaleDB**: Historical data queried for model training (offline pipeline)
- **Feature Store (Feast/Redis)**: Pre-computed features for online serving

### Processing

#### Stage 1 — Feature Engineering Pipeline (Online)
Runs as a Kafka Streams stateful application. For every new candle event per symbol, builds a 150+ dimensional feature vector:

**Feature Categories**:
- **Price Features** (15): Normalized returns (1m to 1D), price vs EMA ratios, distance from 52-week high/low
- **Momentum Features** (20): RSI levels and trends, MACD histogram slope, Stochastic position, ROC
- **Volatility Features** (15): ATR normalized by price, BB width, IV (for options), HV vs IV ratio
- **Volume Features** (15): Volume ratio, OBV slope, CMF value, large block trade detection
- **Trend Features** (20): EMA alignment score, Supertrend state, Ichimoku cloud position
- **Market Context Features** (25): Sector rank, India VIX level, advance/decline ratio, FII flow (daily)
- **Sentiment Features** (10): FinBERT sentiment score (24h rolling), news volume, sentiment momentum
- **Fundamental Features** (15): P/E ratio, P/B ratio, quarterly EPS growth, promoter holding %
- **Temporal Features** (15): Day of week, time of day, days to expiry (for F&O), earnings proximity

#### Stage 2 — Online Model Inference
Four models run in parallel for each symbol trigger:

**Model A — Price Direction Classifier** (LSTM/Transformer):
- Input: 60-period sequence of feature vectors
- Output: Probability distribution [BUY, SELL, NEUTRAL]
- Threshold: Only BUY/SELL predictions with probability > 0.65 proceed

**Model B — Volatility Forecaster** (GARCH-ML Hybrid):
- Input: Recent price return sequence + VIX
- Output: Expected volatility for next 1, 5, 20 sessions
- Used for: Dynamic SL/Target sizing, position sizing recommendation

**Model C — Breakout Probability Model** (Gradient Boosting — XGBoost):
- Input: Volume profile, price compression metrics, momentum features
- Output: Probability of breakout within next 3–10 candles
- Threshold: Only probabilities > 0.70 emitted as breakout signals

**Model D — Sentiment Classifier** (FinBERT — fine-tuned on Indian financial corpus):
- Input: News article text associated with symbol
- Output: Sentiment score [-1.0 to +1.0], confidence
- Updated: Every time a new NewsEvent for the symbol arrives

#### Stage 3 — XAI (Explainability Layer)
For every prediction emitted (probability > threshold):
- **SHAP Values**: TreeExplainer or DeepExplainer computes contribution of each feature to the prediction
- **Top Features Selection**: Top 5 contributing features selected
- **Human-Readable Explanation Generation**: Template-based + LLM-assisted explanation generation:
  > *"RSI(14) at 28.4 (oversold territory) contributed 34% to this BUY signal. MACD bullish crossover on 15m chart contributed 22%. Volume 2.3x above 20-day average contributed 18%. Positive news sentiment (+0.72) in last 4 hours contributed 15%. Price at strong support zone (52-week low + Fibonacci 61.8%) contributed 11%."*
- **Confidence Interval**: Prediction confidence range reported (not just point estimate)
- **Model Version Tag**: Every prediction tagged with the model version that generated it (for audit and A/B testing)

#### Stage 4 — Offline Training Pipeline (Nightly Batch — 20:00 IST)
- Fetches last 30 days of new training data from TimescaleDB
- Retrains models incrementally (transfer learning on existing weights)
- Evaluates new model vs champion model on held-out validation set
- If new model accuracy > champion by ≥2%, promotes to candidate
- MLflow logs all experiments: hyperparameters, metrics, artifacts
- Champion/Challenger deployment via feature flag (10% traffic to challenger initially)
- Full promotion after 5 trading days if challenger outperforms

### Validation
- **Feature Completeness**: Feature vector must have < 5% null values. Symbols with insufficient data excluded from prediction.
- **Prediction Confidence Gate**: Predictions below confidence threshold (0.65) suppressed — never emitted.
- **Model Staleness Check**: If a model has not been retrained in > 7 days, a critical alert is raised to the engineering team.
- **Prediction Distribution Monitor**: If prediction distribution shifts significantly (e.g., >80% BUY in a day), anomaly detected and human review triggered.
- **SHAP Consistency Check**: SHAP values must sum to approximately the model output. Inconsistencies indicate numerical errors and the prediction is suppressed.

### Output
- **Kafka Topic**: `ai.predictions`
  - Partition Key: `symbol_id`
  - Retention: 14 days
- **Event Schema**:
  ```
  AIPredictionEvent {
    trace_id, symbol_id, timeframe,
    model_name, model_version,
    prediction (BUY/SELL/NEUTRAL),
    probability, confidence_interval_low, confidence_interval_high,
    volatility_forecast_1d, volatility_forecast_5d,
    sentiment_score, sentiment_confidence,
    shap_values{ feature_name: shap_value },
    top_5_features[ { feature, contribution_pct, direction } ],
    explanation_text,
    predicted_at
  }
  ```
- **MLflow**: All training runs, metrics, and model artifacts stored
- **S3**: Model artifact binaries stored with versioned keys

### Destination
- **Signal Aggregation & Scoring Service** (primary consumer)
- **Analytics Service** (tracks prediction accuracy over time)
- **Dashboard** (AI insights panel, explanation display)

---

## Flow 5 — Risk Management Flow

### Overview
A continuous, parallel flow that monitors portfolio-level and position-level risk metrics in real-time. It is both reactive (alerts on breaches) and proactive (computes forward-looking risk metrics). This flow exists independently of signal generation — risk is always running.

### Source
- **Portfolio Positions**: User holdings from PostgreSQL (refreshed on any portfolio change event)
- **Live Prices**: Redis cache of latest LTP per symbol (updated by Market Data Processing Service)
- **Kafka**: `processed.candles` — for volatility and correlation updates
- **Kafka**: `processed.market_breadth` — macro risk context (VIX, advance/decline)
- **TimescaleDB**: Historical returns for VaR and correlation computation

### Processing

#### Position-Level Risk (per holding, real-time)
- **Unrealized P&L**: `(current_price - avg_cost_price) × quantity`
- **P&L %**: Unrealized P&L as % of invested capital
- **Stop-Loss Distance**: Current price vs user-defined SL level (% away)
- **Target Distance**: Current price vs user-defined target level
- **Day's Change**: Intraday P&L for the position

#### Portfolio-Level Risk (per user portfolio, updated every 5 minutes)
- **Portfolio Beta**: Weighted average beta of all holdings vs NIFTY 50 (rolling 60-day)
- **Portfolio VaR (95%)**: Historical simulation using 252-day return distribution. *"There is a 5% chance the portfolio loses more than X% in a single day."*
- **Portfolio VaR (99%)**: Conditional VaR (CVaR/Expected Shortfall)
- **Maximum Drawdown**: Rolling 52-week maximum drawdown from peak NAV
- **Sharpe Ratio**: Risk-adjusted return (rolling 252-day)
- **Sortino Ratio**: Downside risk-adjusted return
- **Sector Concentration**: % allocation per sector. Flag if any sector > 30% of portfolio.
- **Single Stock Concentration**: Flag if any single stock > 20% of portfolio.
- **Correlation Matrix**: Rolling 30-day return correlation between all holdings. Flag highly correlated pairs (r > 0.85) — they do not provide true diversification.
- **Liquidity Risk**: Flag illiquid holdings (daily volume < 5x position size — would take > 5 days to exit).

#### Portfolio Health Score Computation (0–100)
Composite score weighted across:
- Diversification quality (30%)
- Risk-adjusted returns (25%)
- Drawdown management (20%)
- Liquidity (15%)
- Alignment with user-defined risk profile (10%)

### Validation
- **Price Staleness Guard**: If LTP for a holding is older than 5 minutes during market hours, risk metrics marked as STALE and user notified.
- **Position Integrity Check**: Total portfolio value must be > 0 and < configured maximum (prevents data corruption from driving erroneous risk alerts).
- **VaR Reasonableness Check**: Portfolio VaR (95%) must be between 0.1% and 50%. Values outside this range indicate data issues.

### Output
- **Kafka Topic**: `risk.portfolio_metrics`
  - Partition Key: `user_id`
- **Kafka Topic**: `risk.breach_events` — emitted when thresholds are crossed:
  - SL breach (price hits user-defined stop-loss level)
  - Target hit (price reaches user-defined target)
  - Sector concentration breach (>30% in single sector)
  - VaR breach (portfolio VaR exceeds user's risk appetite)
  - Portfolio drawdown breach (max drawdown exceeds threshold)
- **PostgreSQL**: Portfolio risk snapshots stored every 5 minutes for trend analysis
- **Dashboard API**: Portfolio health score and risk metrics served via REST

### Destination
- **Alert Engine** (consumes `risk.breach_events` to generate user alerts)
- **Portfolio Intelligence Service** (enriches portfolio views with risk data)
- **Analytics Service** (tracks risk evolution over time)
- **Dashboard** (Risk panel in portfolio view)

---

## Flow 6 — Portfolio Analysis Flow

### Overview
Delivers a holistic, enriched view of the user's portfolio — combining live market data, AI signals, risk metrics, fundamental data, and tax analytics into a unified portfolio intelligence layer. This is a read-heavy, aggregation-intensive flow.

### Source
- **PostgreSQL**: User portfolio holdings (positions, transactions, cost basis)
- **Redis**: Live prices (LTP per symbol)
- **Kafka**: `signals.final` — to highlight active signals on held stocks
- **Kafka**: `risk.portfolio_metrics` — risk overlays
- **TimescaleDB**: Historical prices for performance charting
- **PostgreSQL**: Corporate actions (dividends, splits, bonuses, rights)
- **External** (future): Broker holdings sync via Broker Integration Service

### Processing

#### Real-Time Portfolio State (triggered on every price change for held stocks)
- **Live P&L Dashboard**: Unrealized P&L per position + total portfolio P&L, updated every 5 seconds during market hours
- **Today's P&L**: Intraday change in portfolio value
- **Allocation Map**: Current % allocation per stock, sector, market-cap band (Large/Mid/Small Cap)
- **Signal Overlay**: Any active BUY/SELL signals on held stocks surfaced with AI explanation

#### Performance Analytics (computed every 15 minutes + on-demand)
- **Portfolio NAV History**: Daily portfolio value plotted vs NIFTY 50 benchmark
- **CAGR**: Annualized return since portfolio inception
- **Absolute Returns**: Total return since inception, 1W, 1M, 3M, 6M, 1Y
- **Alpha**: Excess return vs NIFTY 50 benchmark
- **Best/Worst Performers**: Ranked by P&L contribution

#### Corporate Action Processing (event-driven)
- **Dividend**: Credited to portfolio cash balance; Dividend calendar shown
- **Stock Split**: Cost basis adjusted automatically; share quantity updated
- **Bonus**: New shares added; cost basis per share adjusted
- **Rights**: User notified of rights issue eligibility and pricing

#### Tax P&L Analytics (India-specific)
- **FIFO/LIFO Accounting**: Cost basis computed per FIFO method (SEBI standard for India)
- **Holding Period Tracking**: Each lot tagged with purchase date for STCG/LTCG classification
- **STCG**: Short-term capital gains (held < 12 months) — 15% tax rate
- **LTCG**: Long-term capital gains (held > 12 months) — 10% above ₹1 lakh per year
- **Tax Harvesting Suggestions**: Identify positions with unrealized losses that could be booked to offset gains
- **Brokerage + Tax Cost**: STT, exchange fees, GST, and brokerage factored into realized P&L

### Validation
- **Holdings Consistency**: Sum of all transaction quantities must match current holding quantity. Mismatch triggers reconciliation alert.
- **Cost Basis Integrity**: Cost basis must be positive. Zero or negative cost basis treated as data error.
- **Corporate Action Idempotency**: Corporate action events applied exactly once (idempotency key = `ca_event_id`).
- **Tax Period Validation**: All transactions must have valid timestamps within their respective financial years.

### Output
- **PostgreSQL**: Portfolio snapshots (daily EOD snapshot for NAV history)
- **Kafka**: `portfolio.snapshot_events` — published on significant portfolio changes
- **Kafka**: `risk.breach_events` — forwarded from Risk Management Flow
- **REST API responses**: Served to Dashboard and Mobile App (via cache-aside pattern)
- **PDF Reports**: Tax P&L reports generated on demand, stored in S3

### Destination
- **Dashboard** (Portfolio Manager UI — primary consumer)
- **Mobile App** (Portfolio summary + P&L widget)
- **Alert Engine** (portfolio events feed into alert rules)
- **Analytics Service** (tracks portfolio performance trends across users)

---

## Flow 7 — Alert Generation Flow

### Overview
The decision layer that determines *who* gets alerted, *about what*, *when*, and *at what priority*. This flow is a Complex Event Processing (CEP) engine that evaluates a user's personal alert rules against a continuous stream of market and platform events.

### Source
- **Kafka**: `signals.final` — trade signals from Signal Aggregation Service
- **Kafka**: `risk.breach_events` — SL hits, target hits, VaR breaches
- **Kafka**: `processed.candles` — price-level alert evaluation (e.g., "Alert me when RELIANCE crosses ₹2,900")
- **Kafka**: `raw.news` — news-triggered alerts (e.g., "Alert me on any news about TATA MOTORS")
- **PostgreSQL**: User-defined alert rules (watchlists, price alerts, indicator alerts, signal alerts)
- **Redis**: Alert deduplication state (active alert window per user per symbol)

### Processing

#### Step 1 — Alert Rule Cache Warm-up
- All active alert rules for all users loaded into Redis at service startup
- Rules updated in Redis on any user configuration change (event-driven cache invalidation)
- Redis holds: `alert_rules:{user_id}` → list of active rules with conditions

#### Step 2 — Event-to-Rule Matching (CEP)
For every incoming event, the Alert Engine evaluates it against all matching rules:
- **Price Alert**: `if (ltp >= threshold AND direction == CROSS_ABOVE) → trigger`
- **Indicator Alert**: `if (rsi_14 <= 30 AND timeframe == '15m') → trigger`
- **Signal Alert**: `if (signal.symbol in user.watchlist AND signal.grade in ['A','B'] AND signal.direction == 'BUY') → trigger`
- **Portfolio Alert**: `if (portfolio.event_type == 'SL_HIT' AND position.symbol == alert.symbol) → trigger`
- **News Alert**: `if (news.symbol in user.watchlist AND news.sentiment <= -0.5) → trigger`

#### Step 3 — Deduplication
- Before emitting any alert, check Redis key: `dedup:{user_id}:{symbol}:{alert_type}:{condition_hash}`
- If key exists (TTL not expired), alert is suppressed (no spam)
- Deduplication windows by alert type:
  - Price alerts: 15-minute window (same condition won't re-fire for 15 minutes)
  - Signal alerts: 4-hour window per signal type per symbol
  - News alerts: 2-hour window per symbol
  - SL/Target alerts: Fire once, then deactivated until user re-sets

#### Step 4 — Priority Scoring
Every alert assigned a priority:
- **CRITICAL**: SL breach, circuit hit, extreme VaR breach
- **HIGH**: Grade-A signal, 52-week high/low break, earnings surprise
- **MEDIUM**: Grade-B/C signal, price alert triggered, indicator condition met
- **LOW**: News alerts, portfolio concentration warning, daily digest

#### Step 5 — Personalization & Enrichment
- Alert message enriched with: current price, signal details, AI explanation snippet, risk/reward ratio
- Message length optimized per delivery channel (Telegram: rich with formatting, WhatsApp: concise, Email: full detail)
- Tenant subscription tier check: Free tier users get max 5 alerts/day; Pro: 50/day; Enterprise: unlimited

#### Step 6 — Quota Enforcement
- Check user's daily alert count against subscription tier limit
- If limit reached: alert suppressed, user notified via in-app notification only (not external channels)
- Resets at 00:00 IST daily

### Validation
- **User Active Check**: Alerts only generated for users with active subscriptions (not suspended/expired)
- **Channel Opt-in Verification**: Alert only sent to channels the user has explicitly opted into
- **Alert Rule Validity**: Rules with invalid conditions (e.g., references to delisted symbol) cleaned up and user notified
- **Symbol Tradability**: Alerts suppressed for symbols in trading halt or F&O ban period (with notification to user)

### Output
- **Kafka Topic**: `alerts.outbound`
  - Partition Key: `user_id`
  - Retention: 7 days
- **Event Schema**:
  ```
  AlertEvent {
    trace_id, alert_id, user_id, tenant_id,
    alert_type, priority,
    symbol_id, symbol_name,
    trigger_event_type, trigger_event_id,
    headline_message, detail_message,
    delivery_channels[ ] (TELEGRAM/WHATSAPP/EMAIL/PUSH/IN_APP),
    enrichment{ price, signal_grade, ai_explanation, risk_reward },
    generated_at, expires_at
  }
  ```
- **PostgreSQL**: Alert history persisted (user can view past alerts in dashboard)

### Destination
- **Notification Delivery Service** (primary consumer of `alerts.outbound`)
- **Dashboard** (in-app notification center reads from PostgreSQL alert history)
- **Analytics Service** (alert delivery rate tracking)

---

## Flow 8 — Notification Flow

### Overview
The last-mile delivery layer. Receives AlertEvents and reliably delivers them across multiple channels with retry logic, receipt tracking, and channel failover. This flow is purely operational — it makes no business logic decisions about what to send; that was decided by the Alert Engine.

### Source
- **Kafka**: `alerts.outbound` (sole input)

### Processing

#### Step 1 — Channel Routing
Each AlertEvent specifies `delivery_channels[]`. The Notification Service routes to each channel in parallel using separate worker queues:
- `queue.notification.telegram`
- `queue.notification.whatsapp`
- `queue.notification.email`
- `queue.notification.push`
- `queue.notification.inapp`

#### Step 2 — Message Template Rendering
Channel-specific message formatting:

**Telegram**:
```
🚨 GRADE-A BUY SIGNAL — RELIANCE (NSE)

📊 Entry: ₹2,847 | SL: ₹2,780 | Target: ₹2,970
⚡ Confidence: 87% | R:R = 1:2.1
🤖 AI Reason: RSI oversold (28.4) + MACD crossover + 2.3x volume

Strategy: EMA Crossover + Momentum Confluence
Timeframe: 15 Minutes

⚠️ For information only. Not investment advice.
```

**WhatsApp**: Plain text version (no markdown), concise (< 200 chars headline)

**Email**: Full HTML template with equity chart thumbnail, detailed SHAP explanation, portfolio impact analysis

**Push Notification**: 80-char headline, symbol name, signal direction

**In-App**: Full rich JSON payload rendered in dashboard notification panel

#### Step 3 — Rate Limit Check (per channel)
- **WhatsApp**: Meta enforces 1,000 unique conversations/day per number (Business API). Rate limiter enforced.
- **Telegram**: Bot API limit: 30 messages/second globally, 20 messages/minute per chat. Enforced via token bucket.
- **Email**: SendGrid daily limit per plan. Enforced via Redis counter.
- **Push**: FCM/APNs: No hard limit, but > 5 push/hour per device considered aggressive — soft limit enforced.

#### Step 4 — Delivery Attempt
External API call made with:
- Timeout: 5 seconds
- Retry on failure: 3 attempts with exponential backoff (1s, 4s, 16s)
- Circuit Breaker: If > 5 consecutive failures for a channel, circuit opens for 60 seconds (prevent cascade)

#### Step 5 — Channel Failover
If primary channel fails after 3 retries:
- Telegram failure → fallback to Email
- WhatsApp failure → fallback to Email
- Push failure → fallback to In-App only
- Email failure → In-App only + engineering alert

#### Step 6 — Delivery Receipt Tracking
- Success: `status=DELIVERED`, external message ID recorded
- Failure (permanent): `status=FAILED`, reason logged
- Telegram: Read receipts not available (fire-and-forget)
- Email: Open tracking via pixel (optional, user consent required)
- Push: Delivery confirmation from FCM/APNs

### Validation
- **User Opt-In**: Before any external delivery, verify user has opted into the specific channel. Non-opted channels silently skipped.
- **Phone/Email Validity**: Phone number format validated for WhatsApp (E.164). Email format validated. Invalid contacts trigger user profile update request.
- **Alert Expiry**: If `AlertEvent.expires_at` has passed (e.g., intraday signal alert received after market close), delivery suppressed.
- **User Account Active**: Check user account not suspended before delivery.

### Output
- **External Delivery**: Notifications sent to Telegram, WhatsApp, Email, FCM, APNs
- **PostgreSQL**: Delivery log record per alert per channel (status, timestamp, external_message_id)
- **Kafka**: `notifications.delivery_receipts` — delivery status events for analytics

### Destination
- **End Users**: Telegram, WhatsApp, Email inbox, Mobile push, Browser push
- **Analytics Service**: Delivery success rate per channel per tenant
- **Dashboard**: Notification delivery status visible in user's alert history

---

## Flow 9 — Dashboard Data Flow

### Overview
Covers how data reaches the user's browser or mobile app, across both REST (request-response) and WebSocket (real-time push) paths. This flow is read-dominated and heavily cached.

### Source
All data served to the dashboard originates from platform databases and caches:
- **Redis**: Live prices, indicator snapshots, active signals (primary read path — sub-millisecond)
- **PostgreSQL**: User data, portfolio, alert rules, signal history (for non-real-time data)
- **TimescaleDB**: Historical OHLCV data for charting (on-demand queries)
- **ClickHouse**: Analytics queries (performance metrics, signal accuracy stats)
- **Kafka → WebSocket Bridge**: Real-time events pushed directly to connected users

### Processing

#### REST Request Path (Page Load / On-Demand Data)
```
User Request
→ API Gateway (JWT validation + tenant extraction + rate limit)
→ Cache Check (Redis):
    HIT  → Return cached response immediately
    MISS → Query authoritative database → Cache result → Return response
```

**Cache Strategy by Endpoint**:
| Data | Cache TTL | Source |
|---|---|---|
| Live price / LTP | 1 second | Redis (always fresh) |
| Indicator snapshot | 1 minute | Redis |
| Signal feed | 2 minutes | Redis |
| Portfolio P&L | 5 seconds | Redis |
| Historical OHLCV (charting) | 1 hour | Redis (LRU) |
| User profile / settings | 15 minutes | Redis |
| Subscription / plan details | 30 minutes | Redis |
| News feed | 5 minutes | Redis |

**Tenant Isolation Enforcement**:
- Every database query includes `WHERE tenant_id = :tenant_id AND user_id = :user_id`
- Redis keys namespaced: `{tenant_id}:{user_id}:{data_key}`
- No cross-tenant data leakage possible by design

#### WebSocket Real-Time Path (Live Dashboard)
```
User connects via WebSocket → API Gateway upgrades connection
→ WebSocket Server authenticates (JWT in query param or initial message)
→ User subscribes to channels:
    - {user_id}.portfolio.pnl       (live portfolio P&L)
    - {symbol_id}.price             (live price for watchlist symbols)
    - {symbol_id}.signal            (new signals for watchlisted symbols)
    - {user_id}.alerts              (real-time alert delivery)
    - market.breadth                (market-wide breadth updates)
→ WebSocket Server subscribes to corresponding Redis Pub/Sub channels
→ Market Data Processing Service publishes to Redis Pub/Sub
→ WebSocket Server fans out to all subscribed users
```

**Subscription Management**:
- Free tier: Max 10 symbols in real-time watchlist
- Pro tier: Max 100 symbols
- Enterprise tier: Unlimited

**Heartbeat**: WebSocket ping/pong every 30 seconds. Connections silent for > 90 seconds terminated and client expected to reconnect.

#### Mobile-Specific Optimizations
- Response payloads compressed (gzip/brotli)
- Paginated responses (20 items default page size)
- Delta updates via WebSocket (only changed fields, not full snapshot)
- Offline cache: Last-known portfolio state cached locally (React Native AsyncStorage)

### Validation
- **JWT Expiry**: Expired tokens rejected at API Gateway; 401 returned; client refreshes token silently
- **Data Freshness Indicator**: If live price is older than 30 seconds (market hours), UI shows STALE badge
- **Tenant Boundary**: Every query double-validated at both API Gateway and service layer (defense in depth)
- **WebSocket Connection Limit**: Max 3 concurrent WebSocket connections per user (prevent resource abuse)

### Output
- **REST Responses**: JSON responses to web and mobile clients
- **WebSocket Events**: Real-time pushed events to subscribed clients
- **CDN**: Static assets (JS/CSS/images) served from CloudFront edge nodes globally

### Destination
- **Web Dashboard** (Next.js client)
- **Mobile App** (React Native client)
- **External API consumers** (Enterprise tenants using platform API)

---

## Flow 10 — Logging Flow

### Overview
Every service emits structured, machine-readable logs that are centrally aggregated, enriched, indexed, and made searchable. This is an asynchronous, non-blocking flow — logging must never impact application performance.

### Source
- **All 18 platform microservices**: Emit structured JSON logs to stdout (container standard)
- **API Gateway**: Access logs (every request: method, path, latency, status code, user_id, tenant_id)
- **Kafka**: Consumer lag metrics, producer error logs
- **Kubernetes**: Pod lifecycle events (start, crash, eviction)
- **Database**: Slow query logs (PostgreSQL: queries > 100ms), connection pool events

### Processing

#### Log Collection
- **Filebeat** deployed as a DaemonSet on every Kubernetes node
- Tails stdout from all containers on the node
- Adds Kubernetes metadata: pod name, namespace, node name, container name

#### Log Enrichment (Logstash)
```
Raw Log JSON
→ Parse trace_id, span_id (if present)
→ Add: environment (production/staging), region, cluster_name
→ PII Scrubbing: Mask phone numbers, email addresses, API keys (regex replacement)
→ Log Level Normalization: Map all formats to standard DEBUG/INFO/WARN/ERROR/FATAL
→ Timestamp Normalization: Ensure all timestamps in UTC
→ Index routing: Route ERROR/FATAL to high-priority index for faster alerting
```

#### Indexing (Elasticsearch)
- Index naming: `logs-{service}-{YYYY.MM.DD}`
- Index Lifecycle Management (ILM):
  - Hot (0–7 days): Full read/write, SSD storage
  - Warm (7–30 days): Read-only, HDD storage, smaller shards
  - Cold (30–365 days): Read-only, S3-backed (Searchable Snapshots)
  - Delete (> 365 days): Purged (DPDPA compliance)

#### Distributed Trace Correlation
- Every log line that carries a `trace_id` links back to the full distributed trace in Jaeger
- Kibana shows: "View full trace" link for any log event with a trace_id
- This enables: "Show me all logs across all services for this specific API request"

### Validation
- **Schema Validation**: Logstash rejects malformed JSON logs (logs must be valid JSON — no free-text allowed)
- **PII Audit**: Periodic scan of log indices for PII patterns. Any leaked PII triggers automated redaction + security incident
- **Log Volume Anomaly**: If a service emits > 10x its baseline log volume in 5 minutes → alert to engineering (possible runaway logging bug or attack)

### Output
- **Elasticsearch**: Indexed, searchable log store
- **S3**: Cold archive (Searchable Snapshots after 30 days)
- **Alertmanager**: ERROR/FATAL log rates feed into alerting rules

### Destination
- **Kibana**: Engineering team searches logs during incident investigation
- **Grafana**: Log-based metrics (error rate panels)
- **PagerDuty**: High error rate or FATAL log bursts trigger on-call alerts
- **S3**: Long-term compliance archive

---

## Flow 11 — Monitoring Flow

### Overview
Captures all platform operational metrics — infrastructure, application, and business-level — and provides real-time dashboards, automated alerting, and anomaly detection for the engineering and product teams.

### Source
- **Application Metrics**: Every service exposes a `/metrics` endpoint in Prometheus format:
  - HTTP request count, latency (histogram: P50, P95, P99), error rate
  - Custom business metrics: signals generated/hour, alerts delivered/minute, AI inference latency
  - Database connection pool utilization, query latency
- **Infrastructure Metrics**: Kubernetes Node Exporter on every node: CPU, memory, disk I/O, network
- **Kafka Metrics**: JMX exporter: consumer lag per topic/partition, producer throughput, broker health
- **Database Metrics**: PostgreSQL exporter: active connections, replication lag, query throughput; TimescaleDB chunk health
- **Redis Metrics**: Redis exporter: memory usage, hit rate, connected clients, command latency
- **External Dependency Metrics**: HTTP probes to external APIs (NSE, vendor APIs) — availability and latency

### Processing

#### Metric Scraping (Prometheus — Pull Model)
- Prometheus scrapes all `/metrics` endpoints every 15 seconds
- Service discovery via Kubernetes API (all pods with label `monitoring=true` auto-discovered)
- Metrics stored in Prometheus TSDB with 30-day retention
- Remote write to Thanos (long-term metric storage — 1 year retention) or VictoriaMetrics

#### Alert Rule Evaluation (Alertmanager)
Prometheus evaluates alert rules continuously. Example rules:

```
API_HighErrorRate:   5xx rate > 1% for 2 minutes → CRITICAL
Kafka_ConsumerLag:  Consumer lag > 10,000 msgs for 5 minutes → WARNING
AI_InferenceLatency: P99 inference latency > 5s for 3 minutes → WARNING
DB_ConnectionsHigh:  Active DB connections > 80% of pool → WARNING
Market_DataStale:    No new tick received in 60s during market hours → CRITICAL
Alert_DeliveryLow:   Alert delivery success rate < 90% for 5 minutes → HIGH
```

#### Alerting Pipeline
```
Alertmanager receives fired alert
→ Deduplicate (group similar alerts into 1 notification)
→ Route by severity:
    CRITICAL → PagerDuty (on-call engineer woken up)
    WARNING  → Slack #monitoring channel
    INFO     → Slack #platform-health channel
→ Silence active incidents (prevent alert storms during known maintenance)
```

#### Business KPI Tracking
Separate Grafana dashboards for product/business metrics (read from ClickHouse, not Prometheus):
- Daily/Weekly Active Users
- Signals generated vs delivered vs acted upon
- Alert delivery rate per channel
- Subscription tier distribution
- Model prediction accuracy trend (rolling 30-day)
- API response time by endpoint (P95)

#### Anomaly Detection
- Grafana ML-based anomaly detection on key metrics (Grafana ML plugin)
- Detects: Unusual traffic patterns, unexpected signal volume drops, API error rate spikes
- Alerts before threshold breaches when trend is clearly heading toward a breach

### Validation
- **Metric Freshness**: If Prometheus hasn't successfully scraped a target in > 60 seconds, alert fired (target down)
- **Alertmanager Health**: Watchdog alert — a synthetic ALWAYS_FIRING alert that should always be received. If Alertmanager stops forwarding this, it means the alerting pipeline itself is broken.
- **Dashboard Data Freshness**: Grafana panels show "last updated" timestamp. Stale panels highlighted automatically.

### Output
- **Prometheus TSDB**: Metric time-series (30-day hot retention)
- **Thanos/VictoriaMetrics**: Long-term metric storage (1 year)
- **Grafana Dashboards**: Visual operational command center
- **PagerDuty**: On-call incident management
- **Slack**: Team-wide visibility of platform health

### Destination
- **Engineering Team**: Operational dashboards and on-call alerts
- **Product Team**: Business KPI dashboards (read-only Grafana access)
- **Management**: Weekly automated PDF reports of platform health KPIs
- **Incident Postmortems**: Historical metric data used for root cause analysis

---

## Master ASCII Data Flow Diagram

```
╔═══════════════════════════════════════════════════════════════════════════════════════════════════╗
║              COMPLETE DATA FLOW — AI-POWERED INDIAN STOCK MARKET INTELLIGENCE PLATFORM           ║
╚═══════════════════════════════════════════════════════════════════════════════════════════════════╝

 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                                  EXTERNAL DATA SOURCES                                      │
 │  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐  │
 │  │ NSE/BSE     │  │ Data Vendors │  │ News APIs    │  │ BSE/NSE  │  │ Fundamental Data │  │
 │  │ WebSocket   │  │ Kite/Upstox/ │  │ Reuters/ET/  │  │ Corp.    │  │ Screener/CMIE    │  │
 │  │ Tick Feed   │  │ Angel/Dhan   │  │ Moneycontrol │  │ Announce │  │ (Quarterly)      │  │
 │  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘  └────┬─────┘  └────────┬─────────┘  │
 └─────────┼────────────────┼─────────────────┼───────────────┼─────────────────┼─────────────┘
           │  (1) Market    │                  │  (2) News     │  Corporate      │ Fundamental
           │  Data Flow     │                  │  Data Flow    │  Actions        │ Data Flow
           ▼                ▼                  ▼               ▼                 ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────────┐
 │                              INGESTION LAYER                                                │
 │  ┌──────────────────────────────────────────┐    ┌──────────────────────────────────────┐  │
 │  │       Market Data Ingestion Service      │    │     News & Fundamentals Ingestion    │  │
 │  │  • Vendor adapters (canonical model)     │    │  • Feed polling / webhook            │  │
 │  │  • Deduplication, gap detection          │    │  • NER symbol extraction             │  │
 │  │  • Schema validation (Avro)              │    │  • Relevance scoring                 │  │
 │  │  • Circuit breaker detection             │    │  • Deduplication (content hash)      │  │
 │  └──────────────────────┬───────────────────┘    └─────────────────┬────────────────────┘  │
 └─────────────────────────┼───────────────────────────────────────────┼────────────────────┘
                           │                                           │
                           ▼                                           ▼
                ┌──────────────────────┐              ┌───────────────────────────┐
                │ Kafka:               │              │ Kafka:                    │
                │ raw.market.ticks     │              │ raw.news                  │
                │ (partitioned by      │              │ raw.fundamentals          │
                │  symbol_id)          │              └──────────────┬────────────┘
                └──────────┬───────────┘                             │
                           │                                         │
           ┌───────────────┼─────────────────────────────────────────┤
           │               │                                         │
           ▼               ▼                                         ▼
 ┌─────────────────────────────────────────────┐     ┌──────────────────────────────────────┐
 │         PROCESSING LAYER            (3)     │     │       AI / ML ENGINE         (4)     │
 │  ┌──────────────────────────────────────┐   │     │                                      │
 │  │   Market Data Processing Service    │   │     │  ┌────────────────────────────────┐  │
 │  │  • Tick → OHLCV aggregation         │   │     │  │  Feature Engineering Pipeline  │  │
 │  │    (1m,3m,5m,15m,30m,1H,4H,1D)     │   │     │  │  • 150+ features per symbol    │  │
 │  │  • 50+ Technical Indicators         │   │     │  │  • Price + Volume + Sentiment  │  │
 │  │  • Pattern detection                │   │     │  │  • Fundamental + Macro         │  │
 │  │  • Market breadth computation       │   │     │  └──────────────┬─────────────────┘  │
 │  └──────────────────┬───────────────────┘   │     │               │                     │
 │                     │                       │     │  ┌────────────▼─────────────────┐   │
 │  Kafka: processed.candles                   │────▶│  │  Parallel Model Inference    │   │
 │  Kafka: processed.indicators                │     │  │  A: Direction (LSTM)         │   │
 │  Kafka: processed.market_breadth            │     │  │  B: Volatility (GARCH+ML)    │   │
 │  TimescaleDB: historical persistence        │     │  │  C: Breakout (XGBoost)       │   │
 │  Redis: latest snapshot cache              │     │  │  D: Sentiment (FinBERT)      │   │
 └─────────────────────┬───────────────────────┘     │  └──────────────┬─────────────────┘  │
                       │                             │               │                     │
                       │                             │  ┌────────────▼─────────────────┐   │
                       │                             │  │  XAI Layer (SHAP)            │   │
                       │                             │  │  • Feature contributions     │   │
                       │                             │  │  • Human-readable text       │   │
                       │                             │  │  • Confidence intervals      │   │
                       │                             │  └──────────────┬─────────────────┘  │
                       │                             └─────────────────┼────────────────────┘
                       │                                               │
                       │                             Kafka: ai.predictions
                       │                                               │
                       ▼                                               ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────┐
 │                         INTELLIGENCE LAYER                                               │
 │                                                                                         │
 │  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
 │  │              Strategy Engine                                                      │  │
 │  │  • Evaluates 15+ strategies across 2,000+ symbols                                │  │
 │  │  • Multi-timeframe confirmation                                                   │  │
 │  │  • Rule-based signal scoring                                                      │  │
 │  │  Kafka: signals.raw                                                               │  │
 │  └──────────────────────────────────────────────────────────────────────────────────┘  │
 │                              │                                                          │
 │                              ▼  ◀──────────────── Kafka: ai.predictions               │
 │  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
 │  │              Signal Aggregation & Scoring Service                                 │  │
 │  │  • Fuse rule-based + AI predictions                                              │  │
 │  │  • Grade assignment (A/B/C)                                                      │  │
 │  │  • SL + Target + R:R computation                                                 │  │
 │  │  • Signal deduplication                                                          │  │
 │  │  Kafka: signals.final                                                             │  │
 │  └──────────────────────────────────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────────────────────────────────┘
                              │
                              │  signals.final
              ┌───────────────┼────────────────────┐
              │               │                    │
              ▼               ▼                    ▼
 ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────────────────────────┐
 │  (5) RISK MGMT  │  │ (6) PORTFOLIO   │  │      (7) ALERT GENERATION LAYER      │
 │  FLOW           │  │ ANALYSIS FLOW   │  │                                      │
 │                 │  │                 │  │  ┌────────────────────────────────┐  │
 │ • VaR / Beta    │  │ • Real-time P&L │  │  │         Alert Engine           │  │
 │ • Sharpe Ratio  │  │ • Tax P&L       │  │  │  • CEP rule matching           │  │
 │ • Max Drawdown  │  │ • Corp actions  │  │  │  • Deduplication (Redis)       │  │
 │ • Concentration │  │ • Benchmark     │  │  │  • Priority scoring            │  │
 │ • SL/Target hit │  │ • Rebalance     │  │  │  • Quota enforcement           │  │
 │                 │  │   suggestion    │  │  └────────────────┬───────────────┘  │
 │ Kafka:          │  │                 │  │  Kafka: alerts.outbound             │
 │ risk.breach     │──▶                 │  └────────────────────┼────────────────┘
 └─────────────────┘  └────────┬────────┘                       │
                               │                                │
                               │                                ▼
                               │           ┌────────────────────────────────────────────┐
                               │           │     (8) NOTIFICATION DELIVERY LAYER        │
                               │           │                                            │
                               │           │  ┌──────────────────────────────────────┐ │
                               │           │  │  Notification Delivery Service       │ │
                               │           │  │  • Channel routing                   │ │
                               │           │  │  • Template rendering per channel    │ │
                               │           │  │  • Rate limiting (WhatsApp/TG/Email) │ │
                               │           │  │  • Retry + exponential backoff       │ │
                               │           │  │  • Channel failover                  │ │
                               │           │  │  • Delivery receipt tracking         │ │
                               │           │  └───────┬──────┬──────┬──────┬─────────┘ │
                               │           │          │      │      │      │           │
                               │           │       Telegram  WA   Email  Push          │
                               │           │       Bot API   API  SG/SES FCM/APNs     │
                               │           └────────────────────────────────────────────┘
                               │
                               ▼
 ┌─────────────────────────────────────────────────────────────────────────────────────────┐
 │                      (9) DASHBOARD DATA FLOW                                            │
 │                                                                                         │
 │  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
 │  │                          API GATEWAY                                              │  │
 │  │   JWT Validation → Tenant Extraction → Rate Limiting → Service Routing           │  │
 │  └──────────────────────────────────────────────────────────────────────────────────┘  │
 │        │  REST (Cache-aside pattern)              │  WebSocket (Real-time push)         │
 │        ▼                                          ▼                                     │
 │  ┌──────────────────────┐               ┌────────────────────────────────────────────┐ │
 │  │  Redis Cache Layer   │               │  WebSocket Server + Redis Pub/Sub          │ │
 │  │  HIT → return fast   │               │  • User subscribes to symbol channels      │ │
 │  │  MISS → DB → cache   │               │  • Live price, signal, P&L, alert push     │ │
 │  └──────────────────────┘               └────────────────────────────────────────────┘ │
 │        │                                          │                                     │
 │        ▼                                          ▼                                     │
 │  ┌─────────────────────────────────────────────────────────────────────────────────┐   │
 │  │             WEB DASHBOARD (Next.js)     MOBILE APP (React Native)               │   │
 │  └─────────────────────────────────────────────────────────────────────────────────┘   │
 └─────────────────────────────────────────────────────────────────────────────────────────┘

 ┌─────────────────────────────────────────────────────────────────────────────────────────┐
 │               (10) LOGGING FLOW                  (11) MONITORING FLOW                  │
 │                                                                                         │
 │  All Services → Structured JSON stdout           All Services → /metrics (Prometheus)   │
 │        │                                                  │                             │
 │  Filebeat (DaemonSet) → Logstash (enrich)        Prometheus → Scrape every 15s          │
 │        │                                                  │                             │
 │  Elasticsearch (index)   S3 (cold archive)        Alertmanager → PagerDuty / Slack      │
 │        │                                                  │                             │
 │       Kibana (search + dashboards)               Grafana (dashboards + anomaly detect)  │
 └─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Contract Summary

| Kafka Topic | Schema | Partition Key | Retention | Consumers |
|---|---|---|---|---|
| `raw.market.ticks` | MarketTickEvent (Avro) | symbol_id | 7 days | MDP Service, TimescaleDB archiver, Paper Trading |
| `raw.news` | NewsEvent (Avro) | category | 30 days | AI Engine, Analytics, Dashboard |
| `raw.fundamentals` | FundamentalEvent (Avro) | symbol_id | 90 days | AI Engine, Portfolio Service |
| `processed.candles` | EnrichedCandleEvent (Avro) | symbol_id | 3 days | Strategy Engine, AI Engine, Dashboard |
| `processed.indicators` | IndicatorSnapshotEvent (Avro) | symbol_id | 3 days | Alert Engine, Dashboard |
| `processed.market_breadth` | MarketBreadthEvent (Avro) | partition_0 | 3 days | Dashboard, AI Engine |
| `signals.raw` | RawSignalEvent (Avro) | symbol_id | 7 days | Signal Aggregation Service |
| `ai.predictions` | AIPredictionEvent (Avro) | symbol_id | 14 days | Signal Aggregation, Analytics, Dashboard |
| `signals.final` | FinalSignalEvent (Avro) | symbol_id | 14 days | Alert Engine, Portfolio Service, Analytics |
| `risk.portfolio_metrics` | PortfolioRiskEvent (Avro) | user_id | 7 days | Portfolio Service, Dashboard |
| `risk.breach_events` | RiskBreachEvent (Avro) | user_id | 7 days | Alert Engine |
| `alerts.outbound` | AlertEvent (Avro) | user_id | 7 days | Notification Delivery Service |
| `notifications.delivery_receipts` | DeliveryReceiptEvent (Avro) | user_id | 30 days | Analytics Service |

---

## Flow Dependency Map

```
raw.market.ticks
    └─▶ processed.candles
            ├─▶ signals.raw
            │       └─▶ signals.final
            │               ├─▶ alerts.outbound
            │               │       └─▶ [Delivery Channels]
            │               └─▶ [Dashboard]
            └─▶ ai.predictions
                    └─▶ signals.final (merged)

raw.news
    └─▶ ai.predictions (sentiment input)
    └─▶ [Dashboard news feed]

risk.breach_events
    └─▶ alerts.outbound
```

---

## Open Questions Before Phase 3

> [!IMPORTANT]
> **Q1 — Schema Registry**: Confluent Schema Registry (managed) or open-source Apicurio Registry? This affects Kafka schema enforcement approach.

> [!IMPORTANT]
> **Q2 — TimescaleDB Hosting**: Self-hosted TimescaleDB on Kubernetes, or Timescale Cloud (managed)? Managed reduces ops burden but increases cost.

> [!NOTE]
> **Q3 — Historical Data Bootstrap**: How many years of historical OHLCV data should be pre-loaded at launch? (3 years minimum for AI training, 10 years recommended for backtesting quality)

> [!NOTE]
> **Q4 — News Data Licensing**: Reuters/Bloomberg data requires commercial licensing. Should Phase 1 start with free sources (RSS feeds, BSE API) and add paid sources later?

> [!NOTE]
> **Q5 — WhatsApp Template Messages**: WhatsApp Business API requires pre-approved message templates for alerts. Have you initiated the Meta Business Verification process?

---

*Phase 2B — Data Flow Architecture v1.0*
*Status: Pending User Approval*
*Next: Phase 3 — Database Design (awaiting approval)*
