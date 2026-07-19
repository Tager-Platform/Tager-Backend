<div align="center">

# 🛍️ TagerAI

### AI-Native Social Commerce OS for Egyptian Instagram Merchants

**Turning Instagram DMs into a real order pipeline — in Arabic, Franco-Arabic, and pure Egyptian dialect.**

[![.NET](https://img.shields.io/badge/.NET-10-512BD4?style=flat-square&logo=dotnet)](https://dotnet.microsoft.com/)
[![Angular](https://img.shields.io/badge/Angular-21.1.0-DD0031?style=flat-square&logo=angular)](https://angular.io/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Multi--Tenant-336791?style=flat-square&logo=postgresql)](https://www.postgresql.org/)
[![OpenAI](https://img.shields.io/badge/AI-GPT--4o--mini-412991?style=flat-square&logo=openai)](https://openai.com/)
[![SignalR](https://img.shields.io/badge/Realtime-SignalR-00A4EF?style=flat-square)](https://dotnet.microsoft.com/apps/aspnet/signalr)
[![License](https://img.shields.io/badge/License-Academic%2FPortfolio-lightgrey?style=flat-square)]()

</div>

---

## 💡 The Problem

In Egypt, thousands of fashion merchants run their entire business **inside Instagram DMs**. There's no cart, no checkout, no CRM — just a merchant scrolling through hundreds of conversations, manually copying order details into a notebook or an Excel sheet, replying to "بكام القميص ده؟" fifty times a day, and losing customers because a message sat unanswered for six hours.

This isn't a niche behavior — it's the dominant e-commerce model for an entire market that mainstream SaaS (Shopify, WooCommerce) was never designed for, because the *storefront* isn't a website. **It's a chat.**

**TagerAI turns that chat into a structured, AI-assisted business operating system** — without asking the merchant to change how they sell.

---

## 🎯 What TagerAI Does

| Capability | Description |
|---|---|
| 📥 **Unified Inbox** | Every Instagram conversation in one real-time, multi-agent inbox |
| 🧠 **Egyptian-Native AI** | Understands Arabic, Franco-Arabic ("`3ayez el size M`"), and mixed-script messages — not just Modern Standard Arabic |
| 🛒 **Order Extraction** | AI reads a scattered DM conversation and drafts a structured order (product, size, phone, address) with per-field confidence scores |
| 💬 **Reply Suggestions** | Context-aware, tone-matched reply drafts the merchant can send in one click |
| 👥 **CRM & Customer Intelligence** | Auto-tagging, purchase-pattern detection, and customer profiling — built from real conversation history |
| 📦 **Order & Fulfillment Pipeline** | Kanban-style order management from `received` → `shipped`, full audit trail, CSV export for shipping partners |
| 🤖 **Supervised Autonomy** | AI can act on its own *only* above a merchant-configured confidence threshold — with a hard emergency-stop switch, always |
| 🏢 **True Multi-Tenancy** | One platform, many merchant stores, zero cross-tenant data leakage — enforced at the query layer, not just the UI |

> **Design principle:** AI never sends anything without a human in the loop until confidence is measured, tested, and the merchant explicitly opts in. Trust is earned in phases, not assumed on day one.

---

## 🧩 Why This Isn't "Just Another CRUD App"

This project was engineered to production standards deliberately, as a demonstration of real system design — not just feature delivery:

- **Multi-tenant from the schema up** — row-level isolation via `tenant_id` + EF Core global query filters on every soft-deletable entity, so a bug can't leak Merchant A's customer data into Merchant B's dashboard.
- **Layered, boring, correct architecture** — `DAL → BLL → API → Common`, with a strict **Repository + Unit of Work + Orchestrator** discipline: repositories never commit, orchestrators own the single transaction per business operation. This was enforced through actual code review, not just written as a rule (see [Architecture Decisions](#-architecture--engineering-decisions)).
- **A three-phase AI trust model** — Manual → AI-Assisted → Supervised Autonomous — so every AI capability ships behind a feature flag and a confidence gate, never as an all-or-nothing switch.
- **Real-time by design** — SignalR pushes new messages and Kanban updates to connected clients within 2 seconds, scoped to tenant-only groups.
- **A schema that anticipates the roadmap** — e.g. a `stock_quantity` column was added to `product_variants` in Phase 1 (nullable, unused) specifically so Phase 3 stock management doesn't require a migration.

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                         Instagram Graph API                      │
│                    (DMs, OAuth, Webhooks, HMAC)                  │
└───────────────────────────────┬──────────────────────────────────┘
                                 │  Webhook Event
                                 ▼
                 ┌───────────────────────────────┐
                 │   IChannelProvider (Strategy) │  ← extensible: WhatsApp,
                 │   ChannelProviderFactory      │     Facebook, TikTok next
                 └───────────────┬───────────────┘
                                 ▼
                 ┌──────────────────────────────────┐
                 │      WebhookOrchestrator         │
                 │   (Unit of Work · single commit) │
                 └───────┬───────────────┬──────────┘
                         ▼               ▼
              ┌──────────────────┐  ┌────────────────────────┐
              │  PostgreSQL      │  │  SignalR Hub           │
              │  (tenant-scoped) │  │  /hubs/inbox           │
              └──────────────────┘  │  (tenant-grouped push) │
                                    └────────────────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────────┐
                 │        AI Engine (Phase 2+)       │
                 │  Arabic/Franco NLP → Intent       │
                 │  → Order Extraction (confidence)  │
                 │  → Reply Suggestions              │
                 │  → Safety Guardrails              │
                 │  → Supervised Auto-send (Phase 3) │
                 └───────────────────────────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │   Angular 21 (Signals) SPA    │
                 │   Standalone components       │
                 │   Tailwind v4 · Lazy-loaded   │
                 └───────────────────────────────┘
```

### Layered Backend Solution

```
TagerAI.API/            → Controllers, middleware, auth, SignalR hubs
TagerAI.BLL/             → Orchestrators, business rules, validation, AI integration
TagerAI.DAL/             → Repositories, EF Core, migrations, query filters
TagerAI.Common/          → Shared DTOs, error codes, cross-cutting utilities
```

**Non-negotiable rule enforced throughout the codebase:** repositories persist data, they never call `SaveChangesAsync()`. Every business operation flows through a domain-scoped orchestrator that owns exactly one commit — making transactional integrity provable, not just assumed.

---

## 🧠 The AI Engine: Three Phases of Trust

TagerAI's core thesis is that **AI-run commerce fails when merchants can't trust it** — so autonomy is earned incrementally, gated by measured accuracy, not vibes.

| Phase | Mode | What Happens |
|---|---|---|
| **Phase 1 — Manual Core** | 🟢 No AI | Full working platform: inbox, orders, CRM, catalog, CSV export. The foundation has to work perfectly before AI touches it. |
| **Phase 2 — AI Assistant** | 🟡 Co-pilot | AI extracts orders, suggests replies, tags customers, flags follow-ups — **every action requires merchant approval**. Confidence scores are shown per field, color-coded (🟢 ≥0.80 / 🟡 0.60–0.79 / 🔴 <0.60). |
| **Phase 3 — Supervised Autonomy** | 🔴 Opt-in autopilot | Above a merchant-configured confidence threshold (default ≥0.90), AI can auto-send order confirmations. **Opt-in only, default OFF**, full audit log, 5-minute undo window, and an emergency-stop kill switch at all times. |

Every AI output — extraction, reply, tag — passes through a dedicated **Safety Guardrails** layer *after* generation and *before* the merchant ever sees it: confidence-gated blocking, blacklisted-pattern filtering (hallucinated prices, false delivery promises), and full logging for prompt quality review.

---

## 🗄️ Data Model Highlights

A 22-table PostgreSQL schema, designed for correctness under real-world messiness:

- **Snapshot pattern on `order_items`** — product name, variant, and price are frozen at order time. A product renamed or repriced six months later can never corrupt a historical order.
- **Two-tier audit standard** — every table gets `created_at` / `updated_at`; tables where *who* changed something matters (`orders`, `products`, `channels`, tags) get full `created_by` / `updated_by` actor tracking.
- **Append-only `order_status_history`** — the full lifecycle of every order is reconstructable, including whether a transition was human- or AI-triggered (`changed_by_ai`).
- **Denormalized inbox performance columns** — `conversations.customer_name`, `channel_type`, and `last_message_preview` intentionally duplicate data to eliminate three joins from the single hottest query in the system (the inbox list).
- **AES-256 encrypted channel tokens** — Instagram access/refresh tokens never touch the database in plaintext.
- **`source` + `ai_confidence` on every tag relation** — so an AI-applied tag is always visually and structurally distinguishable from a merchant-applied one, and merchants can override AI at any time without data loss.

---

## 🛠️ Tech Stack

<table>
<tr>
<td valign="top" width="50%">

**Backend**
- .NET 8 · ASP.NET Core Web API
- EF Core (code-first migrations)
- PostgreSQL (row-level multi-tenancy)
- ASP.NET Identity + JWT (15-min access / 7-day refresh)
- SignalR (real-time inbox + Kanban)
- FluentValidation
- Hangfire (background AI/reporting jobs)
- Docker + GitHub Actions CI/CD

</td>
<td valign="top" width="50%">

**Frontend**
- Angular 21 (standalone components, no NgModules)
- Angular Signals for state management
- Tailwind CSS v4 (HSL design tokens)
- Lazy-loaded feature modules

**AI / Integrations**
- OpenAI GPT-4o-mini (prompt-engineered, versioned prompts)
- Instagram Graph API + OAuth 2.0
- Extensibility hooks for WhatsApp, Facebook, TikTok, Shopify, WooCommerce

</td>
</tr>
</table>

---

## 🗺️ Product Roadmap

```
PHASE 1 · Manual Core Platform         →  Merchant registers, connects Instagram,
(Sprints 0–3)                             manages orders & catalog manually, exports CSV

PHASE 2 · AI Assistant                 →  AI extracts orders & suggests replies —
(Sprints 4–7)                             merchant approves every action (co-pilot mode)

PHASE 3 · Supervised Autonomy          →  Opt-in auto-confirmation above confidence
(Sprints 8–10)                            threshold, proactive follow-ups, stock AI,
                                           billing & subscription management
```

Each phase gate has hard, measurable exit criteria before the next begins — for example, Phase 2 → Phase 3 requires **≥85% F1 score** on a 50-order human-labeled test set and **zero** safety-guardrail failures in the test suite. AI isn't declared "ready" — it has to prove it.

---

## 📐 Engineering Principles

- **Architectural consistency over novelty** — CQRS/MediatR was explicitly evaluated and ruled out mid-project to preserve a coherent, explainable architecture rather than mixing patterns for their own sake.
- **Planning before code** — every architectural ambiguity (entity hierarchy, cascade behavior, transaction boundaries) is resolved and documented *before* implementation prompts are written.
- **>80% test coverage** as Definition of Done on all backend work.
- **One Meta App, many merchants** — a single Instagram platform integration serves every tenant via per-merchant OAuth tokens, keeping infrastructure cost and complexity flat as the merchant base scales.

---

## 📌 Project Status

This is an actively developed portfolio/capstone project. Multi-tenant auth, the Instagram real-time messaging pipeline, and the full product catalog system are implemented and tested. AI order extraction and reply suggestions are the current focus of development.

---

<div align="center">

**Built as a demonstration of production-grade SaaS architecture — multi-tenancy, real-time systems, and applied AI, engineered end-to-end.**

</div>
