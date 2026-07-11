# Phase 3C — Physical Database Schema Design
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Status**: Approved
> **Version**: 1.5 (Final Production Hardening — Schema Freeze)
> **Previous Version**: 1.4 (Enterprise Hardened)
> **Depends On**: Phase 3A.1 & 3B.1 (Domain Models & ERD) — Approved
> **Next Phase**: Phase 4 — API Specification Design
> **Deployment Context**: Initial single-user/private production deployment. Schema is forward-compatible for future public SaaS evolution.

---

## Part 1: PostgreSQL Schema (Primary Relational Store)

This schema implements all relational entities in **PostgreSQL 16**. To comply with **Rule 1 (Multi-tenant SaaS)** and **Rule 2 (RBAC)**, Row-Level Security (RLS) is enabled on all tenant-specific tables.

### 1. Database Setup & RLS Config helper
```sql
-- Enable standard extension modules
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- ============================================================
-- SECURITY: get_current_tenant_id()
-- ============================================================
-- Extracts the current tenant UUID from the session-local GUC
-- 'app.current_tenant_id', which MUST be set by the application
-- on every pooled connection before executing any SQL.
--
-- v1.5 CHANGE: Fail-fast behavior replaces silent NULL return.
-- If the GUC is missing or empty, the function raises an exception
-- immediately. This prevents RLS bypass on misconfigured connections
-- (e.g. background workers, migration runners, or bare psql sessions).
--
-- SECURITY DEFINER is used so the function always executes with the
-- privileges of its creator (a dedicated db-owner role), not the
-- calling application role. SET search_path eliminates search-path
-- hijacking by pinning execution to the public schema only.
--
-- OPERATIONAL CONTRACT:
-- - All application connection pools MUST execute:
--     SET app.current_tenant_id = '<tenant_uuid>';
--   immediately after acquiring a connection, before any DML.
-- - Migration runners and admin tools MUST connect using a role
--   with BYPASSRLS privilege and must NEVER share the application
--   connection pool. They should set:
--     SET app.current_tenant_id = '00000000-0000-0000-0000-000000000000';
--   as a sentinel value and bypass RLS explicitly.
-- ============================================================
CREATE OR REPLACE FUNCTION get_current_tenant_id()
RETURNS UUID AS $$
DECLARE
    v_tenant_id TEXT;
BEGIN
    v_tenant_id := current_setting('app.current_tenant_id', true);
    IF v_tenant_id IS NULL OR trim(v_tenant_id) = '' THEN
        RAISE EXCEPTION
            'Security violation: app.current_tenant_id is not set in the current session. '
            'All application connections must set this GUC before executing queries.'
            USING ERRCODE = 'insufficient_privilege';
    END IF;
    RETURN v_tenant_id::uuid;
EXCEPTION
    WHEN invalid_text_representation THEN
        RAISE EXCEPTION
            'Security violation: app.current_tenant_id contains an invalid UUID value: %',
            v_tenant_id
            USING ERRCODE = 'invalid_parameter_value';
END;
$$ LANGUAGE plpgsql SECURITY DEFINER SET search_path = public, pg_temp;
```

### 2. Core Tenant, Subscription, & Feature Flag Tables
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

CREATE TABLE tenant_feature_flags (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    feature_name VARCHAR(127) NOT NULL,
    is_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    quota_limit INT NOT NULL DEFAULT 0, -- 0 means unlimited
    usage_counter INT NOT NULL DEFAULT 0,
    reset_interval VARCHAR(31) NOT NULL DEFAULT 'NEVER', -- DAILY, MONTHLY, NEVER
    last_reset_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, feature_name)
);
CREATE INDEX idx_tenant_features_name ON tenant_feature_flags(tenant_id, feature_name);

