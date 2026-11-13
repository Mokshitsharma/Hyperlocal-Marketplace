<!-- readme-refresh 2026-11 -->
<div align="center">

# 🛍️ Hyperlocal Marketplace

**Order from local shops in Indore; riders deliver in about 30 minutes**

<img src="https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white" alt="TypeScript" /> <img src="https://img.shields.io/badge/Express-000000?style=for-the-badge&logo=express&logoColor=white" alt="Express" /> <img src="https://img.shields.io/badge/React-20232A?style=for-the-badge&logo=react&logoColor=61DAFB" alt="React" /> <img src="https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=nextdotjs&logoColor=white" alt="Next.js" /> <img src="https://img.shields.io/badge/Vite-646CFF?style=for-the-badge&logo=vite&logoColor=white" alt="Vite" /> <img src="https://img.shields.io/badge/Socket.io-010101?style=for-the-badge&logo=socketdotio&logoColor=white" alt="Socket.io" /> <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL" /> <img src="https://img.shields.io/badge/Drizzle-C5F74F?style=for-the-badge&logo=drizzle&logoColor=black" alt="Drizzle" />

</div>

## ✨ Highlights

- Customer app: browse shops, cart, cash on delivery, UPI or card, live delivery map
- Store dashboard: incoming orders with a 2-minute countdown and a chime
- Rider flow with live tracking
- MVP build: 52/52 backend tests pass

<!-- body -->
A hyperlocal B2C marketplace. Customers order items from Indore's local shops, and a partner delivery fleet delivers them in about 30 minutes. Built from `local-marketplace-plan.md`.

> **MVP / test build.** Three switches are on by default outside production. `AUTH_DEMO_MODE` makes every login use OTP **1234** with no SMS. `PAYMENTS_DEMO_MODE` means nothing is charged. `DEMO_MODE` uses simulated riders. The real integrations for all three are written and tested, but they're switched off. See [Integrations](#integrations).

```
/backend          Node + Express + Socket.io API, PostgreSQL via Drizzle ORM (TypeScript)
/customer-app     Mobile-first responsive web app, React + Vite (TypeScript)
/store-dashboard  Shop owner dashboard, Next.js (TypeScript)
```

## Run it

Requires Node 22+.

```bash
npm install
cp backend/.env.example backend/.env    # then pick a database (below)
npm run dev                              # API :4000, customer app :5180, dashboard :3000
```

**Database.** The backend always runs on PostgreSQL. Set `DATABASE_URL` in `backend/.env` to one of:

- **Real Postgres (recommended):** install it (`winget install -e --id PostgreSQL.PostgreSQL.17`), create the database (`createdb -U postgres local_marketplace`), then set `DATABASE_URL=postgres://postgres:<password>@localhost:5432/local_marketplace`.
- **No install:** `DATABASE_URL=pglite:./.pglite` runs embedded Postgres (PGlite) from a folder. It's for local testing only; the server refuses it in production.

In development, migrations run automatically at startup and the demo data is seeded into an empty database.

| App | URL | Demo login (AUTH_DEMO_MODE) |
|---|---|---|
| Customer app | http://localhost:5180 | `9876543210` + OTP `1234` (any valid number works) |
| Store dashboard | http://localhost:3000 | `9000000001`–`9000000005` + OTP `1234` |
| API health | http://localhost:4000/api/health | — |

| Command | What it does |
|---|---|
| `npm run dev` | All three apps with hot reload |
| `npm test` | 52 backend tests against in-memory Postgres (state machine, checkout/payments, dispatch retry/escalation, restart recovery, OTP, webhooks) |
| `npm run seed` | Wipe and reseed demo data (5 shops, 29 products, 5 riders, demo customer) |
| `npm run db:migrate -w backend` | Apply migrations (production deploy step) |
| `npm run db:generate -w backend` | Create a new migration after editing `backend/src/db/schema.ts` |

### Test the whole workflow

1. Open the customer app in a phone-sized window and the dashboard for **Sajawat Home Decor** (`9000000002`) side by side.
2. Add cushion covers and a diya set, then check out. **Cash on delivery** creates the order immediately. **UPI/Card** go through a simulated payment first.
3. On the dashboard, the order arrives with a 2-minute countdown and a chime. **Accept** it, then **Mark ready for pickup**.
4. A simulated rider accepts, rides to the shop, picks up and delivers, and the customer sees it live on the map. Dashboard earnings update after delivery.
5. Failure paths to try:
   - **Reject** an order: stock is restored and the payment refunded.
   - Ignore an order: it auto-cancels after 2 minutes.
   - Accept several orders at once: after 5 riders are busy, orders go "Still looking for a rider", and the customer can cancel for a refund.
   - Stop the backend mid-delivery (Ctrl+C) and start it again: the ride resumes from where it was.

## How it fits together

### Order state machine (`backend/src/domain/orderStateMachine.ts`)

