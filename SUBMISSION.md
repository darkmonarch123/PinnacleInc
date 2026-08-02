# Pinnacle — Submission Checklist

**Group number:** _fill in_
**Members:** _fill in_

---

## Rubric

> **Strict Architectural & Deployment Standards**
> Your project will be evaluated against industry-standard engineering practices. As you build, your group must adhere to the following baseline requirements:
>
> - **Clean Local Project Structure** — a standardized, clean directory layout. Keep production code, source assets, and test files separated into distinct, well-organized packages.
> - **Automated Testing Suite (JUnit 5)** — dedicated test classes utilizing the JUnit framework. The entire suite must execute seamlessly via `mvn test` from the command line. Any build that fails its automated test runner will be rejected.
> - **Version Control (GitHub)** — all source code continuously tracked, committed, and pushed to a shared group repository, with a clean commit history.
> - **Containerization (Docker)** — a functional Dockerfile, with a verifiable Docker image capable of running the system in any isolated environment.
> - **Cloud Deployment** — the live, working application fully deployed and accessible on Render or Vercel.

---

## Status

| Requirement | Status | Notes |
|---|---|---|
| Clean project structure | ✅ Done | See below |
| JUnit 5 test suite | ⚠️ Written, not executed here | See below |
| Docker | ✅ Done | `docker-compose.yml` + `backend/Dockerfile` + `frontend/Dockerfile` |
| GitHub | ❌ Not done | Requires your own repo + credentials |
| Cloud deployment | ❌ Not done | Requires your own Render/Vercel account |

### Clean project structure

The backend is organized by feature, not by layer — each business capability owns its own `dto` / `service` / `controller` subpackages, rather than one giant `controllers` folder for the whole app:

```
backend/src/main/java/com/pinnacle/
  entity/          — JPA entities + enums (shared across features)
  repository/       — Spring Data repositories (shared)
  security/         — JWT issuing/validation
  config/           — Security + WebSocket config
  marketdata/        — ingestion, candles, price cache
  oms/               — order placement, risk check, ledger
  position/          — netting, closing, SL/TP watcher
  reporting/         — trade history, stats, CSV export
  watchlist/         — watchlist CRUD
  alerts/            — price alerts, notifications
backend/src/test/java/com/pinnacle/
  (mirrors the same package structure under main)
```

The frontend similarly separates `pages/`, `components/`, `store/`, and `lib/` (API clients), rather than mixing concerns into one folder.

### JUnit 5 test suite — important caveat

Test classes exist for the highest-value logic to verify — the parts where a subtle bug would silently produce wrong numbers rather than an obvious crash:

- `RiskCheckServiceTest` — buying power, min/max order size, ticker tradability
- `LedgerServiceTest` — debit/credit correctness, that every posting persists both a ledger row and the updated account
- `PositionServiceTest` — the netting math: long/short P&L calculation, partial closes, and that a SELL actually nets against an existing long instead of opening a redundant new position
- `TradeHistoryServiceTest` — win rate, avg win/loss ratio, max drawdown, the zero-trades edge case, and equity curve accumulation (including that it sorts chronologically regardless of input order)
- `OrderServiceTest` — the full OMS state machine: market/limit fills, each rejection path (no price, failed risk check, missing limitPrice), non-marketable limit orders staying `ROUTED`, cancellation rules (only `ROUTED` is cancellable, ownership is enforced), and the background matcher's `fillPendingOrder` path
- `AccountServiceTest` — mark-to-market equity calculation, the no-live-tick-yet edge case, and demo-reset's delta math (including that it's a no-op when already at the starting balance)

**I have not been able to run `mvn test` myself.** This chat environment's sandbox doesn't have network access to Maven Central (only a handful of allowlisted domains — GitHub, npm, PyPI, crates.io — none of which serve Java dependencies), so I could write and review the tests carefully but couldn't compile or execute them here. Before you rely on these for the "must pass `mvn test`" requirement:

1. Run `mvn test` yourself in an environment with normal internet access.
2. Fix any compilation errors or assertion mismatches you find — the logic should be correct, but I'm flagging real uncertainty rather than claiming a false guarantee.
3. One known Mockito subtlety already fixed in `OrderServiceTest`: `Order` is a mutable entity, so an `ArgumentCaptor` across multiple `save()` calls on the *same* order instance would only ever see its final state, not a snapshot per call — that test asserts save *count* plus final state instead, rather than an intermediate-value check that would silently always pass or fail for the wrong reason.
4. Consider adding tests for market-data ingestion parsing (`TwelveDataClient`) if you want broader coverage — that wasn't covered in this pass, since it's more about JSON-shape parsing than business logic.

### Docker

`docker-compose.yml` at the repo root brings up Postgres/TimescaleDB, Redis, backend, and frontend together. `backend/Dockerfile` and `frontend/Dockerfile` each build their respective image standalone, so either can also be built and run independently:

```bash
cd backend && docker build -t pinnacle-backend .
cd frontend && docker build -t pinnacle-frontend .
```

Same caveat as the tests: I wrote these Dockerfiles but haven't been able to execute a build in this sandbox (no Docker daemon access, and network is restricted to a short allowlist). Build them yourself before submission to confirm they're clean.

### GitHub — you'll need to do this part

I can't push to a repository on your behalf — that needs your own GitHub account and the group's repo to exist first. Rough steps:

```bash
cd pinnacle
git init
git add .
git commit -m "Initial commit: Pinnacle trading platform scaffold"
git branch -M main
git remote add origin <your-group-repo-url>
git push -u origin main
```

From there, each group member should clone the repo and commit their own changes rather than working from a shared zip, so the commit history actually reflects who did what.

### Cloud deployment — you'll need to do this part too

Render and Vercel both need an account, and typically a connected GitHub repo, to deploy from. Rough shape of what each piece needs:

- **Frontend (Vercel)**: point Vercel at the `frontend/` directory of your repo; it should auto-detect the Vite build. Set `VITE_API_BASE_URL` and `VITE_WS_BASE_URL` as environment variables pointing at wherever the backend ends up.
- **Backend (Render)**: Render can build directly from `backend/Dockerfile`. You'll also need a managed Postgres instance (Render offers one, though you'd need to confirm it supports the TimescaleDB extension — if not, a separate TimescaleDB-compatible host, or Timescale's own cloud service, would be the fallback) and a Redis instance, wired in via the same environment variables `docker-compose.yml` uses (`SPRING_DATASOURCE_URL`, `SPRING_REDIS_HOST`, etc.).

I'd genuinely rather flag that I can't complete this part than claim it's handled — deployment credentials and live infrastructure aren't something I have access to from here.