CREATE TABLE subscription_plans (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(63) NOT NULL UNIQUE, -- FREE, TRIAL, PRO, ENTERPRISE
    price_cents INT NOT NULL DEFAULT 0 CONSTRAINT check_plan_price CHECK (price_cents >= 0),
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
CREATE INDEX idx_tenant_subs_plan ON tenant_subscriptions(plan_id); -- FK Index

-- ============================================================
-- CONSTRAINT: One active subscription per tenant at a time.
-- Prevents billing double-charge and non-deterministic quota
-- enforcement caused by duplicate ACTIVE or TRIAL rows.
-- PAST_DUE and CANCELLED rows are excluded — multiple historical
-- records are valid (one per billing cycle).
-- ============================================================
CREATE UNIQUE INDEX idx_tenant_subs_one_active
    ON tenant_subscriptions(tenant_id)
    WHERE status IN ('TRIAL', 'ACTIVE');
```

### 3. Identity & Role-Based Access Control
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email VARCHAR(255) NOT NULL,
    -- TEXT (not VARCHAR) to accommodate all modern password hashing algorithms.
    -- bcrypt: ~60 chars. PBKDF2: ~75 chars. Argon2id (NIST SP 800-63B): up to 95+ chars.
    -- VARCHAR(255) would silently truncate Argon2id hashes under high-security
    -- parameter configurations, breaking all future password verifications.
    password_hash TEXT NOT NULL,
    status VARCHAR(31) NOT NULL DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, PENDING_MFA
    mfa_secret VARCHAR(127),
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- Case-Insensitive Unique email check per Tenant to enforce security
CREATE UNIQUE INDEX idx_users_email_ci ON users(tenant_id, LOWER(email));
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
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- NULL for system-wide defaults
    PRIMARY KEY (role_id, permission_id)
);
CREATE INDEX idx_role_permissions_tenant ON role_permissions(tenant_id);
CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);
CREATE INDEX idx_role_permissions_perm ON role_permissions(permission_id); -- FK Index

CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);
CREATE INDEX idx_user_roles_tenant ON user_roles(tenant_id);
CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_role ON user_roles(role_id); -- FK Index
```

### 4. Sessions, API Keys, Audit Logs, & Transactional Outbox
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
CREATE INDEX idx_sessions_user ON sessions(user_id); -- FK Index

CREATE TABLE file_storage_metadata (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    uploaded_by UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    bucket_name VARCHAR(127) NOT NULL,
    s3_key VARCHAR(511) NOT NULL UNIQUE,
    file_type VARCHAR(63) NOT NULL, -- BACKTEST_REPORT, TAX_REPORT, MODEL_WEIGHTS
    mime_type VARCHAR(127) NOT NULL,
    size_bytes BIGINT NOT NULL CONSTRAINT check_file_size CHECK (size_bytes >= 0),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_files_tenant ON file_storage_metadata(tenant_id);
CREATE INDEX idx_files_uploader ON file_storage_metadata(uploaded_by); -- FK Index

-- ============================================================
-- COMPLIANCE: audit_logs retention strategy (v1.5 change)
-- ============================================================
-- ON DELETE RESTRICT prevents physical deletion of audit log
-- records when a tenant is deleted or offboarded. This is a
-- mandatory compliance requirement:
--   - SEBI regulations require audit trail retention for 5 years.
--   - India IT Act Section 43A requires security incident records
--     to be preserved regardless of customer lifecycle status.
--   - GDPR deletion requests must anonymise PII (ip_address,
--     user_agent, actor_email) but must NOT delete the audit event.
--
-- TENANT OFFBOARDING CONTRACT (application-layer):
--   1. Mark tenant status = 'DEACTIVATED' (soft delete only).
--   2. Anonymise PII in audit_logs: SET actor_email = 'redacted',
--      ip_address = NULL, user_agent = NULL WHERE tenant_id = $id.
--   3. Retain all audit rows for the compliance retention period.
--   4. Hard deletion of a tenant record is only permitted after
--      the retention period has elapsed and a legal hold check passes.
--
-- FUTURE ROADMAP: When multi-tenant scale requires it, migrate
-- audit_logs to an append-only, immutable store (e.g. AWS QLDB,
-- Kafka compacted topic, or a write-protected PostgreSQL table
-- with a BEFORE DELETE trigger that raises an exception).
-- ============================================================
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT, -- COMPLIANCE: no cascade delete
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    actor_email VARCHAR(255), -- anonymise on GDPR deletion, do not delete row
    action VARCHAR(127) NOT NULL, -- USER_LOGIN, PASSWORD_RESET, EXPORT_DATA
    resource_type VARCHAR(63) NOT NULL, -- PORTFOLIO, SUBSCRIPTION, SYSTEM
    resource_id VARCHAR(127),
    changes JSONB, -- Pre/Post state diffs
    ip_address VARCHAR(45), -- anonymise on GDPR deletion
    user_agent TEXT,        -- anonymise on GDPR deletion
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- BRIN index on time-sequential audit logs to save index space
CREATE INDEX idx_audit_logs_brin ON audit_logs USING brin(created_at);
CREATE INDEX idx_audit_logs_tenant_action ON audit_logs(tenant_id, action);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id); -- FK Index

-- Transactional Outbox Table for Reliability and Kafka Event Duplication
CREATE TABLE outbox_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID REFERENCES tenants(id) ON DELETE CASCADE, -- NULL for system events
    topic VARCHAR(127) NOT NULL, -- Kafka topic target
    key VARCHAR(255) NOT NULL, -- Kafka partition key
    payload JSONB NOT NULL,
    status VARCHAR(31) NOT NULL DEFAULT 'PENDING', -- PENDING, PROCESSED, FAILED
    error_message TEXT,
    retry_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at TIMESTAMP WITH TIME ZONE
);
-- Optimized index for Outbox sweeper to handle status, retries, and sequential ordering
CREATE INDEX idx_outbox_events_sweep ON outbox_events(status, retry_count, created_at) WHERE status = 'PENDING';

