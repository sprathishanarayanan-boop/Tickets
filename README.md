# Stubly: Multi-Tenant Event & Ticket Reservation System

A full-stack web application where organizers host events and sell tiered tickets (VIP, General Admission...), and attendees reserve tickets with a 10-minute hold before paying. Concurrent bookings can never double-book a ticket.

- **Live app:** `<add your deployed frontend URL>`
- **Live API docs:** `<add your deployed API URL>/docs`
- **Repository:** https://github.com/sprathishanarayanan-boop/event-ticket-reservation-system

## Problem statement

Build a web application that allows event organizers to host events and sell tiered tickets (e.g., VIP, General Admission). The application must handle concurrent booking attempts without double-booking seats.

### Functional requirements

- **Role-Based Access Control (RBAC):** three roles: Admin, Organizer and Attendee.
- **Event management:** organizers can create events, specify ticket tiers, set quantities and set sales cut-off dates.
- **Reservation engine:** attendees select tickets and place them in a 10-minute temporary reservation hold before completing checkout.
- **Payment simulation:** a mock gateway endpoint approves or rejects payments asynchronously.
- **Analytics dashboard:** organizers see real-time revenue charts and ticket conversion rates.

### Technical requirements

- **Database transactions:** use transactions or atomic operations (PostgreSQL `SELECT FOR UPDATE` or Redis locks) to prevent race conditions during checkout.
- **Caching layer:** cache active event details in Redis to handle heavy read loads.
- **Background jobs:** a cron/queue worker releases expired ticket holds after 10 minutes.

### Deliverables

- ER diagram of the data model (Users, Events, Tickets, Orders, Transactions)
- Postman collection testing the API routes with edge cases (concurrent bookings, expired sessions)
- Deployed application URL
- Full-stack application on GitHub with this README.md

### Where each requirement is met

| Requirement | Implementation |
|---|---|
| RBAC | `backend/app/deps.py` (`require_roles`), applied on every route |
| Event management | `backend/app/routers/events.py`, form in `frontend/src/pages/Organizer.jsx` |
| Reservation engine | `backend/app/routers/reservations.py`, `backend/app/services/holds.py` |
| Payment simulation | `backend/app/services/payments.py`, `backend/app/routers/payments.py` |
| Analytics dashboard | `backend/app/routers/analytics.py`, charts in `frontend/src/pages/Organizer.jsx` |
| Transactions and locking | `SELECT FOR UPDATE` plus atomic conditional UPDATE in `services/holds.py` |
| Redis caching | `backend/app/cache.py`, used by `routers/events.py` |
| Background job | `release_expired_holds` in `services/holds.py`, scheduled in `backend/app/main.py` |
| ER diagram | `docs/ER_DIAGRAM.md` and `docs/er-diagram.png` |
| Postman collection | `postman/ticket-reservation.postman_collection.json` |
| Deployed URL | Live links at the top of this file |

Structure follows the layout of a React + FastAPI starter: an independent frontend, an independent backend, plus `scripts`, `docs` and `postman` folders.

## Features

| Area | What it does |
|---|---|
| Role-based access control | Three roles: **Admin**, **Organizer**, **Attendee**. Enforced on the server for every route; the role is read from the database so role changes apply immediately. |
| Event management | Organizers create events with several ticket tiers, a quantity and price per tier, and a sales cut-off date. They can stop sales at any time. |
| Reservation engine | Attendees place a 10-minute hold on tickets, then check out. Unpaid holds are released automatically. |
| Payment simulation | A mock gateway approves or declines payments asynchronously. Test cards: approve, decline, or random. |
| Analytics dashboard | Organizers see revenue by day, tickets sold, hold-to-purchase conversion rate and tier sell-through, refreshed every 5 seconds. |
| Admin tools | List users, change roles, trigger the cleanup job on demand. |

## Tech stack

- **Frontend:** React 18, Vite, React Router, Recharts
- **Backend:** FastAPI, SQLAlchemy 2, Pydantic 2, JWT (PyJWT), bcrypt
- **Database:** PostgreSQL (SQLite fallback for quick local runs)
- **Cache:** Redis (in-memory fallback for local runs)
- **Background jobs:** APScheduler worker started with the API
- **Deployment:** Render blueprint (`render.yaml`) with Docker support

