# Phase 3B — Entity Relationship Design (ERD)
## AI-Powered Indian Stock Market Intelligence SaaS Platform

> **Status**: Pending User Approval
> **Version**: 1.1 (Extended with AI Intelligence Entities)
> **Depends On**: Phase 3A.1 — Domain Model Extension (Approved)
> **Next Phase**: Phase 3C — Database Schema Design

---

## Part 1: Bounded Contexts (DDD Mapping)

To control complexity across our 59 business entities, we partition the platform into **8 Bounded Contexts**. Each context contains a clear aggregate root, strict transactional boundaries, and well-defined interfaces (APIs or events) for communicating across contexts.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       SaaS PLATFORM BOUNDED CONTEXTS                        │
│                                                                             │
│  ┌────────────────────────┐  ┌────────────────────────┐  ┌───────────────┐  │
│  │   Identity & Billing   │  │      Market Data       │  │   AI Engine   │  │
│  │   *Tenant (AR)         │  │   *Market Symbol (AR)  │  │ *ModelReg(AR) │  │
│  │   *User (AR)           │  │   *News (AR)           │  │               │  │
│  └───────────┬────────────┘  └───────────┬────────────┘  └───────┬───────┘  │
│              │                           │                       │          │
│              │ Authenticates             │ Ingests               │ Predicts │
│              ▼                           ▼                       ▼          │
│  ┌────────────────────────┐  ┌────────────────────────┐  ┌───────────────┐  │
│  │  Portfolio & Trading   │  │  Strategy & Backtest   │  │    Alerts     │  │
│  │   *Portfolio (AR)      │  │   *Strategy (AR)       │  │ *AlertRule(AR)│  │
│  │   *BrokerAccount (AR)  │  │   *Backtest (AR)       │  │               │  │
│  │                        │  │   *Scanner (AR)        │  │               │  │
│  └───────────┬────────────┘  └───────────┬────────────┘  └───────────────┘  │
│              │                           │                                  │
│              │ Governs                   │ Signals                          │
│              ▼                           ▼                                  │
│  ┌────────────────────────┐  ┌────────────────────────┐                     │
│  │    Risk Management     │  │     Signal Context     │                     │
│  │   *RiskRule (AR)       │  │   *TradeSignal (AR)    │ (AR) = Aggregate Root│
│  └────────────────────────┘  └────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Part 2: Detailed Relationship Specifications

Every entity relationship in the platform is specified below with type, ownership boundaries, deletion behaviors, multi-tenancy implications, and security controls.

---

### Identity & Billing Context Mappings

#### R1: Tenant ── User
- **Relationship**: 1:N (One Tenant to Many Users)
- **Ownership**: Tenant owns User.
- **Cascade Rules**: `ON DELETE CASCADE` (If a tenant organization is purged, all child user records are purged).
- **Multi-Tenant Considerations**: The root mapping. Determines the tenant partition for all user queries.
- **Security Implications**: Users cannot be assigned to more than one Tenant (strict hierarchical boundary).

#### R2: User ── User Profile
- **Relationship**: 1:1 (One User to One Profile)
- **Ownership**: User owns Profile.
- **Cascade Rules**: `ON DELETE CASCADE` (Deleting the User deletes their profile).
- **Multi-Tenant Considerations**: Inherits tenant context from parent User.
- **Security Implications**: Contains PII. Read/write strictly scoped to the owner and Tenant Admin.

#### R3: User ── Session
- **Relationship**: 1:N (One User to Many Active Sessions)
- **Ownership**: User owns Session.
- **Cascade Rules**: `ON DELETE CASCADE` (Deleting the user invalidates all sessions).
- **Multi-Tenant Considerations**: Sessions are strictly bounded within the tenant.
- **Security Implications**: Session revocation must immediately purge child session keys in the database and cache.

