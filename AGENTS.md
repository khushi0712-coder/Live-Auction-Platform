# AGENTS.md — Live Auction Platform

Instructions for AI coding assistants and new developers working in this repository. Read this file before making any change.

---

## 1. Project Context

**Project:** Live Auction Platform  
**Type:** Real-time online auction web application

Sellers create and manage auctions. Bidders take part in live auctions and place bids in real time. The system must safely handle concurrent bids, auction timing, anti-sniping, proxy bidding, deposits, winner selection and payments.

### 1.1 User Roles

| Role | Capabilities |
|---|---|
| Public Visitor | Browse upcoming and active auctions; view auction and item details; register and log in |
| Bidder | Participate in auctions; place live bids; use proxy bidding; pay refundable deposit; manage watchlist; receive outbid and auction notifications; view bid history; complete winner checkout |
| Seller | Complete seller verification; create auctions; add item information; set starting price, reserve price, auction duration and rules; monitor auction activity; view winning bidder information |
| Admin | Manage users, sellers and verification; manage auctions; review suspicious bidding activity; manage reported issues; monitor platform activity |

Authenticated roles in the system are **Bidder**, **Seller** and **Admin**. Public Visitors are unauthenticated.

---

## 2. Project Rules (Always Apply)

- Do not claim a feature is implemented unless it exists in the repository.
- Do not claim tests passed unless they were actually executed.
- Do not invent APIs, database tables, packages, URLs, metrics or deployment results.
- If an architectural or technology decision is not defined, **ask the Team Leader** instead of assuming.
- Keep changes small and reviewable.
- Follow the existing project structure before creating new files.
- Never expose secrets, API keys, passwords or payment credentials.
- Never commit `.env` files.
- Server-side authorization is mandatory; frontend route guards are not security.

---

## 3. Technology Stack

Status labels: **Implemented** = exists in the repository and works; **Planned** = chosen but not yet built. Everything below is **Planned** until the team updates this table after verifying it in the repository.

| Area | Technology | Status |
|---|---|---|
| Frontend | React, TypeScript, Vite, Tailwind CSS, React Router, TanStack Query | Planned |
| Backend | Node.js, Express.js, TypeScript | Planned |
| Database | PostgreSQL, Prisma ORM | Planned |
| Real-time | Socket.IO / WebSocket | Planned |
| Caching and jobs | Redis, BullMQ / background jobs | Planned |
| Authentication and security | JWT, bcrypt, Zod, Helmet, CORS, rate limiting | Planned |
| Payments | Razorpay Test Mode | Planned |
| Testing | Vitest/Jest, Supertest, Playwright, k6 | Planned |
| DevOps | Docker, GitHub, GitHub Actions | Planned |
| Hosting | Vercel (frontend), Render/Railway (backend), managed PostgreSQL and Redis | Planned |

Update the status column only when a technology is actually set up and working.

---

## 4. Architecture

**Architecture: Modular Monolith.** One backend application, split into modules with clear boundaries.

Suggested backend modules: Auth, Users, Sellers, Auctions, Items, Bidding, Payments, Deposits, Notifications, Watchlist, Suspicious Activity, Admin.

**Request flow**

```text
Route → Controller → Service → Data Access
```

**Real-time bid flow**

```text
Client → Socket.IO → Authentication → Bid Validation → Transaction → PostgreSQL → Broadcast Updated Bid
```

- Routes and controllers handle HTTP only; business rules live in services; database access lives in the data access layer.
- Modules communicate through service functions, not by reaching into another module's data access code.
- PostgreSQL is the source of truth. Redis is supporting infrastructure (locks, caching, rate limiting, queues) and must never be the only copy of important data.
- The final folder structure must be **confirmed against the actual repository** before creating new modules or folders.

---

## 5. Critical Auction Rules

These are core business logic. Do not weaken them.

### 5.1 Live Bidding

- Only authenticated bidders can place bids.
- The auction must be `ACTIVE`.
- A bid must satisfy the minimum required increment.
- A bid cannot be placed after auction closure.
- Every accepted bid is stored permanently.

### 5.2 Concurrent Bids

- Multiple users may bid at the same time.
- Bid processing must be atomic.
- Database transactions and/or locking must prevent race conditions.
- The system must never accept two conflicting winning bids for the same auction state.

### 5.3 Anti-Sniping

- A valid bid in the final 2 minutes extends the auction by 2 minutes.
- The extension happens atomically with the bid.
- All connected bidders receive the updated end time.

### 5.4 Proxy Bidding

- A bidder can set a maximum bid.
- The system raises the bidder's active bid only when necessary.
- Proxy bids respect minimum increments and the bidder's maximum.

### 5.5 Deposit

- A bidder may need to pay a refundable deposit before bidding.
- Deposit status is verified server-side.
- Payment confirmation never relies only on frontend information.

### 5.6 Reserve Price

- If the highest bid does not reach the reserve price, the auction ends without a winner.
- The system clearly indicates that the reserve price was not met.

### 5.7 Winner

- The highest valid bid at close wins when reserve requirements are satisfied.
- Winner checkout must be completed within the defined time limit.
- If the winner does not pay, the defined second-highest bidder flow may be triggered.

---

## 6. Backend Rules

- Validate request body, query and path parameters with **Zod**.
- Use parameterized database operations (Prisma or parameterized queries only).
- Never trust client-provided auction status, bid amount, winner status or payment status.
- Validate all auction and bid rules on the server.
- Use database transactions for critical bidding operations.
- Use UTC timestamps everywhere.
- Store money safely and consistently (integer minor units or `DECIMAL`); never use floating-point arithmetic where precision matters.
- Hash passwords with bcrypt.
- Never log passwords, JWTs, payment secrets or sensitive user information.
- Centralize error handling.
- Use consistent API response formats.
- Apply rate limiting to authentication and bidding endpoints.
- Validate WebSocket connections and user permissions.
- Sanitize user-generated content.

