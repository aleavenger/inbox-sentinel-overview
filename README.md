# Inbox Sentinel

**AI-powered email classification and automation system.**

> Self-hosted, multi-tenant email intelligence platform with hybrid deterministic + AI classification, cost guards, reversible mailbox automation, and Stripe billing.

---

## Overview

Inbox Sentinel intelligently organizes incoming emails by combining a rules-based classification engine with AI fallback (OpenAI). The deterministic-first approach minimizes AI costs while maintaining high accuracy. All mailbox operations are reversible, and the system includes multi-layer cost protection to prevent runaway API spend.

### Key Capabilities

- **Hybrid Classification** — Rules engine runs first (keywords, sender domains, blacklist/whitelist); AI fallback only when rules don't match
- **Cost Guard Protection** — Global and per-user daily spend limits prevent runaway OpenAI costs
- **Reversible Automation** — All email moves generate undo tokens (24h expiry)
- **Multi-Provider** — Microsoft Outlook/Exchange, Gmail, Yahoo Mail
- **Self-Hosted** — Single Docker container deployment
- **SaaS-Ready** — Stripe billing, multi-tenant isolation, feature flags

---

## Architecture

```
                    ┌─────────────────────────────┐
                    │     Email Providers          │
                    │  (Outlook / Gmail / Yahoo)   │
                    └──────────────┬──────────────┘
                                   │
                            OAuth + Graph API
                                   │
┌──────────────────────────────────▼──────────────────────────────────┐
│                     Classification Pipeline                         │
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────────────┐  │
│  │ Sender   │──▶│  Rule    │──▶│  Memory  │──▶│  Cost Guard    │  │
│  │ Learning │   │  Engine  │   │  Cache   │   │  (limits check)│  │
│  └──────────┘   └──────────┘   └──────────┘   └───────┬────────┘  │
│                                                        │           │
│                                                ┌───────▼────────┐  │
│                                                │  OpenAI API    │  │
│                                                │  (AI fallback) │  │
│                                                └───────┬────────┘  │
│                                                        │           │
│                                                ┌───────▼────────┐  │
│                                                │  Confidence    │  │
│                                                │  Gate          │  │
│                                                └───────┬────────┘  │
└────────────────────────────────────────────────────────┼────────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │ Mailbox Action  │
                                               │ (move + undo    │
                                               │  token)         │
                                               └─────────────────┘
```

### Classification Pipeline (7 Steps)

1. **Sender Learning** — Per-user sender domain overrides
2. **Sender Intelligence** — Built-in classification for known senders
3. **Rule Engine** — Priority-ordered: blacklist > whitelist > sender cache > domain > keyword
4. **Classification Memory** — Reuse exact matches from cache
5. **Cost Guard** — Enforce global + per-user AI call limits
6. **AI Classification** — OpenAI fallback with cost tracking
7. **Confidence Gate** — Threshold-based acceptance/rejection

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | Python 3.11, FastAPI, Uvicorn |
| **Database** | PostgreSQL 15, SQLAlchemy ORM |
| **AI** | OpenAI API (GPT-4o-mini) |
| **Email** | Microsoft Graph API, IMAP |
| **Billing** | Stripe (checkout, webhooks, portal) |
| **Frontend** | Next.js 14, React 18, TypeScript, Tailwind CSS |
| **Hosting** | Docker (backend), Cloudflare Pages (frontend) |
| **Monitoring** | Prometheus, Sentry, structured JSON logging |
| **Auth** | HttpOnly cookies + API keys (SHA256-hashed) |

---

## Features

### Email Classification
- Hybrid deterministic + AI pipeline
- Per-user custom rules (sender, keyword, domain)
- Classification memory cache
- AI confidence scoring with configurable threshold

### Cost Protection (Multi-Layer)
| Guard | Default |
|-------|---------|
| Global daily AI calls | 10,000 |
| Per-user daily AI calls | 500 |
| Daily dollar limit | $10.00 |
| Free tier daily quota | 10 |
| Abuse detection | Auto-block |

### Mailbox Automation
- Reversible email moves with undo tokens (24h expiry)
- Per-sync caps (max 50 moves, max 5 new folders)
- Abort on excessive failures (100+ threshold)
- Complete action logging with original folder tracking
- **No email deletion** — system invariant, never violated

### Inbox Intelligence (Premium)
- Email bundles — group by (category, sender domain)
- AI-generated daily digest (24h cache)
- Newsletter detection with unsubscribe metadata
- Importance scoring (top-5 ranking)

### Billing & Subscriptions
- Stripe checkout integration
- Subscription states: active, trialing, past_due, suspended, canceled
- Grace period before suspension (configurable)
- Founder and admin subscription sources
- Billing portal access

### Admin Controls
- Feature flags (20+ gates)
- Runtime settings with admin overrides
- User management and statistics
- Audit logging for all admin actions
- System health monitoring

---

## Security

### System Invariants (Non-Negotiable)
1. No email deletion — ever
2. All moves reversible via undo token
3. API keys never returned to frontend
4. Stripe test keys blocked in production
5. All user endpoints require authentication
6. Admin operations require explicit admin token
7. Rate limiting on auth endpoints
8. Multi-layer cost guards always enforced

### Authentication
- Cookie-based auth (HttpOnly, Secure, SameSite=Lax)
- API key authentication (SHA256-hashed storage)
- Magic link login (Resend or SMTP)
- Password login (12-char minimum, bcrypt)
- OAuth flows (Microsoft, Gmail, Yahoo)

---

## API Surface

| Category | Endpoints |
|----------|-----------|
| **Classification** | classify, confirm classification |
| **Mailbox Sync** | sync by provider, undo moves |
| **Rules** | CRUD for user-defined rules (50-rule cap) |
| **Account** | profile, settings, password |
| **Auth** | magic link, password login, OAuth (3 providers) |
| **Intelligence** | bundles, digest, newsletters, importance |
| **Billing** | checkout, verify, portal, webhooks |
| **Admin** | users, stats, feature flags, audit |
| **Health** | basic, database, deep diagnostics, readiness |

---

## Deployment

```
┌─────────────────────────────────────┐
│         Docker Compose              │
│                                     │
│  ┌──────────┐    ┌──────────────┐  │
│  │ postgres │◄───│  api-saas    │  │
│  │   :5432  │    │    :8001     │  │
│  └──────────┘    └──────┬───────┘  │
│                         │          │
│                    ┌────▼─────┐    │
│                    │  nginx   │    │
│                    │  (proxy) │    │
│                    └──────────┘    │
└─────────────────────────────────────┘
```

- Single Docker container (Python 3.11-slim)
- 150+ environment variables (documented in .env.example)
- Feature flags default to safe values (disabled)
- Environment validation enforced at startup

---

## Related Projects

- [Invoice Flow](https://github.com/aleavenger/invoice-flow-overview) — AI invoice automation SaaS
- [PaperclipAI](https://github.com/aleavenger/dev-company-overview) — AI orchestration platform managing development agents

---

## License

This is a proprietary project. Source code is not publicly available.

Architecture overview and technical details available on request.

---

**Built by [Alexandre Pereira dos Santos](https://github.com/aleavenger)**