## How double-booking is prevented

Each ticket tier row stores `quantity`, `sold` and `held`. Creating a hold works like this (`backend/app/routers/reservations.py`, `backend/app/services/holds.py`):

1. Start a database transaction and lock the tier row with `SELECT ... FOR UPDATE`. Other bookings of the same tier wait their turn.
2. Run one atomic statement: `UPDATE ticket_tiers SET held = held + :n WHERE id = :id AND quantity - sold - held >= :n`. If it changes zero rows, there are not enough tickets and the API answers `409`.
3. Insert the reservation and commit.

The same "claim with a conditional UPDATE" pattern guards every state change (hold -> processing, payment settled, hold expired), so each transition happens exactly once even if two requests or two workers collide. A `CHECK (sold + held <= quantity)` constraint is the last line of defence. Rows are always locked in the same order (reservation, then tier), which avoids deadlocks.

## Reservation and payment flow

```
POST /reservations        hold ACTIVE, expires in 10 min, tickets leave the shelf
POST /orders/checkout     hold PROCESSING, order PENDING, returns 202 immediately
   mock gateway answers   (background task, ~2 s)
      approved  -> order PAID, hold CONVERTED, tickets issued, sold += n
      declined  -> order FAILED, hold back to ACTIVE if time remains (retry allowed)
background worker (every 30 s)
   ACTIVE holds past expires_at -> EXPIRED, tickets returned
   payments stuck PENDING for 2 min -> rejected, tickets returned
```

Checking out an expired hold returns `410 Gone` and returns the tickets immediately, without waiting for the worker.

## Caching

`GET /api/events` and `GET /api/events/{id}` are cached in Redis for 30 seconds. Every change that affects availability (new hold, payment, cancellation, expiry, event edit) deletes the affected keys, so users do not see stale counts. If Redis is down the API keeps working and reads from the database.

## Project structure

```
backend/        FastAPI app (app/routers, app/services, models, security), tests, Dockerfile
frontend/       React + Vite single-page app
scripts/        seed_demo.py (demo data), concurrency_test.py (race test)
postman/        Postman collection and environment
docs/           ER diagram (PNG + Mermaid)
docker-compose.yml   local PostgreSQL and Redis
render.yaml     one-click deployment blueprint
```

## Run locally

Prerequisites: Node.js 18+, Python 3.10+. Docker is optional (for PostgreSQL and Redis).

**Option A: quickest (SQLite and in-memory cache, no Docker)**

```powershell
# backend (PowerShell)
cd backend
Copy-Item .env.example .env
# In .env: delete the DATABASE_URL and REDIS_URL lines, set JWT_SECRET to a long random value
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m uvicorn app.main:app --reload
```

```bash
# backend (macOS / Linux)
cd backend && cp .env.example .env     # same .env edits as above
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

**Option B: production-like (PostgreSQL and Redis)**

```bash
docker compose up -d        # from the repo root
# keep DATABASE_URL and REDIS_URL from .env.example, then start the backend as above
```

Open `http://127.0.0.1:8000/docs` for the interactive API docs, or `http://127.0.0.1:8000/health`.

**Frontend** (new terminal):

```bash
cd frontend
cp .env.example .env        # PowerShell: Copy-Item .env.example .env
npm install
npm run dev
```

Open `http://localhost:5173`.

**Demo data:** with the backend running, `python scripts/seed_demo.py` creates `organizer@demo.com` and `attendee@demo.com` (password `password123`) and two events. Set `ADMIN_EMAIL` and `ADMIN_PASSWORD` in `backend/.env` to get an admin account on startup.

## Configuration (backend/.env)

