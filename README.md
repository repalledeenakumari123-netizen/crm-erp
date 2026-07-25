# Mini ERP + CRM Operations Portal

A full-stack ERP + CRM system for a wholesale/distribution business: customer relationship management, product & inventory tracking, and a sales challan (delivery note) workflow with atomic stock control.

## Tech Stack

**Backend:** Node.js · Express.js · TypeScript · Prisma ORM · PostgreSQL · JWT Authentication · bcrypt · express-validator

**Frontend:** React · TypeScript · Vite · React Router DOM · Axios · Tailwind CSS · React Hook Form · React Hot Toast · Heroicons

## Project Structure

```
erp-crm-portal/
├── backend/             # Express + TypeScript + Prisma API
├── frontend/            # React + Vite + Tailwind SPA
├── docs/
│   └── (this README covers setup/deployment/assumptions)
├── postman/
│   └── ERP-CRM-Portal.postman_collection.json
└── README.md
```

## Core Modules

### 1. Authentication & Roles
JWT-based login (access + refresh tokens). Four roles: **Admin**, **Sales**, **Warehouse**, **Accounts**. See [Role Permissions](#role-permissions-assumption) below for exactly what each role can do — this was not fully specified in the assignment brief, so it's a documented assumption.

### 2. Customer CRM
Full CRUD, search (name/mobile/email/business/GST), filter by status/type, follow-up note history (append-only log, not just a single field — so you can see the full follow-up trail per customer), customer detail view.

### 3. Product & Inventory
Product CRUD (name, SKU, category, unit price, current stock, minimum stock alert, location), low-stock filter, and a full stock movement log (every IN/OUT change is recorded with reason, actor, and timestamp — including movements generated automatically by confirming/cancelling a challan).

### 4. Sales Challan
- Select customer, add multiple products with quantities
- Challan number auto-generated (`CH-00001`, `CH-00002`, ...)
- Save as **Draft** (no stock impact) or **Confirmed** (stock deducted atomically)
- **Stock never goes negative**: confirming checks every line item's availability *before* deducting anything; if any item is short, the whole confirm fails with a clear 409 error naming the product and the shortfall — no partial deduction ever happens
- **Product snapshots**: each line item stores the product's name/SKU/price *at the time it was added* (`productNameSnapshot`, `productSkuSnapshot`, `unitPriceSnapshot`), independent of the live `Product` row — so historical challans stay accurate even if a product's price changes or the product is later deleted
- Only **Draft** challans can be edited or confirmed; confirmed/cancelled challans are immutable
- Cancelling a **confirmed** challan restores the stock it had deducted (documented assumption, see below)

## Local Setup

### Prerequisites
- Node.js 18+, npm
- PostgreSQL 14+ (local, or a free-tier Supabase/Neon/Render Postgres instance)

### 1. Database
```bash
createdb erp_crm_portal
```

### 2. Backend
```bash
cd backend
npm install
cp .env.example .env
```
Edit `.env`:
```
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/erp_crm_portal?schema=public"
JWT_SECRET=<long random string>
JWT_REFRESH_SECRET=<a different long random string>
CLIENT_URL=http://localhost:5173
```
Then:
```bash
npx prisma generate
npx prisma migrate dev --name init
npm run seed
npm run dev
```
API runs at `http://localhost:5000/api/v1`. Health check: `GET /api/v1/health`.

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env
npm run dev
```
Runs at `http://localhost:5173`.

### 4. Demo credentials (after `npm run seed`)

| Role | Email | Password |
|---|---|---|
| Admin | admin@erpcrm.com | Password@123 |
| Sales | sales@erpcrm.com | Password@123 |
| Warehouse | warehouse@erpcrm.com | Password@123 |
| Accounts | accounts@erpcrm.com | Password@123 |

## API Reference

Base URL: `/api/v1`. All endpoints except `/auth/login` and `/auth/refresh` require `Authorization: Bearer <token>`.

| Module | Endpoints |
|---|---|
| Auth | `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`, `POST /auth/change-password`, `POST /auth/users` (Admin), `GET /auth/users` (Admin) |
| Customers | `GET /customers`, `GET /customers/:id`, `POST /customers`, `PUT /customers/:id`, `POST /customers/:id/follow-ups` |
| Products | `GET /products`, `GET /products/:id`, `POST /products`, `PUT /products/:id`, `POST /products/:id/movements`, `GET /products/movements` |
| Challans | `GET /challans`, `GET /challans/:id`, `POST /challans`, `PUT /challans/:id`, `PUT /challans/:id/confirm`, `PUT /challans/:id/cancel` |

List endpoints support `?page&limit` pagination and `?search=` where applicable. All endpoints validate input (express-validator) and return proper HTTP status codes (400/401/403/404/409/500) with descriptive error messages. Import `postman/ERP-CRM-Portal.postman_collection.json` for ready-to-run requests with example bodies — it auto-captures the access token after "Login".

## Role Permissions (assumption)

The brief specified the four roles but not exactly what each can do. This is the permission model implemented — documented here since it wasn't fully specified:

| Module | Admin | Sales | Warehouse | Accounts |
|---|---|---|---|---|
| Customers | Full access | Create/edit/view, add follow-ups | No access | View only |
| Products | Full access | View only | Create/edit, record stock movements | View only |
| Stock Movement Log | Full access | No access | View + record | View only |
| Sales Challans | Full access | Create/edit(draft)/confirm/cancel | View only | View only |
| User management | Full access | — | — | — |

Rationale: Sales owns the customer relationship and the challan/order process; Warehouse owns physical inventory accuracy; Accounts needs visibility everywhere for billing/reconciliation but shouldn't mutate operational data; Admin is unrestricted.

## Other Documented Assumptions

- **Cancelling a confirmed challan restores stock.** The brief specifies confirm-time stock deduction but doesn't say what happens on cancellation of an already-confirmed challan. We treat cancellation as a full reversal: stock is restored and an `IN` movement is logged with reason `"Challan <number> cancelled"`. Cancelling a still-draft challan is a no-op on stock, since drafts never touched it.
- **Challan numbering** is sequential per-database (`CH-00001`, `CH-00002`, ...), generated inside the same transaction as challan creation. At the concurrency level expected for this assignment this is sufficient; the `challanNumber` column has a unique constraint as a safety net against a race producing a duplicate.
- **Duplicate products in one challan submission** (the same product added twice in one request) are merged into a single line item with combined quantity, rather than rejected or creating two rows.
- **A deleted product** does not delete its historical challan line items — `ChallanItem.productId` is nullable (`onDelete: SetNull`) and the snapshot fields remain, so past challans stay readable even after a product is removed from the catalog. Such items simply can't be part of a *future* confirm/stock action.
- Jira/GST fields, follow-up notes, and stock-movement reasons are free-text; no formatting/validation was specified beyond "GST number, optional" so it's stored as a plain string.

## Deployment

### Recommended free-tier stack
- **Frontend:** Vercel or Netlify (`frontend/` as the project root, build command `npm run build`, output `dist/`)
- **Backend:** Render or Railway (`backend/` as the project root, build command `npm install && npx prisma generate && npm run build`, start command `npm start`)
- **Database:** Supabase, Neon, or Render Postgres (any managed Postgres works — just point `DATABASE_URL` at it)

### Steps
1. Provision a Postgres database on Supabase/Neon/Render and copy its connection string.
2. Deploy `backend/` to Render/Railway:
   - Set environment variables: `DATABASE_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `CLIENT_URL` (your deployed frontend URL), `NODE_ENV=production`.
   - Build command: `npm install && npx prisma generate && npx prisma migrate deploy && npm run build`
   - Start command: `npm start`
   - After first deploy, run the seed script once via the platform's shell/console: `npm run seed`.
3. Deploy `frontend/` to Vercel/Netlify:
   - Set `VITE_API_BASE_URL` to your deployed backend's `/api/v1` URL.
   - Build command: `npm run build`, output directory: `dist`.
4. Update the backend's `CLIENT_URL` env var to the final deployed frontend URL (for CORS) and redeploy the backend if it was set before the frontend URL was known.

### Environment variables summary

**Backend** (`backend/.env`): `PORT`, `NODE_ENV`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `JWT_REFRESH_SECRET`, `JWT_REFRESH_EXPIRES_IN`, `CLIENT_URL`, `BCRYPT_SALT_ROUNDS`.

**Frontend** (`frontend/.env`): `VITE_API_BASE_URL`.

Never commit `.env` files — both are gitignored; only `.env.example` is checked in.

### If not deploying
Per the assignment, if you choose not to deploy: this README's local setup section is the "working local setup" documentation, `postman/ERP-CRM-Portal.postman_collection.json` is the Postman collection, and you'll additionally want to record a short screen capture walking through: login as each role, create a customer + follow-up, create a product, confirm a challan (showing stock deduct), attempt to over-confirm a challan with insufficient stock (showing the 409 error), and cancel a confirmed challan (showing stock restore).

## Known Limitations

- Challan numbering uses a simple count-based sequence rather than a dedicated atomic counter table; under very high concurrent write load there's a theoretical (though unique-constraint-protected) race — acceptable at this project's scale.
- No file/image upload support (e.g. product photos, GST certificate uploads) — out of scope for this assignment.
- No email/SMS notifications for follow-up due dates or low-stock alerts — the UI surfaces these but doesn't push notifications.
- Frontend bundle is a single chunk (~310 KB gzipped ~100 KB); acceptable for this scope but code-splitting would be a next step for a larger app.
- Test coverage: none included — the assignment brief didn't call out automated tests as a requirement, but this would be the natural next addition (e.g. Jest/Supertest on the challan confirm/cancel stock logic, since that's the highest-risk business logic in the system).
