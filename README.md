# Thrive — Grow Your Wealth

Simulated stock trading platform. Paper trading only — no real money, no real broker integration, no financial licensing.

> **Rebrand + auth migration in progress.** This was originally "Pinnacle" (gold/navy trading-terminal aesthetic, custom JWT auth). It's being reworked toward a warmer, more organic direction referencing [Bamboo](https://investbamboo.com) (deep forest green, lime accent, editorial tone) and migrated from our own JWT auth to **Clerk**. See "Rebrand status" below for exactly what's done vs. still pending.

## Rebrand status

**Done:**
- New design tokens (Tailwind config + global CSS): forest green / lime / cream palette, Plus Jakarta Sans + Inter fonts, pill-shaped buttons, replacing the old gold/navy/serif system
- Every component bulk-migrated off the old color classes and hardcoded hex values
- New logo mark (sprout/leaves, replacing the gold mountain peak) and wordmark ("THRIVE")
- **Landing page fully rebuilt** — new hero layout, copy, and structure inspired by Bamboo's reference screenshots, not just recolored
- **Clerk fully wired on the frontend**: `ClerkProvider` in `main.tsx`, `Login`/`Signup` replaced with Clerk's own `<SignIn>`/`<SignUp>` components, a new `/kyc` page for the mock-KYC step that runs after Clerk sign-up, `ProtectedRoute` checks Clerk's session, and all API client files pull their auth header from Clerk's session token instead of a self-managed localStorage token
- **Clerk fully wired on the backend**: `ClerkTokenVerifier` (JWKS-based RS256 verification, no shared secret), `ClerkAuthFilter` replacing the old `JwtAuthFilter`, auto-provisioning a `User`+`Account` on first request from a new Clerk identity, a `clerk_user_id` column added to `users`, `password_hash` made nullable since Clerk owns credentials now
- Old JWT files deleted (`JwtService`, `JwtAuthFilter`, `AuthService`'s register/login, `RegisterRequest`/`LoginRequest`/`AuthResponse` DTOs) rather than left as dead code

**Not done yet:**
- **Only the Landing page got a real structural redesign.** Dashboard, Portfolio, Trade History, Watchlist, Alerts, Settings, and Orders only got the mechanical color-token swap so far — same layouts as before, just recolored.
- **Clerk JWT template**: whether a session token carries `email`/`name` claims depends on a JWT template configured in your Clerk dashboard, not on any code here. Without one, newly auto-provisioned users get a placeholder email (`{clerk_user_id}@clerk.placeholder`).
- **Never run.** None of the Clerk integration has been executed against a real Clerk app — no network access in this sandbox to verify JWKS fetching works end-to-end. Test the full sign-up → KYC → dashboard flow for real before trusting it.
- `CLERK_SECRET_KEY` is wired into env templates but nothing calls it yet — it's there for when you need Clerk's Backend API to fetch profile data not present in the session token.

## What was scaffolded before the rebrand


## What's scaffolded so far

**Infra**
- `docker-compose.yml` — Postgres (TimescaleDB image) + Redis + backend + frontend
- `db/init.sql` — full relational schema (users, accounts, tickers, orders, positions, trades, ledger_entries, price_alerts, watchlist_items) + TimescaleDB hypertables (`price_ticks` with 7-day retention, `price_ohlc` retained indefinitely) + seed tickers

**Backend** (Spring Boot 3 / Java 21)
- Entities for every table, matching schema exactly (`entity/`)
- Enums encoding the order and position state machines (`entity/enums/`)
- JPA repositories for all entities (`repository/`)
- JWT issuing/validation + stateless security filter chain (`security/`, `config/SecurityConfig.java`)
- Auth flow (historical — **superseded by Clerk, see "Rebrand status" above**): register → auto-provision demo account with $10,000 virtual balance → login → mock KYC endpoint. `AuthService`'s register/login and `JwtService`/`JwtAuthFilter` have been deleted; auth is now `ClerkTokenVerifier`/`ClerkAuthFilter` plus Clerk's own hosted components.
- `application.yml` wired for env-var overrides (DB, Redis, Clerk issuer/secret key, market data provider/key)

**Market data pipeline** (Spring Boot, `com.pinnacle.marketdata`)
- `TwelveDataClient` — polls Twelve Data's `/quote` endpoint (batched across symbols), normalized behind a `MarketDataProviderClient` interface so swapping providers means writing one new class
- `MarketDataIngestionService` — scheduled poll (`pinnacle.market-data.poll-interval-ms`) → caches latest price per symbol in Redis → persists every quote as a raw tick to `price_ticks` → broadcasts over STOMP to `/topic/prices/{symbol}`
- `CandleAggregationService` — scheduled rollup of raw ticks into `price_ohlc` candles for 1m/5m/1h/1D (1W deferred — needs a week-start convention decided first)
- `MarketDataController` — `GET /api/market-data/candles?symbol=&timeframe=&from=&to=` for historical candles
- `WebSocketConfig` — STOMP over SockJS at `/ws`, simple broker on `/topic`
- Note: `price_ticks`/`price_ohlc` are accessed via `JdbcTemplate`, not JPA entities — they're TimescaleDB hypertables with composite keys, which JPA doesn't model cleanly

**Frontend** (React + TypeScript + Vite + Tailwind)
- Brand tokens as Tailwind theme extensions — exact hex values from the brand sheet, plus the derived trading-UI extensions (`tailwind.config.js`)
- Fonts wired: Playfair Display (display), Inter (UI), IBM Plex Mono with tabular numerals (every price/balance/P&L figure) (`src/styles/index.css`)
- `Logo` component (gold peak + ascending bars + serif wordmark)
- `PnlFigure` component — color is never the only signal; every gain/loss pairs emerald/crimson with an icon and explicit +/- sign
- Terminal layout: `IconRail`, `AccountBar` (sticky Balance/Equity/Buying Power/Day P&L/Total P&L), `OrderTicket` (Buy/Sell toggle, order type, quantity stepper, SL/TP, live cost estimate)
- `Dashboard` page assembling the above, responsive: stacks to scrollable sections below `lg`, order ticket moves from a right rail to a bottom section on mobile
- Focus-visible rings and `prefers-reduced-motion` handling in the global stylesheet
- `lib/websocket.ts` — single shared STOMP/SockJS connection, multiplexes subscribers per symbol
- `lib/api.ts` — REST client for historical candles (reads a bearer token from `localStorage`, which nothing writes yet — see below)
- `store/useLivePrices.ts` — Zustand store + `useLivePrice(symbol)` hook, subscribes on mount / unsubscribes on unmount
- `PriceChart` — real Lightweight Charts candlestick series: loads history from `/api/market-data/candles`, nudges the current candle live as ticks arrive
- `WatchlistItem` — shows a "delayed" badge on static fallback data until its first live tick arrives, then switches to the live feed
- `store/useAuth.ts` — Zustand store wrapping register/login/KYC calls, persists token/userId/kycCompleted to `localStorage`
- `Login` and `Signup` pages — Signup is a 3-step wizard (Account → Identity → Confirm) that calls `register` then `submitKyc` against the real backend endpoints
- `ProtectedRoute` — redirects to `/login` if unauthenticated, to `/signup?step=identity` if authenticated but not yet KYC'd, wraps the Dashboard route
- Icon rail now has a working logout button
- `Landing` page — hero built around "Elevate Your Wealth," a quiet ascending-line accent behind the hero echoing the logo's bar-chart motif, feature grid, a genuinely sequential "how it works," and a disclaimer footer restating the simulated-platform constraint
- **Routing restructure**: `Landing` now owns `/`; the authenticated app moved to `/dashboard`. Login/Signup and the icon rail were updated to match.

**OMS** (`com.pinnacle.oms`)
- `OrderService.placeOrder` — the real state machine: `NEW → PENDING_RISK_CHECK → ROUTED → FILLED`, branching to `REJECTED` if the risk check fails or no price is available yet
- `RiskCheckService` — ticker tradability, min/max order size, sufficient buying power (checked for both BUY and SELL — see the scope note below)
- `LedgerService` — the only code path allowed to mutate `Account.balance`/`buyingPower`; always writes an immutable `ledger_entries` row first
- A FILLED order always spawns exactly one `Position` (OPEN) — see the scope note below on why this pass doesn't net a SELL against an existing long
- `OrderMatchingService` — scheduled sweep (5s) that fills pending `ROUTED` limit orders once price crosses the limit, and expires anything ROUTED for 24h+
- `OrderController` — `POST /api/orders` (place), `GET /api/orders` (list), `POST /api/orders/{id}/cancel`
- Frontend: `lib/ordersApi.ts` + `OrderTicket` now actually calls `placeOrder` and shows the fill/rejection result inline instead of just rendering a static form

**Position management** (`com.pinnacle.position`)
- `PositionService.processFill` — this is what OMS's fill hands off to. Nets FIFO against any existing opposite-side position first (closing it, writing a `Trade` row with realized P&L), then opens or weighted-average-adds to a same-side position with whatever's left over. **A SELL can now actually close a long** instead of always opening a new short.
- `closePosition` — the shared closing math (used by netting, manual close, and the SL/TP watcher alike): computes P&L, writes the `Trade`, releases reserved capital via the ledger, posts `REALIZED_PNL`, and moves the position through `PENDING_CLOSE → CLOSED`/`PARTIAL_CLOSE`
- `PositionWatcherService` — scheduled sweep (5s) comparing live price against every open position's stop-loss/take-profit, auto-closing when crossed
- `PositionController` — `GET /api/positions` (list, `openOnly` query param), `PATCH /api/positions/{id}` (update SL/TP, sets `MODIFIED`), `POST /api/positions/{id}/close` (manual full/partial close at current price)
- `OrderService.fill()` was refactored to delegate here instead of always blindly opening a new position — the OMS-pass simplification called out last time is now resolved

**Trade history & reporting** (`com.pinnacle.reporting`)
- `TradeHistoryService` — filterable trade list (symbol / date range / win-loss), stats (win rate, avg win/loss ratio, max drawdown, total realized P&L), an equity curve, and CSV export
- `TradeHistoryController` — `GET /api/trades` (filters via query params), `GET /api/trades/stats`, `GET /api/trades/equity-curve`, `GET /api/trades/export` (CSV download)
- `TradeHistory` page — real stat cards, symbol/win-loss filtering, and a CSV export button that fetches-and-blobs the file (a plain `<a href>` can't carry the auth header this endpoint needs)

**Watchlist & alerts** (`com.pinnacle.watchlist`, `com.pinnacle.alerts`)
- Added a `notifications` table — not in the original schema, needed to back "background job triggers in-app notification on match" for price alerts
- `WatchlistService`/`WatchlistController` — add/remove/list, each item annotated with its live cached price
- `PriceAlertService`/`PriceAlertController` — create/list/delete alerts (target price + above/below)
- `PriceAlertWatcherService` — 5s scheduled sweep matching active alerts against live prices; one-shot (deactivates on trigger, doesn't re-fire)
- `NotificationService` — persists notifications and pushes them live over STOMP to `/topic/notifications/{userId}`, mirroring the existing `/topic/prices/{symbol}` pattern
- Frontend: real `Watchlist` and `Alerts` pages, both wired to the endpoints above; Alerts also subscribes over WebSocket so a triggered alert shows up instantly instead of waiting for a refetch
- `lib/websocket.ts` was generalized from a prices-only wrapper into a generic pub/sub client (`subscribe(topic, listener)`), so notifications could reuse the same shared connection instead of a second one

**Account summary & Portfolio page** (`com.pinnacle.account`)
- Added `AccountService`/`AccountController` (`GET /api/account`) — this didn't exist before and the Portfolio page genuinely needed it: it marks every open position to market against `PriceCacheService`'s live prices and returns balance, buying power, unrealized P&L, and equity (balance + unrealized P&L). This is the first place in the codebase that actually marks positions to market — the reporting equity curve is realized-P&L-only by design (see its note above).
- `Portfolio` page — account summary cards, an allocation bar by symbol (by notional value at entry price), a live-updating open-positions table (mark price, unrealized P&L, inline SL/TP editing, close), and a recent-trades strip reusing `/api/trades`
- Added a `Portfolio` link to the icon rail (`PieChart` icon)

**Dashboard wired to real data**
- `AccountBar` now shows real `balance`/`equity`/`buyingPower` from `GET /api/account` instead of hardcoded numbers
- Watchlist strip now fetches the user's actual `/api/watchlist` instead of a hardcoded symbol list
- The "Open Positions" tab now shows real open positions with live mark price and unrealized P&L (same live-price pattern as the Portfolio page); "Pending Orders" and "Alerts" tabs are now at least switchable but remain placeholders — see below
- "Day P&L" mirrors live unrealized P&L rather than a true since-market-open figure, since there's no daily-snapshot mechanism to compute a real intraday number yet (noted inline in the component and below)

**Settings page** (`com.pinnacle.user`, plus additions to `com.pinnacle.account`)
- Added a `notifications_enabled` column to `users` (not in the original schema) and `UserService`/`UserController` (`GET`/`PATCH /api/users/me`) — profile, base currency, and the two toggles didn't have a backend home yet
- `AccountService.resetDemoBalance` + `POST /api/account/reset` — a **pragmatic** version of "reset demo balance," not the fuller "archive, don't delete" flow noted in the future-backlog section below: it administratively closes open positions and cancels pending orders (no `Trade` rows or extra ledger postings for those — they're being wiped, not traded out), then posts one `DEMO_RESET` ledger entry bringing balance/buying power back to exactly the starting balance. Closed trade history is untouched since nothing here deletes from `trades`.
- `Settings` page — profile form, base currency, notification/2FA toggles (optimistic with rollback on failure), and a confirmation-gated reset button (idle → confirming → resetting, not a single click) since resetting is destructive to open positions

**Orders page & equity curve chart** — the last two items from the master prompt's Build Order
- `Orders` page (`/orders`) — every order with status, filled quantity, limit price, and a cancel button for anything still `ROUTED`; the Dashboard's "Pending Orders" tab now links here instead of being a dead-end placeholder
- `EquityCurveChart` — a real Lightweight Charts line series on the Trade History page, wired to `GET /api/trades/equity-curve`; labeled inline as realized-P&L-only so it doesn't imply it's marking open positions to market (that's `AccountService`'s job, on the Portfolio/Dashboard side)
- Added `AccountServiceTest` (mark-to-market math, reset delta calculation, that reset closes positions/cancels orders without touching trade history) and equity-curve tests in `TradeHistoryServiceTest` (chronological accumulation regardless of input order, the no-trades edge case)

All 7 pages from the master prompt now exist, hit real endpoints, and are reachable from the icon rail — see "What's not built yet" below for what's genuinely still open.

## What's not built yet

Everything remaining is scope explicitly deferred rather than missed:

1. 1-week candle rollup (deferred pending a week-start convention decision)
2. Refresh-token flow — the frontend stores the refresh token but nothing uses it yet to silently renew an expired access token
3. Email verification — `AuthService.register` has a `// TODO` where the verification email would be sent; `email_verified` stays `false` indefinitely right now
4. A real "Day P&L" — would need some kind of daily equity snapshot (e.g. a scheduled job recording equity at a fixed time, or at least at first-login-of-the-day) to diff against; right now it's a stand-in using unrealized P&L
5. The fuller demo-reset flow from the future-backlog section (archival "past seasons" view, reversal ledger entries instead of a single reset entry) — what's built now is a simpler, one-shot version
6. Everything in the "Future backlog" section further down (idempotency keys, rate limiting, partial fills, leaderboard, account funding, symbol search, contract tests, load testing) — all explicitly scoped as beyond this build, not oversights

### Known rough edges in what's built
- `TwelveDataClient`'s response-shape parsing is written to spec but unverified against a live API key/response
- `CandleAggregationService` uses simple fixed-rate scheduling per timeframe rather than a TimescaleDB continuous aggregate
- No WebSocket-level auth yet — anyone who can reach `/ws` can subscribe to any symbol's price feed, and now also any user's notification topic if they guess/obtain the UUID (low risk since it's a random UUID, but worth locking down with per-user destinations before this is ever public-facing)
- No `/logout` backend call — the frontend logout just clears local storage; the JWT stays valid until it expires
- `buyingPower` moves in lockstep with `balance` — no separate margin multiplier modeled; opening any position (long or short) reserves cash the same way
- Only full fills are implemented; `PARTIALLY_FILLED` exists in the schema/enum but nothing produces it yet
- Netting is FIFO only — no LIFO or specific-lot-selection option, and no per-lot cost basis tracking beyond the single weighted-average entry price per position
- The reporting equity curve and max-drawdown calculation are still realized-P&L only, even though `AccountService` now does mark-to-market for the account summary and Dashboard — the two aren't unified, so "equity" means something subtly different depending on which endpoint you're looking at. Worth reconciling if this grows past a demo.
- `TradeHistoryService` filters trades in Java after fetching the full account history rather than pushing filters into SQL — fine at demo scale, would want proper query params if trade volume grows
- `AccountService.unrealizedPnlFor` fetches the `Ticker` by ID inside a loop (N+1-style), same pattern as a few other services in this codebase — fine at demo scale, worth batching if position count grows
- `STARTING_BALANCE` is hardcoded as a frontend constant (`Dashboard.tsx`) to compute total P&L, rather than coming from an endpoint — it happens to match the backend's `pinnacle.demo.starting-balance` default, but the two aren't actually linked, so changing one without the other would silently produce a wrong number
- Price alerts are one-shot: once triggered they deactivate rather than re-arming, and there's no "re-enable" endpoint yet — the user has to create a new alert

## Future backlog

Ideas for hardening and extending this beyond the master prompt's scope, roughly grouped and prioritized by what protects the trading engine's correctness first, then product polish, then ops:

**Correctness / hardening (do these before anything else touches OMS)**
- **Idempotency + duplicate-order protection** — nothing currently stops a double-click or a retried request from placing two orders. Add an `Idempotency-Key` header, a dedup check (small table or Redis) in `OrderService`, and tests: same key twice → same result, no second fill; different keys → separate orders.
- **Rate limiting on order placement** — nothing stops hammering `POST /api/orders`. A per-user token bucket (Bucket4j or Redis-backed) with a 429 response; tests for under-limit passes, over-limit rejects, resets after window.
- **Partial fills** — `PARTIALLY_FILLED` exists in the schema/enum but nothing produces it (flagged earlier in this README). Would mean teaching `OrderMatchingService` to fill against a simulated liquidity/volume cap per tick, and verifying `PositionService.processFill`'s weighted-average math holds up against a partial fill. Riskier than the others since it touches core matching logic already exercised elsewhere — worth its own dedicated pass rather than bundling with something else.

**Product features**
- **Portfolio detail page** — backend already has everything needed (positions, trades, ledger). Mostly a frontend job: per-position P&L breakdown, allocation by symbol, a filterable trade log off the existing `/api/trades` endpoint. Pairs naturally with wiring the Dashboard's static positions table to the real endpoint.
- **Fuller notifications** — the notifications table/service built for price alerts only covers alert triggers right now. Extending it to also fire on order fills and SL/TP-triggered closes (each event type producing exactly one notification row) would make the OMS and position watcher feel as alive as the alert watcher already does.
- **Account funding simulation** — a mock deposit/withdraw flow (capped total, cooldown between deposits) through `LedgerService`, so the $10k isn't fixed forever. `POST /api/account/deposit` / `/withdraw` + an `AccountBar` modal.
- **Multi-account / reset flow** — a "reset my portfolio" endpoint that zeroes positions and restores the $10k balance without deleting history — insert a reversal `ledger_entries` row rather than mutating past records, so old trades stay queryable.
- **Order types beyond market/limit** — genuine stop and stop-limit orders (trigger a market/limit order when price crosses a level, before any position exists) are a meaningfully different code path from `PositionWatcherService`'s SL/TP-on-an-existing-position logic.
- **Symbol search / instrument discovery** — the seed ticker list is static; `GET /api/tickers/search?q=` (backed by the `tickers` table, or Twelve Data's own symbol search if the key is live) plus a frontend autocomplete on the order ticket and watchlist-add flow.
- **Simple leaderboard** — `GET /api/leaderboard` ranking demo accounts by realized P&L or return on the $10k baseline. Cheap, and gives the paper-trading framing a bit of a "product" feel.
- **Structured audit log** — `GET /api/admin/audit` surfacing `ledger_entries` + order state transitions per user, admin-gated. Pure read/query work, no changes to the trading engine, but reinforces the "real platform" feel given KYC's already mocked in.

**Ops / quality**
- **API documentation + contract tests** — wire up springdoc-openapi for auto-generated docs, then contract-test the 4-5 most-used endpoints against what the frontend's TypeScript types expect (`lib/ordersApi.ts` etc.), to catch drift early.
- **Performance/load smoke test** — a JMeter/Gatling script simulating N concurrent users placing orders and subscribing to price feeds, to see whether the 5s scheduled sweeps (`OrderMatchingService`, `PositionWatcherService`) and STOMP broadcast hold up before it's a real concern.

## Running this locally

This repo needs a real dev environment to build and run — Docker, Maven dependency resolution, npm installs. From a machine or environment with those available:

```bash
# 1. Copy env template and add a market data API key if you have one
cp .env.example .env

# 2. Bring up the full stack
docker compose up --build

# Backend: http://localhost:8080
# Frontend: http://localhost:5173
```

**Recommended: continue this build in Claude Code.** It can run `docker compose up`, iterate on Maven/npm errors, and keep the dev servers live while making changes — this chat environment can write files but can't execute the stack end-to-end.