| Variable | Default | Meaning |
|---|---|---|
| `DATABASE_URL` | SQLite file | PostgreSQL URL for real use |
| `REDIS_URL` | empty | Empty means in-process cache (dev only) |
| `JWT_SECRET` | dev value | Set a long random string in production |
| `CORS_ORIGINS` | `http://localhost:5173` | Comma-separated frontend URLs |
| `HOLD_MINUTES` | `10` | Length of a reservation hold |
| `SWEEP_INTERVAL_SECONDS` | `30` | How often the cleanup worker runs |
| `PAYMENT_DELAY_SECONDS` | `2` | Simulated gateway latency |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | unset | Creates an admin on startup when both are set |

Frontend (`frontend/.env`): `VITE_API_URL` (backend URL) and `VITE_CURRENCY` (default `USD`).

## API overview

All routes are under `/api` and take `Authorization: Bearer <token>` unless marked public.

| Method and path | Role | Purpose |
|---|---|---|
| `POST /auth/register`, `POST /auth/login`, `GET /auth/me` | public / any | Accounts (self-registration as attendee or organizer) |
| `GET /events`, `GET /events/{id}` | public | Cached event list and details with live availability |
| `POST /events`, `PATCH /events/{id}`, `DELETE /events/{id}` | organizer (own), admin | Create, edit, stop sales |
| `GET /events/mine` | organizer, admin | Your events |
| `POST /reservations` | attendee | Hold tickets for 10 minutes |
| `GET /reservations/mine`, `GET /reservations/{id}`, `POST /reservations/{id}/cancel` | attendee | Manage holds |
| `POST /orders/checkout` | attendee | Pay for a hold (returns 202, payment is asynchronous) |
| `GET /orders/mine`, `GET /orders/{id}` | attendee, event organizer, admin | Order status, tickets, transactions |
| `POST /payments/mock-gateway/charge` | any user | Stand-alone mock gateway endpoint |
| `GET /analytics/overview` | organizer, admin | Revenue, tickets, conversion, sell-through |
| `GET /admin/users`, `PATCH /admin/users/{id}/role`, `POST /admin/sweep` | admin | User management and manual cleanup |

Test payment tokens: `tok_success` (approved), `tok_fail` (declined), anything else (85% approved).

## Testing

```bash
cd backend && pytest                                  # RBAC, no oversell under threads, checkout, expiry
python scripts/concurrency_test.py --stock 5 --buyers 25   # races 25 clients for 5 tickets against a running API
```

**Postman:** import `postman/ticket-reservation.postman_collection.json` and `postman/local.postman_environment.json`, set `baseUrl`, `adminEmail`, `adminPassword`, then run the collection with the Collection Runner. It covers:

- duplicate and weak registrations, wrong password, missing and invalid or expired tokens, role violations (403)
- invalid event dates and tiers (422)
- booking more than remains (409), booking the last ticket, double holds in one tier, cancelling twice
- **10 simultaneous bookings for 3 tickets: exactly 3 succeed**
- declined card then retry, double checkout (409), reading someone else's order (404)
- **expired holds** (folder 6): start the API with `HOLD_MINUTES=0.05`; checkout returns 410 and tickets return to stock after the cleanup job. With the default 10 minutes these checks are skipped.

## Deployment (Render)

1. Push this repo to GitHub.
2. In Render choose **New > Blueprint** and select the repo. `render.yaml` creates PostgreSQL, Redis (Key Value), the API and the static frontend.
3. When prompted, set `ADMIN_EMAIL` and `ADMIN_PASSWORD`. Leave `CORS_ORIGINS` and `VITE_API_URL` blank for the first deploy.
4. Copy the two generated URLs: set `CORS_ORIGINS` on the API to the frontend URL, and `VITE_API_URL` on the frontend to the API URL, then redeploy both.
5. Run `python scripts/seed_demo.py --api <API URL>` for demo data and paste the live links at the top of this README.

Free-tier services sleep when idle, so the first request after a pause can be slow. Check Render's current free-plan limits before relying on the database long term.

## Design decisions and limits

- Tickets are quantity-based per tier, not numbered seats. A seat map would add a `seats` table with the same locking approach.
- Only attendee accounts can book; organizers and admins manage events.
- Tables are created on startup. For a long-lived production system, add Alembic migrations.
- With several API workers, each runs the cleanup job. That is safe because every release is a conditional update that only one worker can win.
- Payments are simulated; no card data is collected or stored.