#### R4: User ── API Key
- **Relationship**: 1:N (One User to Many API Keys)
- **Ownership**: User owns API Key.
- **Cascade Rules**: `ON DELETE CASCADE`
- **Multi-Tenant Considerations**: Inherits tenant isolation of the user.
- **Security Implications**: API keys must be hashed in storage. Revocation invalidates corresponding access paths.

#### R5: Role ── Permission
- **Relationship**: N:N (Many Roles to Many Permissions)
- **Ownership**: System/Tenant owned.
- **Cascade Rules**: `ON DELETE RESTRICT` (Permissions cannot be deleted if active in roles; role deletion removes map records).
- **Multi-Tenant Considerations**: Default roles are shared across tenants; custom roles mapped to specific permissions are isolated to the tenant.
- **Security Implications**: Access control maps directly to these permissions at the API Gateway.

#### R6: Subscription Plan ── Tenant
- **Relationship**: 1:N (One Plan to Many Tenants)
- **Ownership**: System Configuration.
- **Cascade Rules**: `ON DELETE RESTRICT` (Plan cannot be deleted if a tenant holds it).
- **Multi-Tenant Considerations**: Multi-tenant feature gates evaluate against this plan mapping.
- **Security Implications**: Hard enforcement at gateway prevents tenant feature escalation.

---

### Portfolio & Trading Context Mappings

#### R7: User ── Portfolio
- **Relationship**: 1:N (One User to Many Portfolios)
- **Ownership**: User owns Portfolio.
- **Cascade Rules**: `ON DELETE CASCADE` (Or soft-delete via flag).
- **Multi-Tenant Considerations**: Portfolio explicitly carries `tenant_id` alongside `user_id` for dual query scoping.
- **Security Implications**: Portfolio views require owning-user credentials or Tenant Admin delegation.

#### R8: Portfolio ── Holding
- **Relationship**: 1:N (One Portfolio to Many Holdings)
- **Ownership**: Portfolio owns Holding.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Holding queries filter strictly through Portfolio tenant scope.
- **Security Implications**: Cost-basis updates must perform consistent FIFO calculations on transaction modifications.

#### R9: Portfolio ── Order
- **Relationship**: 1:N (One Portfolio to Many Orders)
- **Ownership**: Portfolio owns Order.
- **Cascade Rules**: `ON DELETE RESTRICT` (Orders must be retained for audit trails).
- **Multi-Tenant Considerations**: Explicit tenant scoping ensures transaction audits are secured by tenant boundaries.
- **Security Implications**: Trade executions are validated against portfolio funds to prevent negative balances.

#### R10: Market Symbol ── Holding
- **Relationship**: 1:N (One Symbol to Many Portfolio Holdings)
- **Ownership**: Reference relationship (System to User data).
- **Cascade Rules**: `ON DELETE RESTRICT` (A symbol cannot be deleted from the system master if held in any portfolio).
- **Multi-Tenant Considerations**: Global system entity mapped across multiple tenant portfolios.
- **Security Implications**: Read-only validation ensures symbols cannot be manipulated in portfolio calculations.

#### R11: User ── Watchlist
- **Relationship**: 1:N (One User to Many Watchlists)
- **Ownership**: User owns Watchlist.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Watchlists are isolated by tenant boundaries.
- **Security Implications**: Users can edit only their watchlists.

#### R12: Watchlist ── Watchlist Item
- **Relationship**: 1:N (One Watchlist to Many Items)
- **Ownership**: Watchlist owns Item.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Strictly bound to parent watchlist tenant.
- **Security Implications**: Symbol references are verified against the active Symbol Master.

---

### Alerts & Notifications Context Mappings

#### R13: User ── Alert Rule
- **Relationship**: 1:N (One User to Many Alert Rules)
- **Ownership**: User owns Alert Rule.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Rule creation limits (quota check) are enforced at tenant plan tier level.
- **Security Implications**: Prevents resource exhaustion from duplicate alert rules.

