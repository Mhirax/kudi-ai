# Kudi AI Bank

Microfinance banking platform — NestJS backend, React frontend, PostgreSQL.

> **Current stage:** 4 of ~11 planned modules built, and as of 2026-08-22
> **all four are MVP-complete** — Identity, Accounts, Compliance (KYC), and
> now Transfers, whose last two open items (no ledger, no executor tests)
> are both closed. Funding/Deposits is next.

**Roadmap source of truth:** this file + [`docs/module.md`](docs/module.md)
(full module-by-module reasoning).
**Audit baseline:** [`docs/AUDIT.md`](docs/AUDIT.md) (2026-08-15 — several
items since closed, see §4). **Contract map:** [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md)

---

## 1. Stack

| Layer | Technology |
|---|---|
| Backend | NestJS + TypeScript (Clean Architecture, DDD, CQRS) |
| Frontend | React + Vite + SCSS |
| Database | PostgreSQL + Prisma |
| Auth | JWT + refresh-token rotation, account lockout after 5 failed attempts |
| Payments | Flutterwave (sole provider) |
| Deployment | Docker + Kubernetes, GitHub Actions |
| Rust engines | scaffolding only — not in the request path |

Redis and RabbitMQ have infrastructure modules but **no business module uses
them yet**. Domain events run in-process via `@nestjs/cqrs`'s `EventBus`.
Don't add a broker until something actually needs one.

---

## 2. Where the project stands

**30 live endpoints across 4 built modules.**

| Module | Endpoints | MVP verdict |
|---|---|---|
| [**identity**](modules/identity/implementation.md) | 7 — register/login/refresh/logout/change-password/get self/by-id | ✅ Complete |
| [**accounts**](modules/accounts/implementation.md) | 8 — open/list/get/credit/debit/freeze/unfreeze/close | ✅ Complete |
| [**transfers**](modules/transfers/implementation.md) | 5 — internal (atomic)/external (Flutterwave saga)/list/get/webhook | ✅ Complete |
| [**compliance**](modules/compliance/implementation.md) | 10 — KYC verify/staff lookup/override/sanctions/staff freeze | ✅ Complete |

**Transfers closed out 2026-08-22.** The ownership-leak bug that originally
raised doubt about this module was fixed earlier, idempotency followed
(2026-08-20 — a retried request now replays the original result instead of
double-debiting someone, via `shared/idempotency/`, a reusable mechanism
Funding/Bills/Loans are expected to reuse rather than each rebuilding), and
now both remaining items are closed too: every balance mutation posts a real
double-entry journal (`shared/ledger/`, adapted to a pre-existing
`ledger_entries` table found live on the DB rather than replacing it), and
`PrismaInternalTransferExecutor` — the highest-risk file in the repo — has
dedicated integration coverage, which in the process caught and fixed a real
bug in failed-transfer persistence. `shared/ledger/` is an interim,
denormalized mechanism, not a substitute for `/rust/ledger-engine` actually
being built — see §4.

**Verified, not asserted:** `tsc --noEmit` clean · 56 unit tests / 11 suites ·
36 integration tests / 8 suites, run against the real Postgres DB in `.env`.

---

## 3. Backend health

**Solid across all four modules:** Clean Architecture/DDD applied for real,
not folder decoration · cross-module calls go through ports, with two
documented exceptions (direct `CommandBus` dispatch, chosen over an
event listener specifically to avoid `EventBus.publish()`'s fire-and-forget
silent-failure risk) · money is `BigInt` end to end, no floats · optimistic
concurrency via `version` columns · rate limiting + account lockout live ·
CORS fails closed by default · request-retry safety (`shared/idempotency/`)
proven correct against a real concurrent race, not just application logic —
the database's own unique constraint is what guarantees it, not a
check-then-act.

**Structural hole that remained, in Transfers, is now closed:** every
balance mutation posts a real double-entry journal (`shared/ledger/`) that
can be reconciled against Flutterwave. Still an interim, denormalized
mechanism — `Account.balance` stays the system of record until
`/rust/ledger-engine` is actually built out.

