# Phase 3C — Physical Database Schema Design
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Status**: Pending User Approval
> **Version**: 1.1 (Extended Enterprise-Grade Spec)
> **Depends On**: Phase 3A.1 & 3B.1 (Domain Models & ERD) — Approved
> **Next Phase**: Phase 4 — API Specification Design

---

## Part 1: PostgreSQL Schema (Primary Relational Store)

This schema implements all relational entities in **PostgreSQL 16**. To comply with **Rule 1 (Multi-tenant SaaS)** and **Rule 2 (RBAC)**, Row-Level Security (RLS) is enabled on all tenant-specific tables.

### 1. Database Setup & RLS Config helper
```sql
-- Enable standard extension modules
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS btree_gin;

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
-- Optimized Partial Index for Active Users
CREATE INDEX idx_users_active ON users(tenant_id, email) WHERE status = 'ACTIVE';

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

### 3. API Key, Session, File Storage, & Audit Tables
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

CREATE TABLE file_storage_metadata (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    uploaded_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bucket_name VARCHAR(127) NOT NULL,
    s3_key VARCHAR(511) NOT NULL UNIQUE,
    file_type VARCHAR(63) NOT NULL, -- BACKTEST_REPORT, TAX_REPORT, USER_EXISTS
    mime_type VARCHAR(127) NOT NULL,
    size_bytes BIGINT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_files_tenant ON file_storage_metadata(tenant_id);

CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    actor_email VARCHAR(255),
    action VARCHAR(127) NOT NULL, -- USER_LOGIN, PASSWORD_RESET, EXPORT_DATA
    resource_type VARCHAR(63) NOT NULL, -- PORTFOLIO, SUBSCRIPTION, SYSTEM
    resource_id VARCHAR(127),
    changes JSONB, -- Pre/Post state diffs
    ip_address VARCHAR(45),
    user_agent TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- BRIN index on time-sequential audit logs to save index space
CREATE INDEX idx_audit_logs_brin ON audit_logs USING brin(created_at);
CREATE INDEX idx_audit_logs_tenant_action ON audit_logs(tenant_id, action);
```

### 4. Billing, Invoices, & Payment Tracking
```sql
CREATE TABLE billing_invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    invoice_number VARCHAR(127) NOT NULL UNIQUE,
    status VARCHAR(31) NOT NULL DEFAULT 'UNPAID', -- UNPAID, PAID, VOIDED, OVERDUE
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    amount_cents BIGINT NOT NULL DEFAULT 0,
    tax_cents BIGINT NOT NULL DEFAULT 0, -- India GST calculation logic
    discount_cents BIGINT NOT NULL DEFAULT 0,
    due_date DATE NOT NULL,
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    pdf_file_id UUID REFERENCES file_storage_metadata(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_invoices_tenant_status ON billing_invoices(tenant_id, status);

CREATE TABLE billing_invoice_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES billing_invoices(id) ON DELETE CASCADE,
    description VARCHAR(255) NOT NULL,
    quantity INT NOT NULL DEFAULT 1,
    unit_price_cents BIGINT NOT NULL DEFAULT 0,
    tax_rate_bps INT NOT NULL DEFAULT 1800, -- 18% standard GST
    amount_cents BIGINT NOT NULL DEFAULT 0
);

CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    invoice_id UUID NOT NULL REFERENCES billing_invoices(id) ON DELETE RESTRICT,
    payment_method VARCHAR(63) NOT NULL, -- RAZORPAY, STRIPE
    external_transaction_id VARCHAR(255) NOT NULL UNIQUE,
    status VARCHAR(31) NOT NULL DEFAULT 'PENDING', -- PENDING, SETTLED, FAILED, REFUNDED
    amount_cents BIGINT NOT NULL,
    fee_cents INT DEFAULT 0,
    gateway_response JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_payments_invoice ON payment_transactions(invoice_id);
```