#### R14: Alert Rule ── Alert Event
- **Relationship**: 1:N (One Rule to Many Historical Alert Events)
- **Ownership**: Event Log.
- **Cascade Rules**: `ON DELETE SET NULL` (Deleting a rule preserves alert logs for user tracking).
- **Multi-Tenant Considerations**: Mapped under tenant context.
- **Security Implications**: History read paths are restricted to the alert owner.

#### R15: Alert Event ── Notification Log
- **Relationship**: 1:N (One Event to Many Notification logs)
- **Ownership**: Alert Event owns Log.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Tenant log isolation.
- **Security Implications**: Logs must omit user-sensitive routing parameters (phone numbers/emails).

---

### Market Data Context Mappings

#### R16: Exchange ── Market Symbol
- **Relationship**: 1:N (One Exchange to Many Symbols)
- **Ownership**: System Reference.
- **Cascade Rules**: `ON DELETE RESTRICT`.
- **Multi-Tenant Considerations**: Global configuration available to all tenants.
- **Security Implications**: System-wide updates are protected behind Admin authentication.

#### R17: Market Symbol ── Historical Price Data
- **Relationship**: 1:N (One Symbol to Many OHLCV records)
- **Ownership**: System Reference.
- **Cascade Rules**: `ON DELETE CASCADE` (Delisting clean-up).
- **Multi-Tenant Considerations**: Shared data source across all tenant applications.
- **Security Implications**: Read-only paths.

#### R18: Market Symbol ── Technical Indicator
- **Relationship**: 1:N (One Symbol to Many calculated indicators)
- **Ownership**: Computed System Data.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Shared cache data.
- **Security Implications**: Cached indicators protected from modification.

---

### AI Engine Context Mappings

#### R19: Model Version ── AI Prediction
- **Relationship**: 1:N (One Model Version to Many Predictions)
- **Ownership**: System data.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Global prediction stream.
- **Security Implications**: Target predictions must be digitally signed to confirm authenticity.

#### R20: AI Prediction ── SHAP Explanation
- **Relationship**: 1:1 (One Prediction to One SHAP Explanation)
- **Ownership**: System data.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Shared context.
- **Security Implications**: Explanation payloads must match prediction vectors.

#### R20.1: Feature Store Version ── AI Feature
- **Relationship**: 1:N (One Feature Store Version to Many AI Features)
- **Ownership**: System configuration mapping.
- **Cascade Rules**: `ON DELETE RESTRICT`.
- **Multi-Tenant Considerations**: Shared global feature mappings.
- **Security Implications**: Schema constraints enforce data completeness.

#### R20.2: Model Version ── Model Monitoring
- **Relationship**: 1:N (One Model Version to Many Monitoring Logs)
- **Ownership**: System log.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: System-wide logging metrics.
- **Security Implications**: Read access limited to system administrators.

#### R20.3: Model Version ── Model Drift
- **Relationship**: 1:N (One Model Version to Many Drift Calculations)
- **Ownership**: System validation data.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Global operational alert limits.
- **Security Implications**: Drift indicators trigger model hot-swapping or retraining loops.

#### R20.4: Model Version ── Confidence Calibration
- **Relationship**: 1:1 (One Model Version to One Confidence Calibration Matrix)
- **Ownership**: System configuration data.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Shared prediction calibration across all tenants.
- **Security Implications**: Calibration weights must be validated against tampering.

#### R20.5: AI Prompt Library ── SHAP Explanation
- **Relationship**: 1:N (One Prompt Configuration to Many Generated SHAP Explanations)
- **Ownership**: System template structure.
- **Cascade Rules**: `ON DELETE RESTRICT` (Prompts in active use cannot be deleted).
- **Multi-Tenant Considerations**: Shared standard templates; custom prompts isolated per tenant.
- **Security Implications**: Escapes few-shot templates to mitigate prompt injection risks.