**Ordinary hardening still owed:** no controller/e2e tests anywhere, `tx: any`
in the transfer executor (deliberate, documented convention — not itself a
bug), frontend ESLint reportedly never runs (per the 2026-08-15 audit, not
re-verified since).

---

## 4. Known gaps

| # | Issue | Severity |
|---|---|---|
| 1 | No controller/e2e tests anywhere, only domain + some integration | Medium |
| 2 | Frontend ESLint never runs — no `frontend/tsconfig.json` | Medium |
| 3 | `client.js` silently drops `ValidationPipe`'s array-format errors — falls back to a generic status message instead of the real reason | Medium |
| 4 | No name enquiry before external transfers — a typo sends money to the wrong person | Low |
| 5 | Dead auth screens — `VerifyOtp.jsx`, `SetupPin.jsx`, no backend behind either | Low |

**Closed this session (2026-08-22):** Transfers' last two open items —
no ledger (`shared/ledger/`, double-entry journal for every balance
mutation) and no tests on `PrismaInternalTransferExecutor` (which caught a
real bug in failed-transfer persistence along the way). Full writeup in
`modules/transfers/implementation.md`.

**Closed 2026-08-20 or earlier:** rate limiting, `CORS_ORIGIN` defaulting to
`*`, client-side-only account provisioning, disabled account lockout, the
`AccountsController` freeze/unfreeze role asymmetry, and Transfers'
idempotency gap. Detail in each module's `implementation.md`.

---

## 5. Open decisions

D2–D6 were not touched this session. Recommendation given; the call is
the team's — full reasoning for each lives in git history/prior discussion.

~~D1 — Idempotency key ownership~~ — **resolved 2026-08-20**, see
`shared/idempotency/`.

| # | Decision | Recommended | Blocks |
|---|---|---|---|
| D2 | Money units in unbuilt modules | Decimal strings on the wire everywhere — `savings`/`cards`/`loans` clients currently use `*Kobo` ints, treat as a defect | Every future module |
| D3 | Hardcoded 20-bank list in `transfer.js` | Add `GET /transfers/banks` proxying Flutterwave, cached | Nothing urgent |
| D4 | No name enquiry before external transfers | Build with Funding — same Flutterwave client work | Nothing urgent |
| D5 | Internal transfers have no UI | Add `GET /users/lookup?phone=` — name + account ID only, nothing else | Nothing urgent |
| D6 | OTP/transaction PIN — screens exist, unrouted, no backend | Decide in-or-out now; a PIN changes the transfer contract | Nothing — D1 resolved without a PIN |

---

## 6. What's next

**Foundational hardening before Funding** (not a module, just prerequisites)
is now fully done: ~~idempotency~~ (✅ 2026-08-20) → ~~`LedgerEntry`
journal~~ (✅ 2026-08-22, `shared/ledger/`) → ~~tests on the transfer
executor~~ (✅ 2026-08-22). Account provisioning and throttler/CORS
hardening were already done too.

**Upcoming modules, in order** — each justified by what it needs to already
exist to mean anything:

| # | Module | Why here |
|---|---|---|
| 5 | **Funding/Deposits** | No way for real money to enter an account today except an admin manually crediting one. `CreditAccountHandler` already anticipates this integration. |
| 6 | **Beneficiaries** | Saved recipients for Transfer. No dependency on Funding — can slot in anytime once Transfers exists, which it already does. Low effort, real UX gap. |
| 7 | **Savings** | `AccountType.SAVINGS` already exists, unused. Mostly reuses Accounts/Transfers mechanics. Needs Funding — nothing to move into savings otherwise. |
| 8 | **Bills** | Same shape as Transfers' external payout (debit → third-party call → compensate on failure). `.env` already has a `BILLER_API_KEY` placeholder. |
| 9 | **Notifications** | Overdue, not just next — every module already emits events nothing consumes. |
| 10 | **Loans** | Deliberately not earlier despite being the flagship microfinance product — needs real repayment/transaction history to underwrite against, and carries the heaviest regulatory risk. |
| 11 | **Cards** | Heaviest external-dependency lift (processor integration, physical logistics). Least differentiated for an MVP. |
| 12 | **Rewards** | Pure value-add, no dependency on anything. Last on purpose. |