### 5. Symbol Master & Derivatives Support
```sql
CREATE TABLE exchanges (
    code VARCHAR(15) PRIMARY KEY, -- NSE, BSE
    name VARCHAR(127) NOT NULL,
    timezone VARCHAR(63) NOT NULL DEFAULT 'Asia/Kolkata',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE market_holidays (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    exchange_code VARCHAR(15) NOT NULL REFERENCES exchanges(code) ON DELETE CASCADE,
    holiday_date DATE NOT NULL,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(31) NOT NULL DEFAULT 'FULL', -- FULL, PARTIAL
    trading_start_time TIME,
    trading_end_time TIME,
    UNIQUE(exchange_code, holiday_date)
);

CREATE TABLE market_symbols (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    exchange_code VARCHAR(15) NOT NULL REFERENCES exchanges(code) ON DELETE RESTRICT,
    ticker VARCHAR(63) NOT NULL, -- RELIANCE, NIFTY
    isin VARCHAR(12) UNIQUE,
    type VARCHAR(31) NOT NULL DEFAULT 'EQUITY', -- EQUITY, FUTURE, OPTION, INDEX
    lot_size INT NOT NULL DEFAULT 1,
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE',
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_symbol_ticker_exchange ON market_symbols(ticker, exchange_code);

CREATE TABLE derivatives_contracts (
    symbol_id UUID PRIMARY KEY REFERENCES market_symbols(id) ON DELETE CASCADE,
    underlying_symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    contract_type VARCHAR(15) NOT NULL, -- FUTURE, OPTION
    option_type VARCHAR(7), -- CALL, PUT, NULL for Futures
    strike_price_cents BIGINT, -- NULL for Futures
    expiry_date DATE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_derivatives_expiry ON derivatives_contracts(expiry_date);
```

### 6. Portfolio, Holdings, Orders, & Executions
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
    price_cents BIGINT,
    trigger_price_cents BIGINT,
    filled_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- Partial index to quickly process pending orders in queue
CREATE INDEX idx_orders_pending ON orders(portfolio_id) WHERE status = 'PENDING';
-- BRIN index for temporal ordering queries
CREATE INDEX idx_orders_brin ON orders USING brin(created_at);

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

CREATE TABLE broker_trade_executions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    broker_order_id UUID NOT NULL REFERENCES broker_orders(id) ON DELETE RESTRICT,
    external_execution_id VARCHAR(255) NOT NULL UNIQUE,
    quantity INT NOT NULL,
    execution_price_cents BIGINT NOT NULL,
    brokerage_fee_cents INT DEFAULT 0,
    stt_charges_cents INT DEFAULT 0, -- Securities Transaction Tax
    exchange_txn_fee_cents INT DEFAULT 0,
    gst_cents INT DEFAULT 0,
    sebi_turnover_fee_cents INT DEFAULT 0,
    stamp_duty_cents INT DEFAULT 0,
    executed_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_trade_executions_broker ON broker_trade_executions(broker_order_id);
```

### 7. Scanners, Watchlists, & Custom Strategies
```sql
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

CREATE TABLE strategies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    version INT NOT NULL DEFAULT 1,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_strategy_version ON strategies(name, version, COALESCE(tenant_id, '00000000-0000-0000-0000-000000000000'::uuid));
-- GIN Index for fast lookup over strategy config features
CREATE INDEX idx_strategies_config_gin ON strategies USING gin(config);