---

## 7. API Conventions

All endpoints live under `/api/...`.

**Success**

```json
{
  "success": true,
  "data": {},
  "meta": {}
}
```

**Error**

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": []
  }
}
```

| Status | Meaning |
|---|---|
| 400 | Validation error |
| 401 | Unauthenticated |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Conflict |
| 422 | Business rule violation |
| 429 | Rate limited |
| 500 | Internal server error |

Do not add or rename endpoints without checking the project's API documentation and asking the Team Leader.

---

## 8. Frontend Rules

- Use a dedicated API/service layer.
- Never hard-code backend URLs; use environment variables.
- Do not expose secrets in frontend environment variables.
- Route guards are for user experience only; backend authorization is mandatory.
- Every page has loading, empty and error states.
- Show clear bid validation errors.
- Do not trust auction status received from the UI.
- Real-time auction data updates without manual refresh.
- Handle WebSocket disconnect and reconnect gracefully (rejoin the room and refetch current state).
- Keep reusable components separate from page-specific components.

---

## 9. Database Rules

- Use primary keys and foreign keys correctly; maintain referential integrity.
- Add unique constraints where required.
- Add indexes for frequently queried fields, including auction status, auction end time, seller ID and bidder ID.
- Use transactions for critical operations.
- Store timestamps consistently (UTC).
- Do not delete financial or bidding records in a way that destroys audit history.
- Preserve bid history for auditability.
- All schema changes go through Prisma migrations; never edit a deployed database by hand.

---

## 10. Security Rules

| Area | Rule |
|---|---|
| Authentication | JWT authentication; bcrypt password hashing |
| Authorization | Role-based access control enforced on the server for every protected route and socket event |
| Input | Zod validation on all inputs |
| SQL injection | Parameterized queries only |
| XSS | Escape output; sanitize user-generated content |
| CSRF | Not applicable to Bearer-token APIs; add protection if cookies are ever used |
| CORS | Allow only configured frontend origins |
| Headers | Helmet security headers |
| Rate limiting | Login, register and bidding endpoints |
| WebSocket | Authenticate the connection and authorize each event |
| Payments | Verify Razorpay signatures on the server |
| Secrets | Environment variables only; never in code, logs or Git |
| Logging | No sensitive data in logs |
| Auditing | Audit log for important admin actions |

---

## 11. Real-Time Rules

| Event | Purpose |
|---|---|
| `auction:join` | Client joins an auction room |
| `auction:leave` | Client leaves an auction room |
| `bid:place` | Client submits a bid |
| `bid:updated` | Server broadcasts the new highest bid |
| `auction:extended` | Server broadcasts a new end time (anti-sniping) |
| `auction:ended` | Server announces the auction result |
| `outbid` | Server notifies a user who lost the lead |
| `notification:new` | Server delivers a new notification |

**Clients must never directly modify authoritative auction state.** Clients send requests; the server validates, commits to PostgreSQL, and then broadcasts.

---

## 12. Git Workflow

- Never commit directly to `main`.
- Work on feature branches and open Pull Requests into `main`.
- At least one teammate reviews each PR.
- The Team Leader merges approved PRs.
- Use Conventional Commits.

```text
feat(bidding): add live bid validation
feat(auction): implement anti-sniping extension
fix(payment): verify razorpay signature
test(bidding): add concurrent bid tests
docs(srs): update auction workflow
```

Before starting work:

```bash
git fetch origin
git merge main
```

Do not commit: `.env`, API keys, passwords, payment credentials, `node_modules`, uploaded user documents, real customer data.

Keep PRs small and explain **what changed**, **why it changed** and **how to test it**.

---

## 13. Team Ownership

Replace placeholders with real names. Work primarily inside your assigned area and avoid unnecessary changes to shared files.

| Role | Responsibility | Branch |
|---|---|---|
| Team Leader — [Name] | Architecture, review, integration | `main` |
| Frontend Developer — [Name] | React UI and bidder/seller pages | `fe/...` |
| Backend Developer — [Name] | APIs and business logic | `be/...` |
| Database Developer — [Name] | PostgreSQL schema and queries | `be/db` |
| QA/Documentation — [Name] | Testing and documentation | `docs/qa` |

---

## 14. Rules for AI Coding Assistants

1. Read `AGENTS.md` before making changes.
2. Do not invent requirements.
3. Do not invent APIs or database structures.
4. Do not change architecture without approval.
5. Do not add dependencies without approval.
6. Do not modify another developer's area unnecessarily.
7. Do not claim tests were executed unless they actually were.
8. Do not claim deployment succeeded unless it was actually verified.
9. Explain every significant change.
10. Prefer small, reviewable changes.
11. Ask the Team Leader when requirements are ambiguous.
12. Follow the project's existing coding conventions.
13. Never expose secrets or sensitive information.

---

## 15. Definition of Done

A change is done only when every applicable item is satisfied.

| Criteria | Done |
|---|---|
| Requirement implemented | [ ] |
| Validation added | [ ] |
| Authorization checked | [ ] |
| Error handling added | [ ] |
| Security considered | [ ] |
| Tests written and executed | [ ] |
| Real-time behavior verified (where applicable) | [ ] |
| Database migration/schema updated (where applicable) | [ ] |
| API documentation updated | [ ] |
| UI loading, empty and error states handled | [ ] |
| No secrets committed | [ ] |
| Code reviewed | [ ] |
| Pull Request approved | [ ] |
| Documentation updated | [ ] |