---

### Trade Signal Context Mappings (Extended Domain)

#### R20.6: AI Prediction ── Trade Signal
- **Relationship**: 1:N (One Prediction can yield Multiple Trade Signals across timeframes)
- **Ownership**: System generated data.
- **Cascade Rules**: `ON DELETE CASCADE` (Or retention-bound pruning).
- **Multi-Tenant Considerations**: Consumed by tenant rules to execute localized portfolio checks.
- **Security Implications**: Verification signature must be attached to the Trade Signal structure.

#### R20.7: Trade Signal ── Signal Performance
- **Relationship**: 1:1 (One Signal to One Performance outcome)
- **Ownership**: System audited ledger.
- **Cascade Rules**: `ON DELETE RESTRICT` (Performance calculations must be kept as historical ledger).
- **Multi-Tenant Considerations**: Shared metrics globally referenced.
- **Security Implications**: Read-only validation rules enforce performance metrics consistency.

#### R20.8: Trade Signal ── AI Feedback
- **Relationship**: 1:N (One Signal to Many User Feedbacks)
- **Ownership**: User owns Feedback.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Isolated by User and Tenant scopes.
- **Security Implications**: Retraining jobs read feedback vectors without loading user PII.

#### R20.9: User ── Scanner
- **Relationship**: 1:N (One User to Many Custom Scanners)
- **Ownership**: User owns Scanner.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Strictly isolated by Tenant context limits.
- **Security Implications**: Resource-bound queries prevent execution timeouts.

#### R20.10: Scanner ── Scanner Result
- **Relationship**: 1:N (One Scanner to Many Results)
- **Ownership**: Ephemeral transaction result.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Inherits scanner context isolation.
- **Security Implications**: Fast eviction strategies in Redis cache prevent memory leak vulnerabilities.

---

### Trading & Broker Context Mappings

#### R21: Broker Integration ── Broker Account
- **Relationship**: 1:N (One Integration to Many Linked Accounts)
- **Ownership**: Config to user mapping.
- **Cascade Rules**: `ON DELETE RESTRICT` (Integration definition cannot be removed if active accounts exist).
- **Multi-Tenant Considerations**: Shared config, isolated user accounts.
- **Security Implications**: Integrations are read-only for standard users.

#### R22: User ── Broker Account
- **Relationship**: 1:N (One User to Many Broker Accounts)
- **Ownership**: User owns Broker Account.
- **Cascade Rules**: `ON DELETE CASCADE` (Unlinking removes tokens).
- **Multi-Tenant Considerations**: Isolated strictly under Tenant scope.
- **Security Implications**: OAuth credentials must be stored double-encrypted.

#### R23: Broker Account ── Broker Order
- **Relationship**: 1:N (One Broker Account to Many Broker Orders)
- **Ownership**: Broker Account owns Broker Order.
- **Cascade Rules**: `ON DELETE RESTRICT` (Ledger integrity).
- **Multi-Tenant Considerations**: Strictly isolated by Tenant context.
- **Security Implications**: Actions require audit records matching live orders.

---

### Backtesting & Strategy Context Mappings

#### R24: User ── Strategy
- **Relationship**: 1:N (One User to Many Custom Strategies)
- **Ownership**: User owns Strategy.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Custom strategy parameters are isolated.
- **Security Implications**: Code sandbox execution prevents resource leaks.

#### R25: Strategy ── Strategy Run
- **Relationship**: 1:N (One Strategy to Many Run instances)
- **Ownership**: Strategy owns Run.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Scoped to the tenant database.
- **Security Implications**: Restricts concurrency of runs.

---

### Risk Management Context Mappings

#### R26: Portfolio ── Risk Event
- **Relationship**: 1:N (One Portfolio to Many Risk Events)
- **Ownership**: Portfolio owns Risk Event.
- **Cascade Rules**: `ON DELETE CASCADE`.
- **Multi-Tenant Considerations**: Tenant-isolated log.
- **Security Implications**: Immutable risk log.