CREATE TABLE strategy_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    strategy_id UUID NOT NULL REFERENCES strategies(id) ON DELETE CASCADE,
    status VARCHAR(31) NOT NULL DEFAULT 'RUNNING',
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
    status VARCHAR(31) NOT NULL DEFAULT 'QUEUED',
    cagr_bps INT,
    sharpe_ratio INT,
    sortino_ratio INT,
    max_drawdown_bps INT,
    report_file_id UUID REFERENCES file_storage_metadata(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE scanners (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    rules JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- GIN index for JSONB rules search
CREATE INDEX idx_scanners_rules_gin ON scanners USING gin(rules);

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
    metric_snapshots JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_scanner_matches_run ON scanner_matches(scanner_run_id);
```

### 8. AI Infrastructure & Model Accuracy Metadata
```sql
CREATE TABLE model_registries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(127) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ai_models (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    registry_id UUID NOT NULL REFERENCES model_registries(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    feature_config JSONB NOT NULL DEFAULT '[]',
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
    version_string VARCHAR(63) NOT NULL,
    status VARCHAR(31) NOT NULL DEFAULT 'CANDIDATE',
    feature_store_version_id UUID NOT NULL REFERENCES feature_store_versions(id) ON DELETE RESTRICT,
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    metrics JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE model_artifacts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    file_id UUID NOT NULL REFERENCES file_storage_metadata(id) ON DELETE RESTRICT,
    sha256_checksum VARCHAR(64) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE model_calibrations (
    model_version_id UUID PRIMARY KEY REFERENCES model_versions(id) ON DELETE CASCADE,
    calibration_method VARCHAR(63) NOT NULL,
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

CREATE TABLE prediction_accuracies (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    eval_date DATE NOT NULL,
    precision_bps INT NOT NULL, -- Precision in basis points
    recall_bps INT NOT NULL,
    f1_bps INT NOT NULL,
    mean_squared_error DOUBLE PRECISION,
    confusion_matrix JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_model_accuracy_eval ON prediction_accuracies(model_version_id, eval_date);
```

### 9. Signals, AI Chat, & Explainability History
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
    status VARCHAR(31) NOT NULL, -- TARGET_HIT, SL_HIT, EXPIRED
    mfe_bps INT,
    mae_bps INT,
    realized_return_bps INT,
    analyzed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Explainability Compliance Archival Store (SHAP Matrices auditing)
CREATE TABLE shap_explanations_history (
    prediction_id UUID PRIMARY KEY,
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE RESTRICT,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    shap_matrix JSONB NOT NULL, -- Raw weights mapping
    base_value DOUBLE PRECISION NOT NULL,
    feature_values JSONB NOT NULL, -- Feature values at time of execution
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_shap_history_symbol ON shap_explanations_history(symbol_id);

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

CREATE TABLE ai_chat_histories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    user_message TEXT NOT NULL,
    model_response TEXT NOT NULL,
    model_version_id UUID REFERENCES model_versions(id) ON DELETE SET NULL,
    prompt_tokens INT NOT NULL DEFAULT 0,
    completion_tokens INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- BRIN index on time-sequential chat events
CREATE INDEX idx_ai_chat_brin ON ai_chat_histories USING brin(created_at);
```

### 10. Alert Configurations & API Usage Metrics
```sql
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
-- GIN index for parameters rules search
CREATE INDEX idx_alert_rules_params_gin ON alert_rules USING gin(condition_params);
-- Partial index on active alert rules
CREATE INDEX idx_alert_rules_active ON alert_rules(symbol_id) WHERE is_active = TRUE;

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
    status VARCHAR(31) NOT NULL DEFAULT 'SENT',
    error_message TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE api_usage_analytics (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    api_key_id UUID REFERENCES api_keys(id) ON DELETE SET NULL,
    endpoint_path VARCHAR(255) NOT NULL,
    request_method VARCHAR(15) NOT NULL,
    response_code INT NOT NULL,
    latency_ms INT NOT NULL,
    bytes_sent BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_api_usage_brin ON api_usage_analytics USING brin(created_at);
CREATE INDEX idx_api_usage_metrics ON api_usage_analytics(tenant_id, endpoint_path, response_code);
```

### 11. Risk Management, Webhooks, & News/Sentiment Contexts
```sql
CREATE TABLE risk_rules (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID REFERENCES portfolios(id) ON DELETE CASCADE,
    rule_type VARCHAR(63) NOT NULL,
    rule_params JSONB NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE risk_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    risk_rule_id UUID NOT NULL REFERENCES risk_rules(id) ON DELETE CASCADE,
    severity VARCHAR(15) NOT NULL DEFAULT 'WARNING',
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
    events JSONB NOT NULL DEFAULT '[]',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE news_articles (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    headline VARCHAR(511) NOT NULL,
    body_excerpt TEXT NOT NULL,
    source VARCHAR(127) NOT NULL,
    author VARCHAR(127),
    article_url VARCHAR(511) NOT NULL UNIQUE,
    published_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_news_published ON news_articles(published_at DESC);

CREATE TABLE news_sentiment_scores (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    news_article_id UUID NOT NULL REFERENCES news_articles(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE CASCADE,
    sentiment_score_bps INT NOT NULL, -- Sentiment score scale -10000 to +10000
    confidence_pct INT NOT NULL, -- 0 to 100
    impact_multiplier INT NOT NULL DEFAULT 100, -- Scale multiplier (100 = 1.0x)
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_news_sentiment_unique ON news_sentiment_scores(news_article_id, symbol_id);

CREATE TABLE economic_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    country VARCHAR(63) NOT NULL,
    category VARCHAR(127) NOT NULL,
    release_time TIMESTAMP WITH TIME ZONE NOT NULL,
    actual_value VARCHAR(63),
    forecast_value VARCHAR(63),
    previous_value VARCHAR(63),
    impact_level VARCHAR(15) NOT NULL DEFAULT 'LOW', -- LOW, MEDIUM, HIGH
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_economic_release ON economic_events(release_time DESC);
```

### 12. RLS Policies Configuration (Rule 1)
```sql
-- Enable Row Level Security on all tenant tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE file_storage_metadata ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing_invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;
ALTER TABLE holdings ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE broker_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE broker_orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE broker_trade_executions ENABLE ROW LEVEL SECURITY;
ALTER TABLE watchlists ENABLE ROW LEVEL SECURITY;
ALTER TABLE strategies ENABLE ROW LEVEL SECURITY;
ALTER TABLE strategy_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE backtests ENABLE ROW LEVEL SECURITY;
ALTER TABLE scanners ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_feedbacks ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_chat_histories ENABLE ROW LEVEL SECURITY;
ALTER TABLE alert_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE alert_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_preferences ENABLE ROW LEVEL SECURITY;
ALTER TABLE notification_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_usage_analytics ENABLE ROW LEVEL SECURITY;
ALTER TABLE risk_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE risk_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE webhook_subscriptions ENABLE ROW LEVEL SECURITY;

-- Dynamic isolation policies mapping
CREATE POLICY users_isolation ON users FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY profiles_isolation ON user_profiles FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY subscriptions_isolation ON tenant_subscriptions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY api_keys_isolation ON api_keys FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY sessions_isolation ON sessions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY files_isolation ON file_storage_metadata FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY audits_isolation ON audit_logs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY invoices_isolation ON billing_invoices FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY payments_isolation ON payment_transactions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY portfolios_isolation ON portfolios FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY holdings_isolation ON holdings FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY orders_isolation ON orders FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_accounts_isolation ON broker_accounts FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_orders_isolation ON broker_orders FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY executions_isolation ON broker_trade_executions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY watchlists_isolation ON watchlists FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY strategies_isolation ON strategies FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY runs_isolation ON strategy_runs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY backtests_isolation ON backtests FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY scanners_isolation ON scanners FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY feedback_isolation ON ai_feedbacks FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY chat_history_isolation ON ai_chat_histories FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY alert_rules_isolation ON alert_rules FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY alert_events_isolation ON alert_events FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY preferences_isolation ON notification_preferences FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY logs_isolation ON notification_logs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY usage_metrics_isolation ON api_usage_analytics FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY risk_rules_isolation ON risk_rules FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY risk_events_isolation ON risk_events FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY webhooks_isolation ON webhook_subscriptions FOR ALL USING (tenant_id = get_current_tenant_id());
```

---

## Part 2: TimescaleDB Schema & Continuous Aggregates

Built in **TimescaleDB** using hypertable abstractions. We define the continuous aggregation roll-ups that translate raw tick variables into OHLCV views.

### 1. Base Time-Series Tables
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
SELECT create_hypertable('market_ticks', 'time', chunk_time_interval => INTERVAL '1 day');
CREATE INDEX idx_ticks_symbol_time ON market_ticks(symbol_id, time DESC);

-- Compression setup
ALTER TABLE market_ticks SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('market_ticks', INTERVAL '2 days');

CREATE TABLE ohlcv_candles_1m (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    open_cents BIGINT NOT NULL,
    high_cents BIGINT NOT NULL,
    low_cents BIGINT NOT NULL,
    close_cents BIGINT NOT NULL,
    volume BIGINT NOT NULL
);
SELECT create_hypertable('ohlcv_candles_1m', 'time', chunk_time_interval => INTERVAL '7 days');
CREATE UNIQUE INDEX idx_ohlcv_1m ON ohlcv_candles_1m(symbol_id, time DESC);

ALTER TABLE ohlcv_candles_1m SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('ohlcv_candles_1m', INTERVAL '14 days');
```

### 2. Continuous Aggregates (Rollups)
```sql
-- 5-Minute Candle Rollup Definition
CREATE MATERIALIZED VIEW ohlcv_candles_5m
WITH (timescaledb.continuous) AS
SELECT
    time_bucket(INTERVAL '5 minutes', time) AS time,
    symbol_id,
    first(open_cents, time) AS open_cents,
    max(high_cents) AS high_cents,
    min(low_cents) AS low_cents,
    last(close_cents, time) AS close_cents,
    sum(volume) AS volume
FROM ohlcv_candles_1m
GROUP BY time, symbol_id;

SELECT add_continuous_aggregate_policy('ohlcv_candles_5m',
    start_offset => INTERVAL '1 day',
    end_offset => INTERVAL '10 minutes',
    schedule_interval => INTERVAL '5 minutes');

-- 1-Hour Candle Rollup Definition
CREATE MATERIALIZED VIEW ohlcv_candles_1h
WITH (timescaledb.continuous) AS
SELECT
    time_bucket(INTERVAL '1 hour', time) AS time,
    symbol_id,
    first(open_cents, time) AS open_cents,
    max(high_cents) AS high_cents,
    min(low_cents) AS low_cents,
    last(close_cents, time) AS close_cents,
    sum(volume) AS volume
FROM ohlcv_candles_1m
GROUP BY time, symbol_id;

SELECT add_continuous_aggregate_policy('ohlcv_candles_1h',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- 1-Day Candle Rollup Definition
CREATE MATERIALIZED VIEW ohlcv_candles_1d
WITH (timescaledb.continuous) AS
SELECT
    time_bucket(INTERVAL '1 day', time) AS time,
    symbol_id,
    first(open_cents, time) AS open_cents,
    max(high_cents) AS high_cents,
    min(low_cents) AS low_cents,
    last(close_cents, time) AS close_cents,
    sum(volume) AS volume
FROM ohlcv_candles_1m
GROUP BY time, symbol_id;

SELECT add_continuous_aggregate_policy('ohlcv_candles_1d',
    start_offset => INTERVAL '30 days',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '12 hours');
```

---

## Part 3: ClickHouse Schema (Analytics OLAP Store)

Aggregated reporting structures designed to parse system audit logs and signal distributions quickly.

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

---

## Part 4: Redis Data Structures (In-Memory Key Schema)

Memory namespaces mapped for rate-limits, session validation, and real-time WebSocket distribution.

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
      { "name": "indicators_json", "type": "string" },
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

*Phase 3C — Physical Database Schema Design v1.1*
*Status: Pending User Review and Approval*
*Next Phase: Phase 4 — API Specification Design (awaiting approval)*