-- API Request Idempotency Key Ledger
CREATE TABLE idempotency_keys (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    key_value VARCHAR(255) NOT NULL,
    request_path VARCHAR(511) NOT NULL,
    response_code INT,
    response_body TEXT,
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tenant_id, user_id, key_value)
);
CREATE INDEX idx_idempotency_expiry ON idempotency_keys(expires_at) WHERE expires_at < CURRENT_TIMESTAMP;
```

### 5. Billing, Invoices, & Payment Tracking
```sql
CREATE TABLE billing_invoices (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    invoice_number VARCHAR(127) NOT NULL UNIQUE,
    status VARCHAR(31) NOT NULL DEFAULT 'UNPAID', -- UNPAID, PAID, VOIDED, OVERDUE
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    amount_cents BIGINT NOT NULL DEFAULT 0 CONSTRAINT check_invoice_amount CHECK (amount_cents >= 0),
    tax_cents BIGINT NOT NULL DEFAULT 0 CONSTRAINT check_invoice_tax CHECK (tax_cents >= 0), -- India GST
    discount_cents BIGINT NOT NULL DEFAULT 0 CONSTRAINT check_invoice_discount CHECK (discount_cents >= 0),
    due_date DATE NOT NULL,
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    pdf_file_id UUID REFERENCES file_storage_metadata(id) ON DELETE SET NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_invoices_tenant_status ON billing_invoices(tenant_id, status);
CREATE INDEX idx_invoices_pdf_file ON billing_invoices(pdf_file_id); -- FK Index

CREATE TABLE billing_invoice_items (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    invoice_id UUID NOT NULL REFERENCES billing_invoices(id) ON DELETE CASCADE,
    description VARCHAR(255) NOT NULL,
    quantity INT NOT NULL DEFAULT 1 CONSTRAINT check_item_qty CHECK (quantity > 0),
    unit_price_cents BIGINT NOT NULL DEFAULT 0 CONSTRAINT check_item_price CHECK (unit_price_cents >= 0),
    tax_rate_bps INT NOT NULL DEFAULT 1800, -- 18% standard GST
    -- Enforce deterministic line calculation
    amount_cents BIGINT GENERATED ALWAYS AS (quantity * unit_price_cents) STORED
);
CREATE INDEX idx_invoice_items_invoice ON billing_invoice_items(invoice_id); -- FK Index

CREATE TABLE payment_transactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE RESTRICT,
    invoice_id UUID NOT NULL REFERENCES billing_invoices(id) ON DELETE RESTRICT,
    payment_method VARCHAR(63) NOT NULL, -- RAZORPAY, STRIPE
    external_transaction_id VARCHAR(255) NOT NULL UNIQUE,
    status VARCHAR(31) NOT NULL DEFAULT 'PENDING', -- PENDING, SETTLED, FAILED, REFUNDED
    amount_cents BIGINT NOT NULL CONSTRAINT check_pay_amount CHECK (amount_cents >= 0),
    fee_cents INT DEFAULT 0 CONSTRAINT check_pay_fee CHECK (fee_cents >= 0),
    gateway_response JSONB,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_payments_invoice ON payment_transactions(invoice_id);
CREATE INDEX idx_payments_tenant ON payment_transactions(tenant_id); -- FK Index
```

### 6. Symbol Master & Derivatives Support
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
CREATE INDEX idx_holidays_exchange ON market_holidays(exchange_code); -- FK Index

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
CREATE INDEX idx_symbols_exchange ON market_symbols(exchange_code); -- FK Index

CREATE TABLE derivatives_contracts (
    symbol_id UUID PRIMARY KEY REFERENCES market_symbols(id) ON DELETE CASCADE,
    underlying_symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    contract_type VARCHAR(15) NOT NULL, -- FUTURE, OPTION
    option_type VARCHAR(7), -- CALL, PUT, NULL for Futures
    strike_price_cents BIGINT CONSTRAINT check_strike_price CHECK (strike_price_cents >= 0), -- NULL for Futures
    expiry_date DATE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_derivatives_expiry ON derivatives_contracts(expiry_date);
CREATE INDEX idx_derivatives_underlying ON derivatives_contracts(underlying_symbol_id); -- FK Index
```

### 7. Portfolios, Holdings, Orders, OCC, & Double-Entry Ledger
```sql
CREATE TABLE portfolios (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(31) NOT NULL DEFAULT 'PAPER', -- PAPER, LIVE
    currency VARCHAR(3) NOT NULL DEFAULT 'INR',
    balance_cents BIGINT NOT NULL DEFAULT 100000000 CONSTRAINT check_portfolio_bal CHECK (balance_cents >= 0),
    version INT NOT NULL DEFAULT 1, -- Optimistic Concurrency Control (OCC)
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_portfolios_user ON portfolios(tenant_id, user_id);

-- Double-Entry Bookkeeping Ledger to ensure Financial Correctness
CREATE TABLE portfolio_ledger_entries (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE RESTRICT,
    type VARCHAR(31) NOT NULL, -- DEPOSIT, WITHDRAW, TRADE_BUY, TRADE_SELL, FEE, GST, STT
    amount_cents BIGINT NOT NULL, -- Negative for debits, Positive for credits
    balance_after_cents BIGINT NOT NULL CONSTRAINT check_ledger_bal_post CHECK (balance_after_cents >= 0),
    reference_id VARCHAR(127), -- Linked Order ID or Transaction ID
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_portfolio_ledger_brin ON portfolio_ledger_entries USING brin(created_at);
CREATE INDEX idx_portfolio_ledger_portfolio ON portfolio_ledger_entries(portfolio_id);

CREATE TABLE holdings (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    quantity INT NOT NULL DEFAULT 0,
    average_cost_cents BIGINT NOT NULL DEFAULT 0 CONSTRAINT check_avg_cost CHECK (average_cost_cents >= 0),
    version INT NOT NULL DEFAULT 1, -- Optimistic Concurrency Control (OCC)
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT check_positive_quantity CHECK (quantity >= 0)
);
-- Covering index for performance aggregation
CREATE UNIQUE INDEX idx_holdings_covering ON holdings(portfolio_id, symbol_id) INCLUDE (quantity, average_cost_cents);
CREATE INDEX idx_holdings_symbol ON holdings(symbol_id); -- FK Index

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    portfolio_id UUID NOT NULL REFERENCES portfolios(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    direction VARCHAR(15) NOT NULL, -- BUY, SELL
    type VARCHAR(15) NOT NULL, -- MARKET, LIMIT, STOP_LOSS, STOP_LOSS_LIMIT
    status VARCHAR(31) NOT NULL DEFAULT 'PENDING', -- PENDING, SUBMITTED, FILLED, CANCELLED, REJECTED
    quantity INT NOT NULL CONSTRAINT check_order_qty CHECK (quantity > 0),
    price_cents BIGINT CONSTRAINT check_order_price CHECK (price_cents >= 0),
    trigger_price_cents BIGINT CONSTRAINT check_trigger_price CHECK (trigger_price_cents >= 0),
    filled_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
-- Optimized Partial Index on Pending Orders
CREATE INDEX idx_orders_pending_partial ON orders(portfolio_id) WHERE status = 'PENDING';
CREATE INDEX idx_orders_brin ON orders USING brin(created_at);
CREATE INDEX idx_orders_symbol ON orders(symbol_id); -- FK Index

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
    -- encryption_key_version identifies which application-managed encryption key
    -- was used to encrypt the token columns above. This enables selective
    -- re-encryption row-by-row when a key rotation event occurs, without
    -- requiring a full table lock or simultaneous user re-authorisation.
    -- The actual key material lives in an external secrets manager (e.g. AWS KMS,
    -- HashiCorp Vault). The application reads this version alongside the ciphertext
    -- to select the correct decryption key. Default 1 = initial key version.
    encryption_key_version INT NOT NULL DEFAULT 1,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_broker_account_user ON broker_accounts(user_id, integration_id);
CREATE INDEX idx_broker_accounts_integration ON broker_accounts(integration_id); -- FK Index
-- Supports efficient identification of rows encrypted under an old key version
-- during background re-encryption jobs after key rotation.
CREATE INDEX idx_broker_accounts_key_ver ON broker_accounts(encryption_key_version);

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
CREATE INDEX idx_broker_orders_account ON broker_orders(broker_account_id); -- FK Index
CREATE INDEX idx_broker_orders_order ON broker_orders(order_id); -- FK Index

CREATE TABLE broker_trade_executions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    broker_order_id UUID NOT NULL REFERENCES broker_orders(id) ON DELETE RESTRICT,
    external_execution_id VARCHAR(255) NOT NULL UNIQUE,
    quantity INT NOT NULL CONSTRAINT check_exec_qty CHECK (quantity > 0),
    execution_price_cents BIGINT NOT NULL CONSTRAINT check_exec_price CHECK (execution_price_cents >= 0),
    brokerage_fee_cents INT DEFAULT 0 CONSTRAINT check_exec_brokerage CHECK (brokerage_fee_cents >= 0),
    stt_charges_cents INT DEFAULT 0 CONSTRAINT check_exec_stt CHECK (stt_charges_cents >= 0), -- Securities Transaction Tax
    exchange_txn_fee_cents INT DEFAULT 0 CONSTRAINT check_exec_fee CHECK (exchange_txn_fee_cents >= 0),
    gst_cents INT DEFAULT 0 CONSTRAINT check_exec_gst CHECK (gst_cents >= 0),
    sebi_turnover_fee_cents INT DEFAULT 0 CONSTRAINT check_exec_sebi CHECK (sebi_turnover_fee_cents >= 0),
    stamp_duty_cents INT DEFAULT 0 CONSTRAINT check_exec_stamp CHECK (stamp_duty_cents >= 0),
    executed_at TIMESTAMP WITH TIME ZONE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_trade_executions_broker ON broker_trade_executions(broker_order_id);
CREATE INDEX idx_trade_executions_tenant ON broker_trade_executions(tenant_id); -- FK Index
```

### 8. Scanners, Watchlists, & Custom Strategies
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
CREATE INDEX idx_watchlist_items_symbol ON watchlist_items(symbol_id); -- FK Index

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
CREATE INDEX idx_strategy_runs_strategy ON strategy_runs(strategy_id); -- FK Index

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
CREATE INDEX idx_backtests_strategy ON backtests(strategy_id); -- FK Index

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
CREATE INDEX idx_scanners_tenant ON scanners(tenant_id); -- FK Index

CREATE TABLE scanner_runs (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- Scoped for RLS
    scanner_id UUID NOT NULL REFERENCES scanners(id) ON DELETE CASCADE,
    execution_time TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    duration_ms INT
);
CREATE INDEX idx_scanner_runs_tenant_time ON scanner_runs(tenant_id, execution_time DESC);
CREATE INDEX idx_scanner_runs_scanner ON scanner_runs(scanner_id); -- FK Index

CREATE TABLE scanner_matches (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- Scoped for RLS
    scanner_run_id UUID NOT NULL REFERENCES scanner_runs(id) ON DELETE CASCADE,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE CASCADE,
    metric_snapshots JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_scanner_matches_tenant_run ON scanner_matches(tenant_id, scanner_run_id);
CREATE INDEX idx_scanner_matches_symbol ON scanner_matches(symbol_id); -- FK Index
```

### 9. AI Infrastructure & Model Accuracy Metadata
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

-- ============================================================
-- AI GOVERNANCE: model_versions lifecycle (v1.5)
-- ============================================================
-- Valid status values are enforced at the database level via a
-- CHECK constraint. Only status transitions permitted by the
-- defined Finite State Machine (FSM) are allowed.
--
-- Permitted FSM transitions:
--   CANDIDATE  -> STAGING   (promoted for shadow testing)
--   CANDIDATE  -> FAILED    (evaluation failed)
--   STAGING    -> PRODUCTION (promoted after A/B validation)
--   STAGING    -> CANDIDATE  (demoted — needs more tuning)
--   STAGING    -> FAILED    (shadow testing failed)
--   PRODUCTION -> DEPRECATED (superseded by a newer model version)
--   DEPRECATED -> (terminal — no further transitions allowed)
--   FAILED     -> (terminal — no further transitions allowed)
--
-- ENFORCEMENT:
--   The CHECK constraint below prevents any invalid status string
--   from being written. The FSM transition rules documented above
--   MUST be enforced by the application service layer (ModelService)
--   when updating the status field. Do not allow direct SQL updates
--   to model_versions.status from API handlers.
--
-- FUTURE ROADMAP: When model governance requires stricter guarantees,
--   implement a PostgreSQL BEFORE UPDATE trigger that enforces the
--   FSM transition matrix at the database level. For v1.5 (private
--   deployment), application-layer enforcement is sufficient.
-- ============================================================
CREATE TABLE model_versions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_id UUID NOT NULL REFERENCES ai_models(id) ON DELETE CASCADE,
    version_string VARCHAR(63) NOT NULL,
    -- CHECK enforces valid status values. FSM transitions are enforced at app layer.
    status VARCHAR(31) NOT NULL DEFAULT 'CANDIDATE'
        CONSTRAINT chk_model_version_status
        CHECK (status IN ('CANDIDATE', 'STAGING', 'PRODUCTION', 'DEPRECATED', 'FAILED')),
    feature_store_version_id UUID NOT NULL REFERENCES feature_store_versions(id) ON DELETE RESTRICT,
    hyperparameters JSONB NOT NULL DEFAULT '{}',
    metrics JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_model_versions_model ON model_versions(model_id); -- FK Index
-- Supports fast queries for the currently active production model
CREATE INDEX idx_model_versions_production ON model_versions(model_id, status)
    WHERE status = 'PRODUCTION';

CREATE TABLE model_artifacts (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    file_id UUID NOT NULL REFERENCES file_storage_metadata(id) ON DELETE RESTRICT,
    sha256_checksum VARCHAR(64) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_model_artifacts_ver ON model_artifacts(model_version_id); -- FK Index
CREATE INDEX idx_model_artifacts_file ON model_artifacts(file_id); -- FK Index

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
CREATE INDEX idx_prediction_accuracies_ver ON prediction_accuracies(model_version_id); -- FK Index
```

### 10. Signals, AI Chat, & Explainability History
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
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE, -- Scoped for RLS
    model_version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE RESTRICT,
    symbol_id UUID NOT NULL REFERENCES market_symbols(id) ON DELETE RESTRICT,
    shap_matrix JSONB NOT NULL, -- Raw weights mapping
    base_value DOUBLE PRECISION NOT NULL,
    feature_values JSONB NOT NULL, -- Feature values at execution timestamp
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_shap_history_tenant ON shap_explanations_history(tenant_id);
CREATE INDEX idx_shap_history_symbol ON shap_explanations_history(symbol_id);
CREATE INDEX idx_shap_history_ver ON shap_explanations_history(model_version_id); -- FK Index

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
CREATE INDEX idx_feedbacks_signal ON ai_feedbacks(signal_id); -- FK Index

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
CREATE INDEX idx_ai_chat_ver ON ai_chat_histories(model_version_id); -- FK Index
```

### 11. Alert Configurations & API Usage Metrics
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
CREATE INDEX idx_alert_rules_user ON alert_rules(user_id); -- FK Index

CREATE TABLE alert_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    rule_id UUID REFERENCES alert_rules(id) ON DELETE SET NULL,
    signal_id UUID REFERENCES trade_signals(id) ON DELETE SET NULL,
    trigger_message TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_alert_events_rule ON alert_events(rule_id); -- FK Index
CREATE INDEX idx_alert_events_signal ON alert_events(signal_id); -- FK Index

CREATE TABLE notification_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channels JSONB NOT NULL DEFAULT '{"email": true, "push": true, "telegram": false, "whatsapp": false}',
    telegram_chat_id VARCHAR(127),
    whatsapp_phone_number VARCHAR(31),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_preferences_tenant ON notification_preferences(tenant_id); -- FK Index

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
CREATE INDEX idx_notification_logs_alert ON notification_logs(alert_event_id); -- FK Index

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
CREATE INDEX idx_api_usage_key ON api_usage_analytics(api_key_id); -- FK Index
```

### 12. Risk Management, Webhooks, & News/Sentiment Contexts
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
CREATE INDEX idx_risk_rules_portfolio ON risk_rules(portfolio_id); -- FK Index

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
CREATE INDEX idx_risk_events_portfolio ON risk_events(portfolio_id);
CREATE INDEX idx_risk_events_rule ON risk_events(risk_rule_id); -- FK Index

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
CREATE INDEX idx_webhooks_user ON webhook_subscriptions(user_id); -- FK Index

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
    confidence_pct INT NOT NULL,
    impact_multiplier INT NOT NULL DEFAULT 100,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE UNIQUE INDEX idx_news_sentiment_unique ON news_sentiment_scores(news_article_id, symbol_id);
CREATE INDEX idx_news_sentiment_symbol ON news_sentiment_scores(symbol_id); -- FK Index

CREATE TABLE economic_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    country VARCHAR(63) NOT NULL,
    category VARCHAR(127) NOT NULL,
    release_time TIMESTAMP WITH TIME ZONE NOT NULL,
    actual_value VARCHAR(63),
    forecast_value VARCHAR(63),
    previous_value VARCHAR(63),
    impact_level VARCHAR(15) NOT NULL DEFAULT 'LOW',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_economic_release ON economic_events(release_time DESC);
```

### 13. RLS Policies Configuration (Rule 1)
```sql
-- Enable Row Level Security on all tenant tables (including extended specs)
ALTER TABLE tenant_feature_flags ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE tenant_subscriptions ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE file_storage_metadata ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE outbox_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE idempotency_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing_invoices ENABLE ROW LEVEL SECURITY;
ALTER TABLE payment_transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolios ENABLE ROW LEVEL SECURITY;
ALTER TABLE portfolio_ledger_entries ENABLE ROW LEVEL SECURITY;
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
ALTER TABLE scanner_runs ENABLE ROW LEVEL SECURITY;
ALTER TABLE scanner_matches ENABLE ROW LEVEL SECURITY;
ALTER TABLE shap_explanations_history ENABLE ROW LEVEL SECURITY;
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
ALTER TABLE roles ENABLE ROW LEVEL SECURITY;
ALTER TABLE role_permissions ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_roles ENABLE ROW LEVEL SECURITY;

-- Dynamic isolation policies mapping (System-wide templates read allowed: tenant_id IS NULL)
CREATE POLICY feature_flags_isolation ON tenant_feature_flags FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY users_isolation ON users FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY profiles_isolation ON user_profiles FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY subscriptions_isolation ON tenant_subscriptions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY api_keys_isolation ON api_keys FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY sessions_isolation ON sessions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY files_isolation ON file_storage_metadata FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY audits_isolation ON audit_logs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY outbox_isolation ON outbox_events FOR ALL USING (tenant_id = get_current_tenant_id() OR tenant_id IS NULL);
CREATE POLICY idempotency_isolation ON idempotency_keys FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY invoices_isolation ON billing_invoices FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY payments_isolation ON payment_transactions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY portfolios_isolation ON portfolios FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY ledger_isolation ON portfolio_ledger_entries FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY holdings_isolation ON holdings FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY orders_isolation ON orders FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_accounts_isolation ON broker_accounts FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY broker_orders_isolation ON broker_orders FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY executions_isolation ON broker_trade_executions FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY watchlists_isolation ON watchlists FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY strategies_isolation ON strategies FOR ALL USING (tenant_id = get_current_tenant_id() OR tenant_id IS NULL);
CREATE POLICY runs_isolation ON strategy_runs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY backtests_isolation ON backtests FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY scanners_isolation ON scanners FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY scanner_runs_isolation ON scanner_runs FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY scanner_matches_isolation ON scanner_matches FOR ALL USING (tenant_id = get_current_tenant_id());
CREATE POLICY shap_history_isolation ON shap_explanations_history FOR ALL USING (tenant_id = get_current_tenant_id());
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
CREATE POLICY roles_isolation ON roles FOR ALL USING (tenant_id = get_current_tenant_id() OR tenant_id IS NULL);
CREATE POLICY role_permissions_isolation ON role_permissions FOR ALL USING (tenant_id = get_current_tenant_id() OR tenant_id IS NULL);
CREATE POLICY user_roles_isolation ON user_roles FOR ALL USING (tenant_id = get_current_tenant_id());
```

---

## Part 2: TimescaleDB Schema & Continuous Aggregates

Built in **TimescaleDB** using hypertable abstractions. We define the continuous aggregation roll-ups that translate raw tick variables into OHLCV views.

### 1. Base Time-Series Tables
```sql
CREATE TABLE market_ticks (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    symbol_id UUID NOT NULL,
    ltp_cents BIGINT NOT NULL CONSTRAINT check_tick_ltp CHECK (ltp_cents >= 0),
    ltq INT NOT NULL CONSTRAINT check_tick_ltq CHECK (ltq >= 0),
    atp_cents BIGINT NOT NULL CONSTRAINT check_tick_atp CHECK (atp_cents >= 0),
    total_volume BIGINT NOT NULL CONSTRAINT check_tick_vol CHECK (total_volume >= 0),
    open_interest BIGINT NOT NULL CONSTRAINT check_tick_oi CHECK (open_interest >= 0),
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

-- Real-Time Data Ingestion Quality Audit logs
CREATE TABLE data_quality_logs (
    time TIMESTAMP WITH TIME ZONE NOT NULL,
    source VARCHAR(63) NOT NULL, -- TICK_INGEST, NEWS_INGEST
    severity VARCHAR(15) NOT NULL, -- WARNING, CRITICAL
    error_code VARCHAR(63) NOT NULL,
    error_details JSONB NOT NULL
);
SELECT create_hypertable('data_quality_logs', 'time', chunk_time_interval => INTERVAL '7 days');
CREATE INDEX idx_data_quality_source ON data_quality_logs(source, time DESC);
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

### 3. Dead Letter Queue (DLQ) & Retry Topics

> **v1.5 Addition**: The following DLQ and retry topics MUST be created alongside primary topics. Without them, a single poison message (malformed tick, invalid JSON from a third-party webhook) will stall the entire consumer group partition, halting signal generation during a live trading session.

| Primary Topic | Retry Topic | DLQ Topic | Max Retries | Retry Backoff |
|---|---|---|---|---|
| `raw.market.ticks` | — | `raw.market.ticks.dlq` | 0 (no retry — stale ticks are worthless) | — |
| `processed.candles` | `processed.candles.retry` | `processed.candles.dlq` | 3 | 1s, 5s, 30s |
| `signals.final` | `signals.final.retry` | `signals.final.dlq` | 3 | 2s, 10s, 60s |
| `alerts.outbound` | `alerts.outbound.retry` | `alerts.outbound.dlq` | 5 | 1s, 5s, 15s, 60s, 300s |

**DLQ Consumer Contract**:
- A dedicated DLQ monitor service must poll all `.dlq` topics.
- Any message arriving in a DLQ must trigger a `CRITICAL` alert to the operations channel.
- DLQ messages must be retained for 7 days for manual inspection and replay.
- `raw.market.ticks.dlq` uses no retry — stale market data has zero recovery value. Log and discard.

**FUTURE ROADMAP**: At enterprise scale, introduce Kafka Streams exactly-once semantics (EOS) with `isolation.level=read_committed` on consumers, a Schema Registry for Avro contract enforcement, and automated DLQ replay pipelines. For v1.5 private deployment, manual DLQ inspection and replay via a CLI tool is sufficient.

---

## Part 6: Application-Layer Contracts

The following production concerns are **not** implemented as database triggers in v1.5 (appropriate for a private single-user deployment). They are documented here as **mandatory application-layer contracts** that every service touching these tables MUST implement. They are architected to be retrofittable as database-level triggers at a later stage without any schema changes.

---

### Contract 1: Optimistic Concurrency Control (OCC) for Portfolios & Holdings

**Applies to**: `portfolios`, `holdings`

**Problem**: Both tables carry a `version INT` column. Without enforced checks, concurrent transactions can overwrite each other's changes, causing double-spending of paper trading capital or incorrect holding quantities.

**Mandatory Pattern** (must be followed in `PortfolioService` and `HoldingsService`):
```python
# CORRECT — always include version predicate in UPDATE
rows_updated = db.execute(
    "UPDATE portfolios SET balance_cents = :new_balance, version = version + 1, "
    "updated_at = NOW() WHERE id = :id AND version = :expected_version",
    {"new_balance": new_balance, "id": portfolio_id, "expected_version": current_version}
).rowcount

if rows_updated == 0:
    raise OptimisticLockError("Portfolio was modified by another transaction. Retry.")
```
- Always read `version` in the same transaction as the SELECT that precedes an UPDATE.
- Always check `rowcount == 1` after UPDATE. If `rowcount == 0`, raise a retryable conflict error.
- The API layer must translate `OptimisticLockError` to HTTP 409 Conflict.
- **Never** issue a bare `UPDATE portfolios SET balance_cents = X WHERE id = Y` without the version check.

**FUTURE ROADMAP**: Retrofit as a `BEFORE UPDATE` trigger on `portfolios` and `holdings` that raises `SQLSTATE 40001` (serialization failure) when `NEW.version != OLD.version + 1`.

---

### Contract 2: Ledger Serialization for portfolio_ledger_entries

**Applies to**: `portfolio_ledger_entries`, `portfolios`

**Problem**: The ledger stores a running `balance_after_cents`. Two concurrent writes computing this value independently will produce an incorrect ledger — one row will reflect a wrong running balance, creating silent drift between the ledger and the actual portfolio balance.

**Mandatory Pattern** (must be followed in `LedgerService`):
```python
# Every ledger write MUST be wrapped in a transaction that locks the parent portfolio row
with db.transaction():
    # Row-level lock — blocks concurrent writes to the same portfolio
    portfolio = db.execute(
        "SELECT id, balance_cents, version FROM portfolios "
        "WHERE id = :id FOR UPDATE",
        {"id": portfolio_id}
    ).fetchone()

    new_balance = portfolio.balance_cents + amount_cents  # negative for debits
    if new_balance < 0:
        raise InsufficientFundsError()

    db.execute(
        "INSERT INTO portfolio_ledger_entries "
        "(tenant_id, portfolio_id, type, amount_cents, balance_after_cents, reference_id) "
        "VALUES (:tenant_id, :portfolio_id, :type, :amount, :new_balance, :ref)",
        {...}
    )
    db.execute(
        "UPDATE portfolios SET balance_cents = :new_balance, version = version + 1 "
        "WHERE id = :id AND version = :expected_version",
        {"new_balance": new_balance, "id": portfolio_id, "expected_version": portfolio.version}
    )
```
- The `SELECT ... FOR UPDATE` is the serialization gate. Never write a ledger entry outside this pattern.
- Both the INSERT into `portfolio_ledger_entries` and the UPDATE to `portfolios` must commit atomically.

**FUTURE ROADMAP**: Validate `balance_after_cents` consistency server-side via a scheduled reconciliation job that compares `portfolios.balance_cents` against `SUM(portfolio_ledger_entries.amount_cents)` per portfolio.

---

### Contract 3: Invoice Header Reconciliation

**Applies to**: `billing_invoices`, `billing_invoice_items`

**Problem**: `billing_invoices.amount_cents` (header total) is a stored column independent of `billing_invoice_items.amount_cents` (generated line totals). They can drift apart due to application bugs or partial rollbacks.

**Mandatory Pattern** (must be followed in `BillingService`):
```python
def finalise_invoice(invoice_id: UUID):
    line_total = db.execute(
        "SELECT COALESCE(SUM(amount_cents), 0) FROM billing_invoice_items "
        "WHERE invoice_id = :id", {"id": invoice_id}
    ).scalar()

    rows = db.execute(
        "UPDATE billing_invoices SET amount_cents = :total, status = 'UNPAID' "
        "WHERE id = :id AND status = 'DRAFT'",
        {"total": line_total, "id": invoice_id}
    ).rowcount

    if rows == 0:
        raise InvoiceStateError("Invoice is not in DRAFT state or was not found.")
```
- Invoices must go through a `DRAFT` state while line items are being added.
- `amount_cents` on the header must only be written by `finalise_invoice()` — never by individual item writers.
- Before marking an invoice `PAID`, assert `payment.amount_cents >= invoice.amount_cents`.

---

### Contract 4: Payment Amount Validation

**Applies to**: `payment_transactions`, `billing_invoices`

**Problem**: A payment webhook from Razorpay or Stripe can carry a manipulated or replayed `amount` that does not match the invoice amount. Recording it without validation silently activates a subscription that was not fully paid.

**Mandatory Pattern** (must be followed in `PaymentWebhookHandler`):
```python
def handle_payment_settled(external_txn_id: str, paid_amount_cents: int, invoice_id: UUID):
    # 1. Verify webhook HMAC signature BEFORE any DB operation
    verify_webhook_signature(request)  # raises if invalid

    # 2. Check idempotency — replay protection
    if db.exists("payment_transactions", external_transaction_id=external_txn_id):
        return  # already processed

    # 3. Load and validate the invoice
    invoice = db.get("billing_invoices", id=invoice_id)
    if paid_amount_cents < invoice.amount_cents:
        raise PaymentMismatchError(
            f"Underpayment: received {paid_amount_cents}, expected {invoice.amount_cents}"
        )

    # 4. Write payment record and update invoice atomically
    with db.transaction():
        db.insert("payment_transactions", {
            "invoice_id": invoice_id,
            "external_transaction_id": external_txn_id,
            "amount_cents": paid_amount_cents,
            "status": "SETTLED",
            ...
        })
        db.execute(
            "UPDATE billing_invoices SET status = 'PAID' WHERE id = :id",
            {"id": invoice_id}
        )
```
- HMAC verification must happen before any database read or write.
- Payment validation must happen before subscription activation.

---

### Contract 5: Redis Live Price Key TTL Management

**Applies to**: `market:price:{symbol_id}` Redis keys

**Problem**: Live price keys have no TTL. After market hours, keys for all 5,000+ symbols persist indefinitely. Over months, stale options/futures keys accumulate as new contract expiries are introduced, growing Redis memory monotonically until eviction pressure causes unpredictable key loss (potentially evicting sessions or rate limit counters).

**Mandatory Pattern** (must be followed in `MarketDataIngestionService` and `SchedulerService`):
```python
# On every tick write — set or refresh TTL to market close + 30 minutes
PIPELINE:
  HSET market:price:{symbol_id} ltp_cents {ltp} volume {vol} ts {ts}
  EXPIREAT market:price:{symbol_id} {market_close_unix + 1800}  # 17:00 IST

# Scheduled job at 15:55 IST — pre-emptively refresh TTL on all active keys
# to guard against clock skew or a missed tick EXPIREAT
for symbol_id in active_symbols:
    redis.expireat(f"market:price:{symbol_id}", next_market_close_timestamp + 1800)
```
- `active_symbols` is the set of all symbols with at least one active alert rule or watchlist entry.
- Redis `maxmemory-policy` must be set to `volatile-lru` so that non-expiring keys (sessions, rate limits) are protected from eviction when memory pressure occurs.
- Expired-key notification (keyspace events) should be disabled in production to avoid event flood.

**FUTURE ROADMAP**: At scale, replace ad-hoc TTL management with a Redis Streams-based tick fan-out architecture where live prices are never stored in Redis — they are streamed to WebSocket consumers directly from a Kafka consumer bridge.

---

## Part 7: Future Roadmap Items

The following architectural capabilities are intentionally deferred for the v1.5 private deployment. Each is documented with its trigger condition and implementation recommendation so it can be added without schema redesign.

| # | Capability | Trigger Condition | Implementation Recommendation |
|---|---|---|---|
| 1 | OCC Trigger Enforcement | When concurrent trading volume > 100 req/sec on portfolios | PostgreSQL `BEFORE UPDATE` trigger on `portfolios` and `holdings` raising `SQLSTATE 40001` on version mismatch |
| 2 | Immutable Audit Logs | When regulatory audit or SOC 2 compliance is required | Add `BEFORE DELETE` trigger on `audit_logs` that raises an exception unconditionally. Or migrate to AWS QLDB. |
| 3 | Invoice Reconciliation Trigger | When billing team size > 1 or automated invoice generation is introduced | `CONSTRAINT TRIGGER DEFERRABLE` on `billing_invoices` validating header vs. line item sum |
| 4 | Kafka EOS + Schema Registry | When consumer group lag > 5 seconds during market hours | Enable `isolation.level=read_committed`, deploy Confluent Schema Registry, add `.retry` topic consumers with exponential backoff |
| 5 | Redis Streams Tick Fan-out | When WebSocket client count > 1,000 | Replace `market:price:*` Hash keys with Redis Streams (`XADD market.ticks.{symbol_id}`). Subscribe clients via `XREAD` instead of polling. |
| 6 | Model FSM DB Trigger | When multiple engineers deploy models independently | PostgreSQL `BEFORE UPDATE` trigger enforcing the CANDIDATE → STAGING → PRODUCTION state machine |
| 7 | Multi-region Read Replicas | When read query latency > 100ms P99 | PostgreSQL logical replication to a read replica in the same region. TimescaleDB streaming replication for tick data. |
| 8 | Broker Token HSM Rotation | When broker OAuth tokens require compliance-grade key management | Integrate AWS KMS with automatic key rotation. Background job scans `broker_accounts` by `encryption_key_version`, re-encrypts rows on key change. |

---

*Phase 3C — Physical Database Schema Design v1.5*
*Status: Approved — Final Production Hardening*
*Deployment Context: Private single-user production. Forward-compatible for public SaaS evolution.*
*Next Phase: Phase 4 — API Specification Design*