---

### Infrastructure Context Mappings

#### R27: Tenant ── Audit Log
- **Relationship**: 1:N (One Tenant to Many Audit Logs)
- **Ownership**: Tenant owns Audit Log.
- **Cascade Rules**: `ON DELETE RESTRICT` (Audit logs are read-only and immutable; cannot be deleted during operation).
- **Multi-Tenant Considerations**: Isolated admin trail per tenant.
- **Security Implications**: Write-once, read-many validation.

---

## Part 3: High-Level ER Diagram (ASCII)

```
       ┌───────────────────────┐             ┌───────────────────────┐
       │   Subscription Plan   │             │        Tenant         │
       └───────────┬───────────┘             └───────────┬───────────┘
                   │ 1                                   │ 1
                   │                                     │
                   │ N                                   ├───────────────────────┐
         ┌─────────▼───────────┐                         │                       │
         │   Tenant Contract   │                         │ N                     │ N
         └─────────┬───────────┘               ┌─────────▼───────────┐ ┌─────────▼───────────┐
                   │ 1                         │        User         │ │     Audit Log       │
                   │                           └────┬────────────┬───┘ └─────────────────────┘
                   │ N                              │ 1          │ 1
         ┌─────────▼───────────┐          1:N       │            │
         │   Billing Invoice   │◀───────────────────┤            │ 1:1
         └─────────┬───────────┘                    │            │
                   │ 1                              │ N          └─────────┐
                   │                                │                      │
                   │ N                              │            ┌─────────▼───────────┐
         ┌─────────▼───────────┐                    │            │    User Profile     │
         │ Payment Transaction │                    │            └─────────────────────┘
         └─────────────────────┘                    │
                                                    │
                 ┌──────────────────────────────────┴──────────────────────────────────┐
                 │                                                                     │
                 │ N                                                                   │ N
       ┌─────────▼───────────┐                                               ┌─────────▼───────────┐
       │     Portfolio       │                                               │    Watchlist        │
       └────┬────────────┬───┘                                               └─────────┬───────────┘
            │ 1          │ 1                                                           │ 1
            │            │                                                             │
            │ N          │ N                                                           │ N
  ┌─────────▼─────┐  ┌───▼───────────┐                                       ┌─────────▼───────────┐
  │   Holding     │  │    Order      │                                       │   Watchlist Item    │
  └─────────▲─────┘  └───┬───────────┘                                       └─────────▲───────────┘
            │            │ 1                                                           │
            │            │                                                             │
            │            │ N                                                           │
            │            │                                                             │
            │            │                                                             │
            │            │                                                             │
            │ 1          │ 1                                                           │ 1
  ┌─────────┴────────────┴─────────────────────────────────────────────────────────────┴───────────┐
  │                                       Market Symbol                                            │
  └─────────────────────────────────────────────▲──────────────────────────────────────────────────┘
                                                │ 1
                                                │ N
                                     ┌──────────┴──────────┐
                                     │   Historical Price  │
                                     └─────────────────────┘
```

---

## Part 4: Domain-wise ER Diagrams (ASCII)

### 1. Identity & Access Context

```
   ┌──────────────┐
   │    Tenant    │
   └──────┬───────┘
          │ 1
          │
          │ N
   ┌──────▼───────┐ 1           1:1 ┌──────────────┐
   │    User      ├────────────────▶│ User Profile │
   └──┬────────┬──┘                 └──────────────┘
      │ 1      │ 1
      │        │
      │ N      │ N
┌─────▼────┐ ┌─▼────────┐
│ Session  │ │ API Key  │
└──────────┘ └──────────┘
```

### 2. AI & Model Monitoring Context (Extended)