> **Note — the Funding screen has to be built from nothing (frontend).**
> Funding is the first module with no frontend to grow from: there is no
> `src/features/funding/` at all, where every other unbuilt module (bills,
> cards, loans, savings, rewards) at least has a stub screen already routed.
> Its API contract is pinned ahead of the backend in
> [`frontend/src/api/funding.js`](frontend/src/api/funding.js) as five
> `pendingEndpoint` stubs — create/fetch a virtual account, create a
> checkout, list deposits, look one up — so the screen's data needs are
> already known. Two surfaces to build: a **Fund Wallet** entry point, which
> `QuickActions` is currently missing (it offers only Send · Airtime · Bills
> · Savings), and a **deposit history** view. Keep the `pendingEndpoint`
> wrappers in place until the backend lands — per §7, no mock data in the
> frontend.

---

## 7. Next session — start here

1. Transfers has a clean verdict — all four built modules are MVP-complete.
   Start **Funding/Deposits** (§6): it can build on `shared/idempotency/`
   and `shared/ledger/` directly rather than re-solving retry-safety or
   double-entry bookkeeping from scratch.
2. Whether the still-missing `/rust/ledger-engine` (Transfers'
   `shared/ledger/` is a deliberate interim substitute, not that) needs to
   exist before a real production launch is a decision that hasn't been
   made — not assumed either way.

**Do not:** reintroduce mock data in the frontend · add a Rust engine to the
request path (still 50-line stubs) · wire Redis/RabbitMQ before a module
needs one.

---

## 8. Getting started

```sh
./scripts/setup.sh                              # install + generate Prisma client
docker compose up -d postgres redis rabbitmq    # local infrastructure
npm run prisma:migrate                          # migrate
npm run start:web-api                           # backend — port 3002, prefix /api/v1
cd frontend && npm run dev                      # frontend, separate terminal
```

**No mock mode.** The frontend talks to the real backend only. Features
whose backend module doesn't exist render an explicit "not available yet"
state — see [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md).

```sh
npm run typecheck            # currently clean
npm run test:unit            # 56 tests, 11 suites
npm run test:integration     # 36 tests, 8 suites, against the real DB
cd frontend && npm run build
```

---

## 9. Layout

```
apps/            Deployable NestJS apps (web / mobile / admin API)
gateway/         Cross-cutting middleware, guards, exception filters
modules/         DDD bounded contexts — identity, accounts, transfers, compliance
integrations/    Third-party adapters (Flutterwave)
shared/          Framework-agnostic kernel — value objects, enums, exceptions
infrastructure/  Prisma, Redis, RabbitMQ, config
frontend/        React + Vite client
rust/            Core banking engines — scaffolding, not in the request path
docs/            Audit, API contract, architecture, module roadmap, onboarding
tests/           unit (populated) · integration (populated) · e2e · performance (empty)
```

---

## 10. Conventions

- **Money:** `BigInt` minor units in the DB, decimal strings (`"1500.00"`) on
  the wire. Never floats, never kobo integers in JSON.
- **Lists:** `{ data: [], meta: { total, page, limit, totalPages } }`.
- **Errors:** throw a `DomainException` subclass; the global
  `HttpExceptionFilter` normalises the envelope.
- **Cross-module calls:** depend on the other module's port (interface + DI
  token), never its internals.
- **Cross-module reactions:** default to a domain event, handled in the
  consuming module. Exception: when the reaction is itself an invariant that
  must never silently fail, dispatch directly via the shared `CommandBus`
  instead, and document why at the call site (`RegisterUserHandler` →
  Accounts is the current example).
- **Concurrency:** every balance-bearing row carries a `version` column,
  written via conditional `updateMany`.
- **Frontend:** one API client per backend module. A method calls a real
  endpoint or throws `FeatureNotAvailableError` — never a mock, never a
  silent fallback.