```
placed ──store accepts──▶ store_confirmed ──rider picks up──▶ picked_up ──▶ on_the_way ──▶ delivered
  │                          │                                   │              │
  └─ rejected / timed out /  └─ store cancels, or customer       └──── ops only ─┘
     customer cancels           cancels once rider search                ▼
                                has escalated                        cancelled
```

- Store confirmation is a **hard gate**: no rider is requested until the shop accepts. Pickup needs a rider **and** the shop's "ready" mark.
- Every transition checks who is making it (customer / store / driver / system / ops) and is applied with a conditional `UPDATE … WHERE status = <expected>`. A webhook racing a store click can't double-apply.
- Every status change stamps `last_status_update_at`.

### Checkout and payment (`services/checkout.ts`)

A checkout reserves stock atomically (`stock_qty >= qty` in the `UPDATE`, so there's no overselling). COD turns into an order immediately. UPI/card turn into an order only after the payment is verified and captured, so stores never see unpaid orders. Unpaid checkouts release their stock when the customer closes the payment sheet or after 15 minutes. Store payouts count only delivered orders whose money was actually captured (`amount_captured`).

### Rider dispatch (`services/dispatch/`, `services/orders.ts`)

After the store accepts, a rider is requested with `order_type = marketplace`. If nobody accepts within `DISPATCH_WINDOW_SECONDS` (180) the request is retried. After `DISPATCH_MAX_ATTEMPTS` (3) the order is **escalated**: the customer is told and offered a cancel-and-refund, the store and ops are alerted, and retries continue. If a rider declines, the request is re-sent. If a rider goes offline **before** pickup, the order is re-dispatched. If they go offline **after** pickup, the order is flagged for ops, because the goods are with the rider.

### Restart recovery and stuck orders (`services/recovery.ts`)

At startup, every in-flight order (`store_confirmed`, `picked_up`, `on_the_way`) is reconciled using `last_status_update_at`:

- **Quiet longer than `STUCK_FACTOR` × expected delivery time** (default 2 × 30 min; scaled by `DEMO_SPEED` in demo mode): flagged `needs_review` with a reason, and ops is alerted. It is never silently resumed or left frozen.
- **No rider yet:** the rider request is re-sent.
- **Rider working:** re-attached. The simulator continues the ride from the rider's last saved position; the fleet keeps working the job itself.

A startup warning lists what was resumed, re-dispatched or flagged. A watchdog repeats the stuck check every minute while the server runs. `GET /api/health` returns 503 if the database is down, and reports integration modes, the count of orders needing review, and the startup recovery report.

### Ops API (until there's an admin panel)

Set `OPS_API_KEY` (and optionally `OPS_WEBHOOK_URL` for Slack/Discord alerts), then call with the header `x-ops-key`:

- `GET /api/ops/orders/review`: orders flagged `needs_review` or with rider search escalated
- `POST /api/ops/orders/:id/resolve` with `{ "action": "cancel" | "mark_delivered" | "redispatch" | "clear_flag", "note": "…" }`

## Integrations

| Switch (default outside prod) | When on | When off (real) | Status |
|---|---|---|---|
| `DEMO_MODE` | Simulated riders | `FleetDispatchService` → fleet dispatch HTTP API + signed webhooks at `/api/driver/webhook` | Interface and flow final; **HTTP details are assumptions**. See the open questions (Q1–Q8) at the top of `services/dispatch/fleet.ts` |
| `PAYMENTS_DEMO_MODE` | Payments succeed instantly, nothing charged | Razorpay Orders + Checkout; capture, refunds, webhooks at `/api/payments/webhook` | Implemented and tested against a fake Razorpay, **not yet run against a Razorpay test account**. Parked for the MVP |
| `AUTH_DEMO_MODE` | OTP is always `1234` | 6-digit OTP sent via MSG91, stored hashed, 5-minute expiry, 5 tries, 30 s resend cooldown, 5 per hour | Implemented and tested against a fake MSG91; needs a DLT-approved template |

With `NODE_ENV=production` all three switches default **off**. The server refuses to start in production if `AUTH_DEMO_MODE` is on, if `JWT_SECRET` is the default, or if any real integration is missing its keys.

### Still to do before a real pilot

- Answer the fleet API questions (Q1–Q8 in `services/dispatch/fleet.ts`) and adjust `FleetDispatchService` and the webhook schema in `routes/driver.ts`.
- When online payment is back in scope: run a Razorpay test-mode checkout end to end and register the webhook.
- The tracking map is a schematic, and addresses snap to a fixed list of Indore areas (`domain/catalog.ts`). Swap in Google Maps/Places.
- There's no admin panel yet, so shop onboarding means inserting into `stores` (or editing the seed). Use the ops API for flagged orders.
- Push notifications (FCM) for customers with the app closed.

---

<div align="center"><sub>Built by <a href="https://github.com/Mokshitsharma">Mokshit Sharma</a> · <a href="https://mokshitsharma27.vercel.app/">Portfolio</a> · <a href="https://www.linkedin.com/in/mokshit-sharma-75b5ab305">LinkedIn</a></sub></div>