```
       ┌────────────────┐
       │ Model Registry │
       └───────┬────────┘
               │ 1
               │
               │ N
       ┌───────▼────────┐
       │    AI Model    │
       └───────┬────────┘
               │ 1
               │
               │ N
       ┌───────▼────────┐           ┌──────────────────────┐
       │ Model Version  │◀──────────┤ FeatureStore Version │
       └─┬───┬───┬───┬──┘           └──────────────────────┘
         │   │   │   │
         │   │   │   └───────────────┐ 1:1
         │   │   │                   ▼
         │   │   │          ┌──────────────────────┐
         │   │   │          │Confidence Calibration│
         │   │   │          └──────────────────────┘
         │   │   │
         │   │   └───────────┐ 1:N
         │   │               ▼
         │   │      ┌──────────────────────┐
         │   │      │   Model Monitoring   │
         │   │      └──────────┬───────────┘
         │   │                 │ 1
         │   │                 │
         │   │                 │ N
         │   │      ┌──────────▼───────────┐
         │   └─────▶│     Model Drift      │
         │          └──────────────────────┘
         │
         │ 1:N
         ▼
 ┌──────────────┐ 1      1:1 ┌──────────────┐
 │ AI Prediction├───────────▶│ SHAP Explan. │
 └───────┬──────┘            └───────▲──────┘
         │ 1                         │ N
         │                           │
         │ N                         │ 1
 ┌───────▼──────┐           ┌────────┴──────┐
 │ Trade Signal ├──────────▶│Prompt Library │
 └──────────────┘           └───────────────┘
```

### 3. Trade Signal & Scanner Performance Context (Extended)

```
        ┌──────────────┐             ┌──────────────┐
        │     User     │             │    Tenant    │
        └─┬──────────┬─┘             └─┬──────────┬─┘
          │          │                 │          │
          │ 1:N      └────────┐        │ 1:N      │
          ▼                   ▼        ▼          │
 ┌────────────────┐ 1:N ┌──────────────┐          │
 │    Scanner     ├────▶│ ScannerResult│          │
 └────────┬───────┘     └────────▲─────┘          │
          │                      │                │
          └──────────────────────┼────────────────┘
                                 │ 1:N
                                 ▼
                        ┌────────────────┐
                        │ Market Symbol  │
                        └────────▲───────┘
                                 │ 1:N
                                 ▼
                        ┌────────────────┐ 1    1:1 ┌─────────────────┐
                        │  Trade Signal  ├─────────▶│SignalPerformace │
                        └────────┬───────┘          └─────────────────┘
                                 │ 1
                                 │
                                 │ N
                        ┌────────▼───────┐
                        │  AI Feedback   │
                        └────────────────┘
```

---

## Part 5: Relationship Summary Table (Extended)

