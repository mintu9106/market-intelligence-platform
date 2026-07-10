# Phase 3C — Physical Database Schema Design
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Status**: Pending User Approval
> **Version**: 1.0
> **Depends On**: Phase 3A.1 & 3B.1 (Domain Models & ERD) — Approved
> **Next Phase**: Phase 4 — API Specification Design

---

## Part 1: PostgreSQL Schema (Primary Relational Store)

This schema implements all core identity, billing, portfolio, strategy, scanner, and metadata entities in **PostgreSQL 16**. To comply with **Rule 1 (Multi-tenant SaaS)** and **Rule 2 (RBAC)**, Row-Level Security (RLS) is enabled on all tenant-specific tables.

### 1. Database Setup & RLS Config helper
```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Helper function to extract tenant ID from current transaction context
CREATE OR REPLACE FUNCTION get_current_tenant_id()
RETURNS UUID AS $$
BEGIN
    RETURN current_setting('app.current_tenant_id', true)::uuid;
EXCEPTION
    WHEN OTHERS THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### 2. Core Tenant & Identity Tables
```sql
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    subdomain VARCHAR(63) UNIQUE,
    custom_domain VARCHAR(255) UNIQUE,
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, DEACTIVATED
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE subscription_plans (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(63) NOT NULL UNIQUE, -- FREE, TRIAL, PRO, ENTERPRISE
    price_cents INT NOT NULL DEFAULT 0,
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    watchlist_limit INT NOT NULL DEFAULT 1,
    portfolio_limit INT NOT NULL DEFAULT 1,
    alerts_daily_limit INT NOT NULL DEFAULT 10,
    features JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tenant_subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    plan_id UUID NOT NULL REFERENCES subscription_plans(id) ON DELETE RESTRICT,
    status VARCHAR(31) NOT NULL DEFAULT 'TRIAL', -- TRIAL, ACTIVE, PAST_DUE, CANCELLED
    trial_start_at TIMESTAMP WITH TIME ZONE,
    trial_end_at TIMESTAMP WITH TIME ZONE,
    billing_cycle_start TIMESTAMP WITH TIME ZONE,
    billing_cycle_end TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_tenant_subs_tenant ON tenant_subscriptions(tenant_id);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, PENDING_MFA
    mfa_secret VARCHAR(127),
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    first_name VARCHAR(127) NOT NULL,
    last_name VARCHAR(127) NOT NULL,
    phone_number VARCHAR(31),
    avatar_url VARCHAR(511),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_profiles_tenant ON user_profiles(tenant_id);

CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- NULL means System-wide role
    name VARCHAR(63) NOT NULL, -- ADMIN, ANALYST, PREMIUM_USER, FREE_USER
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_role_name_tenant ON roles(name, COALESCE(tenant_id, '00000000-0000-0000-0000-000000000000'::uuid));

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code VARCHAR(63) NOT NULL UNIQUE, -- e.g., 'backtest:execute', 'alert_rule:create'
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE role_permissions (
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
```

### 3. API Key, Session, & Domain Event Tables
```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(127) NOT NULL,
    key_hash VARCHAR(64) NOT NULL UNIQUE, -- SHA-256 hash of key
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE',
    expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_tenant_user ON api_keys(tenant_id, user_id);

CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(64) NOT NULL UNIQUE,
    ip_address VARCHAR(45),
    user_agent TEXT,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_sessions_token ON sessions(token_hash);

CREATE TABLE domain_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- system level events can be null
    aggregate_type VARCHAR(63) NOT NULL, -- PORTFOLIO, ALERT, STRATEGY
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(127) NOT NULL, -- PORTFOLIO_CREATED, ALERT_TRIGGERED
    payload JSONB NOT NULL,
    trace_id UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_domain_events_aggregate ON domain_events(aggregate_type, aggregate_id);
```

### 4. Portfolio & Watchlist Tables
```sql
CREATE TABLE portfolios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(31) NOT NULL DEFAULT 'PAPER', -- PAPER, LIVE
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    balance_cents BIGINT NOT NULL DEFAULT 100000000, -- 10,000,000.00 INR virtual capital
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_portfolios_user ON portfolios(tenant_id, user_id);

CREATE TABLE market_symbols (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    exchange VARCHAR(15) NOT NULL, -- NSE, BSE
    ticker VARCHAR(63) NOT NULL, -- RELIANCE, NIFTY
    isin VARCHAR(12) UNIQUE,
    lot_size INT NOT NULL DEFAULT 1,
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE',
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_symbol_ticker_exchange ON market_symbols(ticker, exchange);

CREATE TABLE holdings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    quantity INT NOT NULL DEFAULT 0,
    average_cost_cents BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT check_positive_quantity CHECK (quantity >= 0)
);
CREATE UNIQUE INDEX idx_holding_portfolio_symbol ON holdings(portfolio_id, symbol_id);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    direction VARCHAR(15) NOT NULL, -- BUY, SELL
    type VARCHAR(15) NOT NULL, -- MARKET, LIMIT, STOP_LOSS, STOP_LOSS_LIMIT
    status VARCHAR(31) NOT NULL DEFAULT 'PENDING', -- PENDING, SUBMITTED, FILLED, CANCELLED, REJECTED
    quantity INT NOT NULL,
    price_cents BIGINT, -- NULL for Market Order
    trigger_price_cents BIGINT, -- For stop orders
    slippage_cents INT DEFAULT 0,
    brokerage_fee_cents INT DEFAULT 0,
    tax_charges_cents INT DEFAULT 0,
    filled_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_orders_portfolio ON orders(portfolio_id);

CREATE TABLE watchlists (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_watchlists_user ON watchlists(tenant_id, user_id);

CREATE TABLE watchlist_items (
    watchlist_id UUID NOT NULL REFERENCES watchlists(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE CASCADE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (watchlist_id, symbol_id)
);
```

### 5. Strategy, Scanners, & Backtest Tables
```sql
CREATE TABLE strategies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- NULL means System-wide strategy
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_strategy_version ON strategies(name, version, COALESCE(tenant_id, '00000000-0000-0000-0000-000000000000'::uuid));

CREATE TABLE strategy_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    strategy_id UUID NOT NULL REFERENCES strategies(id) ON DELETE CASCADE,
    status VARCHAR(31) NOT NULL DEFAULT 'RUNNING', -- RUNNING, COMPLETED, FAILED
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE backtests (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    strategy_id UUID NOT NULL REFERENCES strategies(id) ON DELETE CASCADE,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    initial_capital_cents BIGINT NOT NULL,
    status VARCHAR(31) NOT NULL DEFAULT 'QUEUED', -- QUEUED, RUNNING, COMPLETED, FAILED
    cagr_bps INT, -- Basis points
    sharpe_ratio INT, -- Multiplied by 1000 for precision
    sortino_ratio INT,
    max_drawdown_bps INT,
    report_s3_url VARCHAR(511),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE scanners (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    rules JSONB NOT NULL DEFAULT '{}', -- Technical indicators or feature levels criteria
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE scanner_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scanner_id UUID NOT NULL REFERENCES scanners(id) ON DELETE CASCADE,
    execution_time TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    duration_ms INT
);
CREATE INDEX idx_scanner_runs_time ON scanner_runs(execution_time DESC);

CREATE TABLE scanner_matches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    scanner_run_id UUID NOT NULL REFERENCES scanner_runs(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE CASCADE,
    metric_snapshots JSONB NOT NULL DEFAULT '{}', -- Stores current value of scanned indicator targets
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_scanner_matches_run ON scanner_matches(scanner_run_id);
```

### 6. AI Engine Configuration Tables
```sql
CREATE TABLE model_registries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(127) NOT NULL UNIQUE, -- E.g., 'NSE_LSTM_PRICE_DIRECTION'
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ai_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    registry_id UUID NOT NULL REFERENCES model_registries(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    feature_config JSONB NOT NULL DEFAULT '[]', -- List of feature names required
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE feature_store_versions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    version VARCHAR(63) NOT NULL UNIQUE,
    definitions JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE model_versions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES ai_models(id) ON DELETE CASCADE,
    version_string VARCHAR(63) NOT NULL, -- e.g., 'v1.4.2'
    status VARCHAR(31) NOT NULL DEFAULT 'CANDIDATE', -- CANDIDATE, CHAMPION, DEPRECATED
    feature_store_version_id UUID NOT NULL REFERENCES feature_store_versions(id) ON DELETE RESTRICT,
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    metrics JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE model_artifacts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    s3_path VARCHAR(511) NOT NULL,
    sha256_checksum VARCHAR(64) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE model_calibrations (
    model_version_id UUID PRIMARY KEY REFERENCES model_versions(id) ON DELETE CASCADE,
    calibration_method VARCHAR(63) NOT NULL, -- PLATT_SCALING, ISOTONIC_REGRESSION
    coefficients JSONB NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE prompt_libraries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(127) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    prompt_template TEXT NOT NULL,
    parameters JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_prompt_ver ON prompt_libraries(name, version);
```

### 7. Trade Signal, Alerts, & Notification Tables
```sql
CREATE TABLE trade_signals (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    direction VARCHAR(15) NOT NULL, -- BUY, SELL
    version INT NOT NULL DEFAULT 1,
    entry_range_low_cents BIGINT NOT NULL,
    entry_range_high_cents BIGINT NOT NULL,
    stop_loss_cents BIGINT NOT NULL,
    target_price_cents BIGINT NOT NULL,
    confidence_calibrated INT NOT NULL, -- 0-100
    rationale TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_trade_signals_symbol ON trade_signals(symbol_id, created_at DESC);

CREATE TABLE signal_performances (
    signal_id UUID PRIMARY KEY REFERENCES trade_signals(id) ON DELETE CASCADE,
    status VARCHAR(31) NOT NULL, -- TARGET_HIT, SL_HIT, EXPIRED, CANCELLED
    mfe_bps INT, -- Max Favorable Excursion in basis points
    mae_bps INT, -- Max Adverse Excursion
    realized_return_bps INT,
    analyzed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ai_feedbacks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    signal_id UUID NOT NULL REFERENCES trade_signals(id) ON DELETE CASCADE,
    rating INT CHECK (rating BETWEEN 1 AND 5),
    user_comment TEXT,
    followed_trade BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE alert_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE CASCADE,
    condition_type VARCHAR(63) NOT NULL, -- PRICE_CROSSES, INDICATOR_LEVEL
    condition_params JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE alert_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rule_id UUID REFERENCES alert_rules(id) ON DELETE SET NULL,
    signal_id UUID REFERENCES trade_signals(id) ON DELETE SET NULL,
    trigger_message TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notification_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channels JSONB NOT NULL DEFAULT '{"email": true, "push": true, "telegram": false, "whatsapp": false}',
    telegram_chat_id VARCHAR(127),
    whatsapp_phone_number VARCHAR(31),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notification_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    alert_event_id UUID REFERENCES alert_events(id) ON DELETE CASCADE,
    channel VARCHAR(31) NOT NULL, -- EMAIL, TELEGRAM, WHATSAPP, PUSH
    status VARCHAR(31) NOT NULL DEFAULT 'SENT', -- SENT, DELIVERED, FAILED
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 8. Risk Management, Webhooks, & Broker Tables
```sql
CREATE TABLE risk_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID REFERENCES portfolios(id) ON DELETE CASCADE,
    rule_type VARCHAR(63) NOT NULL, -- MAX_SECTOR_CONCENTRATION, MAX_DRAWDOWN_BREACH
    rule_params JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE risk_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    risk_rule_id UUID NOT NULL REFERENCES risk_rules(id) ON DELETE CASCADE,
    severity VARCHAR(15) NOT NULL DEFAULT 'WARNING', -- WARNING, CRITICAL
    description TEXT NOT NULL,
    resolved BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE webhook_subscriptions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    target_url VARCHAR(511) NOT NULL,
    secret_key VARCHAR(127) NOT NULL,
    events JSONB NOT NULL DEFAULT '[]', -- ['signal.created', 'alert.triggered']
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE broker_integrations (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(63) NOT NULL UNIQUE, -- ZERODHA, ANGEL_ONE, UPSTOX
    auth_config JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE broker_accounts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    integration_id UUID NOT NULL REFERENCES broker_integrations(id) ON DELETE RESTRICT,
    broker_client_id VARCHAR(127) NOT NULL,
    access_token_encrypted TEXT NOT NULL,
    refresh_token_encrypted TEXT,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_broker_account_user ON broker_accounts(user_id, integration_id);

CREATE TABLE broker_orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    broker_account_id UUID NOT NULL REFERENCES broker_accounts(id) ON DELETE CASCADE,
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE RESTRICT,
    external_order_id VARCHAR(127) NOT NULL UNIQUE,
    broker_status VARCHAR(63) NOT NULL,
    error_message TEXT,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 9. RLS Policies Configuration (Rule 1)
```sql
-- Enable Row Level Security on all tenant tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE domain_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;
ALTER TABLE holdings ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE watchlists ENABLE ROW LEVEL SECURITY;
ALTER TABLE alert_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE alert_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE strategies ENABLE ROW LEVEL SECURITY;
ALTER TABLE strategy_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE backtests ENABLE ROW LEVEL SECURITY;
ALTER TABLE scanners ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_feedbacks ENABLE ROW LEVEL SECURITY;
ALTER TABLE risk_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE risk_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhook_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE broker_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE broker_orders ENABLE ROW LEVEL SECURITY;

-- Dynamic isolation policies mapping
CREATE POLICY user_tenant_isolation ON users FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY profile_tenant_isolation ON user_profiles FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY billing_tenant_isolation ON tenant_subscriptions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY api_key_tenant_isolation ON api_keys FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY session_tenant_isolation ON sessions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY events_tenant_isolation ON domain_events FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY portfolios_tenant_isolation ON portfolios FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY holdings_tenant_isolation ON holdings FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY orders_tenant_isolation ON orders FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY watchlists_tenant_isolation ON watchlists FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY rules_tenant_isolation ON alert_rules FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY alerts_tenant_isolation ON alert_events FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY preferences_tenant_isolation ON notification_preferences FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY logs_tenant_isolation ON notification_logs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY strategies_tenant_isolation ON strategies FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY runs_tenant_isolation ON strategy_runs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY backtests_tenant_isolation ON backtests FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY scanners_tenant_isolation ON scanners FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY feedback_tenant_isolation ON ai_feedbacks FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY risk_rules_tenant_isolation ON risk_rules FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY risk_events_tenant_isolation ON risk_events FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY webhooks_tenant_isolation ON webhook_subscriptions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_accounts_tenant_isolation ON broker_accounts FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_orders_tenant_isolation ON broker_orders FOR ALL USING (tenant_id = get_current_tenant_id());
```

---

## Part 2: TimescaleDB Schema (Time-Series Store)

Built in **TimescaleDB** using hypertable abstractions. It holds high-throughput market ticks, OHLCV candles, computed indicators, system monitoring, and portfolio histories.

### 1. Market Ticks Table (Hypertable)
```sql
CREATE TABLE market_ticks (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    ltp_cents BIGINT NOT NULL,
    ltq INT NOT NULL,
    atp_cents BIGINT NOT NULL,
    total_volume BIGINT NOT NULL,
    open_interest BIGINT NOT NULL,
    bid_price_cents BIGINT,
    ask_price_cents BIGINT,
    bid_qty INT,
    ask_qty INT
);
-- Convert to hypertable partitioned by time (1-day chunk interval)
SELECT create_hypertable('market_ticks', 'time', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_ticks_symbol_time ON market_ticks(symbol_id, time DESC);

-- Apply compression policy (after 2 days)
ALTER TABLE market_ticks SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('market_ticks', INTERVAL '2 days');
```

### 2. OHLCV Candles Table
```sql
CREATE TABLE ohlcv_candles (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    timeframe VARCHAR(15) NOT NULL, -- 1m, 3m, 5m, 15m, 1H, 1D
    open_cents BIGINT NOT NULL,
    high_cents BIGINT NOT NULL,
    low_cents BIGINT NOT NULL,
    close_cents BIGINT NOT NULL,
    volume BIGINT NOT NULL,
    vwap_cents BIGINT NOT NULL
);
-- 1-week chunk interval
SELECT create_hypertable('ohlcv_candles', 'time', chunk_time_interval => INTERVAL '7 days');
CREATE UNIQUE INDEX idx_ohlcv_unique ON ohlcv_candles(symbol_id, timeframe, time DESC);

ALTER TABLE ohlcv_candles SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol_id, timeframe',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ohlcv_candles', INTERVAL '14 days');
```

### 3. Technical Indicators Table
```sql
CREATE TABLE technical_indicators (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    timeframe VARCHAR(15) NOT NULL,
    indicators JSONB NOT NULL -- Stores pre-computed values e.g., {"rsi": 42.1, "ema_50": 284000}
);
SELECT create_hypertable('technical_indicators', 'time', chunk_time_interval => INTERVAL '7 days');
CREATE UNIQUE INDEX idx_indicators_unique ON technical_indicators(symbol_id, timeframe, time DESC);

ALTER TABLE technical_indicators SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol_id, timeframe',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('technical_indicators', INTERVAL '14 days');
```

### 4. AI Features & Predictions Table
```sql
CREATE TABLE ai_feature_snapshots (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    feature_store_version_id UUID NOT NULL,
    features JSONB NOT NULL -- Vector features
);
SELECT create_hypertable('ai_feature_snapshots', 'time', chunk_time_interval => INTERVAL '7 days');

CREATE TABLE ai_predictions_snapshots (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    model_version_id UUID NOT NULL,
    prediction VARCHAR(15) NOT NULL, -- BUY, SELL, NEUTRAL
    probability_bps INT NOT NULL, -- Probability in bps (e.g. 8700 = 87%)
    confidence_interval_low INT NOT NULL,
    confidence_interval_high INT NOT NULL,
    shap_values JSONB NOT NULL
);
SELECT create_hypertable('ai_predictions_snapshots', 'time', chunk_time_interval => INTERVAL '7 days');
```

### 5. Model Monitoring, Drift, & Portfolio NAV Tables
```sql
CREATE TABLE model_monitoring_logs (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    model_version_id UUID NOT NULL,
    inference_latency_ms INT NOT NULL,
    cpu_utilization_pct INT NOT NULL,
    gpu_utilization_pct INT NOT NULL,
    memory_used_bytes BIGINT NOT NULL
);
SELECT create_hypertable('model_monitoring_logs', 'time', chunk_time_interval => INTERVAL '7 days');

CREATE TABLE model_drift_logs (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    model_version_id UUID NOT NULL,
    drift_type VARCHAR(31) NOT NULL, -- DATA_DRIFT, CONCEPT_DRIFT
    feature_name VARCHAR(127) NOT NULL,
    drift_score INT NOT NULL -- Multiplied by 10000
);
SELECT create_hypertable('model_drift_logs', 'time', chunk_time_interval => INTERVAL '30 days');

CREATE TABLE portfolio_nav_history (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    portfolio_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    nav_cents BIGINT NOT NULL,
    cash_cents BIGINT NOT NULL,
    holdings_value_cents BIGINT NOT NULL
);
SELECT create_hypertable('portfolio_nav_history', 'time', chunk_time_interval => INTERVAL '30 days');
```

---

## Part 3: ClickHouse Schema (Analytics OLAP Store)

ClickHouse implements our high-speed reporting database using the `MergeTree` family. Data is partitioned to isolate periods and optimized with sorting keys to accelerate aggregations.

### 1. Signal Performance Analytics
```sql
CREATE TABLE signal_performance_analytics (
    signal_id UUID,
    symbol_id UUID,
    symbol_ticker LowCardinality(String),
    strategy_id UUID,
    strategy_name LowCardinality(String),
    direction Enum8('BUY' = 1, 'SELL' = 2),
    confidence_score UInt8,
    status Enum8('TARGET_HIT' = 1, 'SL_HIT' = 2, 'EXPIRED' = 3, 'CANCELLED' = 4),
    mfe_bps Int32,
    mae_bps Int32,
    realized_return_bps Int32,
    signal_date Date,
    signal_time DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(signal_date)
ORDER BY (strategy_id, symbol_ticker, signal_date)
SETTINGS index_granularity = 8192;
```

### 2. Alert Delivery Analytics
```sql
CREATE TABLE alert_delivery_analytics (
    alert_id UUID,
    tenant_id UUID,
    user_id UUID,
    channel Enum8('EMAIL' = 1, 'TELEGRAM' = 2, 'WHATSAPP' = 3, 'PUSH' = 4),
    priority Enum8('CRITICAL' = 1, 'HIGH' = 2, 'MEDIUM' = 3, 'LOW' = 4),
    status Enum8('SENT' = 1, 'DELIVERED' = 2, 'FAILED' = 3),
    error_code LowCardinality(String),
    event_date Date,
    event_time DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (tenant_id, channel, status, event_date)
SETTINGS index_granularity = 8192;
```

### 3. User Activity Logs
```sql
CREATE TABLE user_activity_logs (
    log_id UUID,
    tenant_id UUID,
    user_id UUID,
    role LowCardinality(String),
    action LowCardinality(String), -- e.g., 'backtest_executed', 'order_placed'
    ip_address String,
    device_type LowCardinality(String),
    duration_ms UInt32,
    event_date Date,
    event_time DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (tenant_id, user_id, action, event_date)
SETTINGS index_granularity = 8192;
```

---

## Part 4: Redis Data Structures (In-Memory Key Schema)

Redis structures are designed for low-latency lookups, rate limiting, and real-time WebSocket distribution.

```
                  ┌──────────────────────────────────────────────┐
                  │                 REDIS STRUCTURES             │
                  └──────────────────────┬───────────────────────┘
                                         │
                 ┌───────────────────────┼───────────────────────┐
                 ▼                       ▼                       ▼
      ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
      │  String Key-Value   │ │      Hash Keys      │ │    Set Key-Value    │
      ├─────────────────────┤ ├─────────────────────┤ ├─────────────────────┤
      │ • sessions          │ │ • market:price      │ │ • watchlist         │
      │ • ratelimit         │ │ • tenant:config     │ │ • ws:subscriptions  │
      │ • dedup:alert       │ └─────────────────────┘ └─────────────────────┘
      └─────────────────────┘
```

### 1. Key Patterns Specification

| Key Schema | Redis Type | Fields / Value description | TTL | Purpose |
|---|---|---|---|---|
| `session:{token_hash}` | String | JSON: `{"user_id": "uuid", "tenant_id": "uuid", "role": "admin"}` | 15 minutes | Fast auth validator |
| `market:price:{symbol_id}` | Hash | `ltp_cents` (int), `volume` (int), `oi` (int), `bid_cents` (int), `ask_cents` (int), `ts` (timestamp) | None | Real-time LTP tracking |
| `watchlist:{user_id}:{id}` | Set | Set of `symbol_id` strings | None | Watchlist monitoring |
| `ratelimit:{user_id}:{epoch}` | String | Integer counter | 1 minute | API rate limits |
| `dedup:alert:{user_id}:{symbol}:{id}` | String | `1` (existence matches duplicate state) | 4 hours | Avoid user alert spam |
| `ws:sub:{socket_id}` | Set | List of subscribed topic strings (e.g., `symbol:RELIANCE`) | Session | WebSocket fan-out route mapping |
| `tenant:{tenant_id}:config` | Hash | `subscription_tier` (string), `is_active` (bool), `daily_quota` (int) | 30 minutes | SaaS profile caching |

---

## Part 5: Kafka Topic Schema & Partition Design

Enforces strict partition ordering by grouping events with transaction keys. Message schemas are managed using Avro templates.

### 1. Ingest & Processing Streams

```
  ┌─────────────────────────────────────────────────────────────────────────┐
  │                           KAFKA TOPIC FLOWS                             │
  └────────────────────────────────────┬────────────────────────────────────┘
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 ▼                       ▼                     ▼
      ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
      │  raw.market.ticks   │ │  processed.candles  │ │   signals.final     │
      ├─────────────────────┤ ├─────────────────────┤ ├─────────────────────┤
      │ Key: symbol_id      │ │ Key: symbol_id      │ │ Key: symbol_id      │
      │ Order: Per symbol   │ │ Order: Per symbol   │ │ Order: Per symbol   │
      └─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

#### Topic: `raw.market.ticks`
- **Partition Key**: `symbol_id` (Guarantees ordered tick processing per stock).
- **Payload Schema (Avro)**:
  ```json
  {
    "type": "record",
    "name": "MarketTickEvent",
    "namespace": "in.marketintelligence.marketdata",
    "fields": [
      { "name": "trace_id", "type": "string" },
      { "name": "symbol_id", "type": "string" },
      { "name": "exchange", "type": "string" },
      { "name": "ltp_cents", "type": "long" },
      { "name": "ltq", "type": "int" },
      { "name": "atp_cents", "type": "long" },
      { "name": "total_volume", "type": "long" },
      { "name": "open_interest", "type": "long" },
      { "name": "bid_price_cents", "type": ["null", "long"], "default": null },
      { "name": "ask_price_cents", "type": ["null", "long"], "default": null },
      { "name": "bid_qty", "type": ["null", "int"], "default": null },
      { "name": "ask_qty", "type": ["null", "int"], "default": null },
      { "name": "exchange_timestamp", "type": "long" },
      { "name": "ingestion_timestamp", "type": "long" }
    ]
  }
  ```

#### Topic: `processed.candles`
- **Partition Key**: `symbol_id`
- **Payload Schema (Avro)**:
  ```json
  {
    "type": "record",
    "name": "EnrichedCandleEvent",
    "namespace": "in.marketintelligence.processing",
    "fields": [
      { "name": "trace_id", "type": "string" },
      { "name": "symbol_id", "type": "string" },
      { "name": "timeframe", "type": "string" },
      { "name": "open_cents", "type": "long" },
      { "name": "high_cents", "type": "long" },
      { "name": "low_cents", "type": "long" },
      { "name": "close_cents", "type": "long" },
      { "name": "volume", "type": "long" },
      { "name": "vwap_cents", "type": "long" },
      { "name": "indicators_json", "type": "string" }, -- Serialized indicator snapshot
      { "name": "timestamp", "type": "long" }
    ]
  }
  ```

#### Topic: `signals.final`
- **Partition Key**: `symbol_id`
- **Payload Schema (Avro)**:
  ```json
  {
    "type": "record",
    "name": "FinalSignalEvent",
    "namespace": "in.marketintelligence.intelligence",
    "fields": [
      { "name": "trace_id", "type": "string" },
      { "name": "signal_id", "type": "string" },
      { "name": "symbol_id", "type": "string" },
      { "name": "direction", "type": "string" },
      { "name": "confidence_calibrated", "type": "int" },
      { "name": "entry_range_low_cents", "type": "long" },
      { "name": "entry_range_high_cents", "type": "long" },
      { "name": "stop_loss_cents", "type": "long" },
      { "name": "target_price_cents", "type": "long" },
      { "name": "rationale_text", "type": "string" },
      { "name": "shap_values_json", "type": "string" },
      { "name": "prompt_library_id", "type": "string" },
      { "name": "timestamp", "type": "long" }
    ]
  }
  ```

### 2. Alert & Distribution Streams

#### Topic: `alerts.outbound`
- **Partition Key**: `user_id` (Ensures that message delivery retries or ordering queues maintain consistency per customer).
- **Payload Schema (Avro)**:
  ```json
  {
    "type": "record",
    "name": "OutboundAlertEvent",
    "namespace": "in.marketintelligence.alerts",
    "fields": [
      { "name": "trace_id", "type": "string" },
      { "name": "alert_event_id", "type": "string" },
      { "name": "tenant_id", "type": "string" },
      { "name": "user_id", "type": "string" },
      { "name": "priority", "type": "string" },
      { "name": "headline", "type": "string" },
      { "name": "body", "type": "string" },
      { "name": "target_channels", "type": { "type": "array", "items": "string" } },
      { "name": "timestamp", "type": "long" }
    ]
  }
  ```

---

*Phase 3C — Physical Database Schema Design v1.0*
*Status: Pending User Review and Approval*
*Next Phase: Phase 4 — API Specification Design (awaiting approval)*
