# Contributing to the AI-Powered Indian Stock Market Intelligence Platform

Thank you for your interest in contributing. This is an enterprise-grade, production-focused project. We hold contributions to a high standard — quality, security, and maintainability are non-negotiable.

Please read this guide completely before opening issues or submitting pull requests.

---

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [Development Philosophy](#development-philosophy)
3. [How to Contribute](#how-to-contribute)
4. [Branch Strategy](#branch-strategy)
5. [Commit Message Standard](#commit-message-standard)
6. [Code Standards](#code-standards)
7. [Testing Requirements](#testing-requirements)
8. [Pull Request Process](#pull-request-process)
9. [Security Vulnerability Reporting](#security-vulnerability-reporting)
10. [Architecture Decision Records](#architecture-decision-records)

---

## Code of Conduct

This project operates under a professional, respectful working environment. We do not tolerate harassment, discrimination, or unprofessional conduct. Contributors are expected to review and adhere to these norms.

---

## Development Philosophy

Before writing any code, internalize these principles:

| Principle | What It Means |
|---|---|
| **Clean over fast** | Readable, maintainable code is prioritized over clever optimizations |
| **Modular** | Every module has a single, clear responsibility |
| **Tested** | No feature lands without unit and integration tests |
| **Documented** | Complex logic requires docstrings, architectural changes require ADRs |
| **Secure by default** | Never commit secrets; validate all inputs; follow least privilege |
| **No magic** | Avoid implicit behavior; prefer explicit configuration |

---

## How to Contribute

### Step 1 — Prerequisites

Ensure you have the following installed:
- Python 3.12+
- Node.js 20+ and npm 10+
- Docker Desktop
- Git 2.40+
- Make (for Makefile commands)

### Step 2 — Fork and Clone

```bash
# Fork the repository on GitHub, then:
git clone https://github.com/YOUR_USERNAME/market-intelligence-platform.git
cd market-intelligence-platform
```

### Step 3 — Set Up Development Environment

```bash
# Backend (Python)
cd backend/<service_name>
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
# OR
.venv\Scripts\activate     # Windows

pip install -r requirements.txt
pip install -r requirements-dev.txt

# Frontend (Web)
cd frontend/web
npm install

# Frontend (Mobile)
cd frontend/mobile
npm install
```

### Step 4 — Create a Branch

See [Branch Strategy](#branch-strategy) below.

### Step 5 — Make Your Changes

Follow the [Code Standards](#code-standards) in this guide.

### Step 6 — Test Your Changes

See [Testing Requirements](#testing-requirements).

### Step 7 — Submit a Pull Request

See [Pull Request Process](#pull-request-process).

---

## Branch Strategy

We follow **GitFlow** with the following branch conventions:

| Branch Pattern | Purpose | Example |
|---|---|---|
| `main` | Production-ready code only. Protected. | — |
| `develop` | Integration branch. All features merge here. | — |
| `feature/<ticket>-<description>` | New feature development | `feature/MIP-42-alert-engine` |
| `fix/<ticket>-<description>` | Bug fixes | `fix/MIP-77-kafka-consumer-lag` |
| `hotfix/<ticket>-<description>` | Critical production fixes | `hotfix/MIP-99-auth-bypass` |
| `release/<version>` | Release preparation | `release/1.0.0` |
| `docs/<description>` | Documentation only changes | `docs/update-api-spec` |
| `chore/<description>` | Tooling, CI, dependencies | `chore/upgrade-fastapi-0.110` |

**Rules**:
- Never commit directly to `main` or `develop`
- Every branch must be linked to a GitHub Issue or ticket
- Branch names must be lowercase with hyphens (no underscores, no spaces)
- Delete branches after merge

---

## Commit Message Standard

We enforce [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body]

[optional footer(s)]
```

### Types

| Type | When to Use |
|---|---|
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation changes only |
| `style` | Formatting (no logic change) |
| `refactor` | Code restructuring (no feature/fix) |
| `perf` | Performance improvement |
| `test` | Adding or updating tests |
| `build` | Build system or dependency changes |
| `ci` | CI/CD configuration changes |
| `chore` | Routine tasks (cleanup, tooling) |
| `revert` | Reverting a previous commit |
| `security` | Security-related fixes |

### Scopes

Use the service or module name as scope:

`auth`, `market-data`, `processing`, `strategy`, `ai-engine`, `signals`,
`portfolio`, `alerts`, `notifications`, `backtesting`, `paper-trading`,
`analytics`, `frontend-web`, `frontend-mobile`, `infra`, `docs`, `ci`

### Examples

```
feat(alert-engine): add CEP rule matching for price alerts

fix(market-data): handle WebSocket reconnection on vendor timeout

docs(architecture): add Phase 2B data flow diagram

perf(strategy): reduce indicator computation time by 40% using vectorization

security(auth): enforce RS256 algorithm on all JWT validation paths

ci(github-actions): add Trivy container security scan on PR merge

BREAKING CHANGE: MarketTickEvent schema v2 removes deprecated `ltp_time` field
```

### Rules
- Summary line must be ≤ 72 characters
- Use imperative mood ("add", not "added" or "adds")
- Do NOT capitalize the first word of the summary
- Do NOT end with a period
- Reference issues in the footer: `Closes #42` or `Fixes #77`

---

## Code Standards

### Python

| Tool | Purpose | Config |
|---|---|---|
| `ruff` | Linting + import sorting | `pyproject.toml` |
| `black` | Code formatting (88 char line length) | `pyproject.toml` |
| `mypy` | Static type checking (strict mode) | `mypy.ini` |
| `bandit` | Security vulnerability scanning | `.bandit` |

**Rules**:
- All functions must have type hints (enforced by mypy strict)
- All public functions must have docstrings (Google style)
- No bare `except:` clauses — always catch specific exceptions
- No `print()` statements — use the structured logger
- All secrets via environment variables — never hardcoded
- Maximum function complexity: 10 (cyclomatic complexity — enforced by ruff)
- Maximum file length: 500 lines — split into modules if exceeded

### TypeScript / JavaScript

| Tool | Purpose | Config |
|---|---|---|
| `eslint` | Linting | `.eslintrc.json` |
| `prettier` | Code formatting | `.prettierrc` |
| `tsc` | TypeScript compilation check | `tsconfig.json` |

**Rules**:
- `strict: true` in all `tsconfig.json` files
- No `any` types without explicit justification comment
- All React components must be functional (no class components)
- Props must be typed with interfaces (not inline types for reusable components)
- No direct DOM manipulation (use React refs)

### General Rules (All Languages)

- No magic numbers — use named constants
- No dead code
- Files must end with a newline
- No trailing whitespace
- UTF-8 encoding only

---

## Testing Requirements

**No PR is merged without adequate tests.**

| Test Type | Location | Tool | Minimum Requirement |
|---|---|---|---|
| Unit Tests | `tests/unit/` | pytest (Python), Jest (JS) | 80% coverage on changed code |
| Integration Tests | `tests/integration/` | pytest + testcontainers | All new API endpoints |
| E2E Tests | `tests/e2e/` | Playwright | Critical user flows |
| Load Tests | `tests/load/` | Locust | New services: baseline load test |

**Running Tests**:
```bash
# Python unit tests
pytest tests/unit/ -v --cov=app --cov-report=term-missing

# Python integration tests (requires Docker)
pytest tests/integration/ -v

# Frontend tests
cd frontend/web && npm test

# Load tests
locust -f tests/load/locustfile.py --host=http://localhost:8000
```

**Test Quality Rules**:
- Tests must be deterministic (no flaky tests)
- No test should depend on execution order
- Use fixtures for shared setup, not `setUp` anti-patterns
- Mock external dependencies (Kafka, databases) in unit tests
- Integration tests use real services via Docker Compose (`docker-compose.test.yml`)
- Test names must describe the scenario: `test_jwt_expired_token_returns_401()`

---

## Pull Request Process

### PR Checklist (Self-Review Before Submitting)

```
[ ] My branch is up to date with develop (rebased, not merged)
[ ] All tests pass locally (pytest, npm test)
[ ] Code coverage has not decreased
[ ] No new linting errors (ruff, eslint)
[ ] Type checks pass (mypy, tsc)
[ ] Security scan passes (bandit)
[ ] No secrets, credentials, or PII in the diff
[ ] API changes are reflected in OpenAPI spec
[ ] Database migrations are backward-compatible
[ ] CHANGELOG.md updated under [Unreleased]
[ ] PR title follows Conventional Commits format
[ ] PR description explains what changed and WHY
[ ] Linked to the relevant GitHub Issue
```

### PR Size Guidelines

| Size | Lines Changed | Policy |
|---|---|---|
| XS | < 50 | Fast-track review |
| S | 50–200 | Standard review |
| M | 200–500 | Requires detailed description |
| L | 500–1000 | Must be split if possible |
| XL | > 1000 | Requires architecture review before coding |

### Review Requirements

- **Minimum 1 reviewer** for S/M PRs
- **Minimum 2 reviewers** for L/XL PRs (one must be a senior engineer)
- **Mandatory security review** for any changes to auth, secrets handling, or broker integration
- All CI checks must be green before merge
- No self-approval

### Merge Policy

- Squash and merge for feature branches (clean history on develop)
- Merge commit for release branches to main (preserves release history)
- Always delete the source branch after merge

---

## Security Vulnerability Reporting

**Do NOT open a public GitHub Issue for security vulnerabilities.**

If you discover a security vulnerability:

1. **Email**: security@marketintelligence.in (placeholder — update before launch)
2. **Include**: Description of the vulnerability, reproduction steps, potential impact
3. **Response**: You will receive acknowledgment within 24 hours and a fix timeline within 72 hours
4. **Disclosure**: We follow responsible disclosure — vulnerabilities are not publicized until patched

We appreciate and recognize all responsible security disclosures.

---

## Architecture Decision Records (ADRs)

All significant architectural decisions must be documented as ADRs.

**When is an ADR required?**
- Introducing a new technology to the stack
- Changing a core design pattern
- Making a decision that is difficult to reverse
- Choosing between two viable alternatives

**ADR Template Location**: `docs/Decisions/ADR-TEMPLATE.md`

**ADR Naming**: `docs/Decisions/ADR-XXXX-short-title.md`
(e.g., `docs/Decisions/ADR-0001-use-kafka-as-message-broker.md`)

**ADR Status Values**: `Proposed` → `Accepted` → `Deprecated` → `Superseded`

---

## Questions?

For development questions, open a GitHub Discussion (not an Issue).
For urgent concerns, reach out via the project's internal communication channel.

---

*Thank you for helping build a world-class platform. Quality over speed, always.*