| Parent Entity | Child Entity | Type | Cascade Rule | Multi-tenant Constraint | Security Control |
|---|---|---|---|---|---|
| `Tenant` | `User` | 1:N | `ON DELETE CASCADE` | Isolated under tenant ID | Bound to tenant domain |
| `User` | `User Profile` | 1:1 | `ON DELETE CASCADE` | Inherits parent scope | PII access restrictions |
| `User` | `Session` | 1:N | `ON DELETE CASCADE` | Strict local validate | Auto-invalidation on pw reset |
| `User` | `API Key` | 1:N | `ON DELETE CASCADE` | Inherits parent scope | SHA-256 secure hash checks |
| `Subscription` | `Tenant` | 1:N | `ON DELETE RESTRICT` | Multi-tenant limit check | Plan gates verified at gateway |
| `Tenant` | `Billing Invoice` | 1:N | `ON DELETE RESTRICT` | Mapped to billing ID | Cross-tenant block enforced |
| `User` | `Portfolio` | 1:N | `ON DELETE CASCADE` | Scoped by tenant + user | Owner-only credentials |
| `Portfolio` | `Holding` | 1:N | `ON DELETE CASCADE` | Inherited from Portfolio | Consistent cost computation |
| `Portfolio` | `Order` | 1:N | `ON DELETE RESTRICT` | Strict audit trail | Balance validation checks |
| `User` | `Watchlist` | 1:N | `ON DELETE CASCADE` | Scoped to active user | Owner-only modifications |
| `Watchlist` | `Watchlist Item` | 1:N | `ON DELETE CASCADE` | Scoped to parent watchlist | Symbol registry validity checks |
| `User` | `Alert Rule` | 1:N | `ON DELETE CASCADE` | Limit quotas per plan | Rule creation limits |
| `Alert Rule` | `Alert Event` | 1:N | `ON DELETE SET NULL` | Mapped under tenant ID | Scope rules |
| `Alert Event` | `Notification Log` | 1:N | `ON DELETE CASCADE` | Log scoping | Suppress routing PII |
| `Exchange` | `Market Symbol` | 1:N | `ON DELETE RESTRICT` | Global data | Admin write authorization |
| `Market Symbol` | `Historical Price` | 1:N | `ON DELETE CASCADE` | Global access | Read-only |
| `Market Symbol` | `Technical Ind.` | 1:N | `ON DELETE CASCADE` | Read access | Read-only |
| `Model Version` | `AI Prediction` | 1:N | `ON DELETE CASCADE` | Global access | Cryptographic sig verification |
| `AI Prediction` | `SHAP Explanation`| 1:1 | `ON DELETE CASCADE` | Mapped payload | Vector checks |
| `User` | `Broker Account` | 1:N | `ON DELETE CASCADE` | Isolated under Tenant | Double-encrypted credentials |
| `Broker Account`| `Broker Order` | 1:N | `ON DELETE RESTRICT` | Scoped to Tenant | Strict audit alignment |
| `User` | `Strategy` | 1:N | `ON DELETE CASCADE` | User-isolated scope | Sandbox execution checks |
| `Strategy` | `Strategy Run` | 1:N | `ON DELETE CASCADE` | Scoped to tenant | Run limit checks |
| `Portfolio` | `Risk Event` | 1:N | `ON DELETE CASCADE` | Tenant-scoped log | Immutable write logs |
| `Tenant` | `Audit Log` | 1:N | `ON DELETE RESTRICT` | Tenant logs | Write-once read-many |
| `FeatureStore Ver`| `AI Feature` | 1:N | `ON DELETE RESTRICT` | Shared schema | Version check constraints |
| `Model Version` | `Model Monitoring` | 1:N | `ON DELETE CASCADE` | Admin logging only | Restricted admin views |
| `Model Version` | `Model Drift` | 1:N | `ON DELETE CASCADE` | Global state checks | Auto retraining trigger hooks |
| `Model Version` | `Conf. Calibration`| 1:1 | `ON DELETE CASCADE` | Shared coefficients | Read-only access |
| `AI Prompt Lib` | `SHAP Explanation`| 1:N | `ON DELETE RESTRICT` | Template versioning | Strict sanitization checks |
| `AI Prediction` | `Trade Signal` | 1:N | `ON DELETE CASCADE` | Tenant evaluation checks| Hash signatures enforced |
| `Trade Signal` | `Signal Performance`| 1:1 | `ON DELETE RESTRICT` | Shared stat reference | Read-only logs |
| `Trade Signal` | `AI Feedback` | 1:N | `ON DELETE CASCADE` | Scoped to active User | PII scrub runs |
| `User` | `Scanner` | 1:N | `ON DELETE CASCADE` | Scoped to Tenant quotas | Rate limit controls |
| `Scanner` | `Scanner Result` | 1:N | `ON DELETE CASCADE` | Inherited context | Fast Redis eviction |

---

*Phase 3B — Extended Entity Relationship Design v1.1*
*Status: Pending User Review and Approval*
*Next Phase: Phase 3C — Database Schema Design (awaiting approval)*
