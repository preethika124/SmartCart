# SmartCart workflow and code map

## Runtime flow

1. **Storefront boot:** `frontend/src/App.tsx` loads session, categories and cart/wishlist state from the API. The catalogue pages call `/products/search`; product pages call `/products/:id`. UI state is not backed by localStorage.
2. **Customer session:** registration/login are handled by `/auth/register` and `/auth/login`. The API sets an HttpOnly session cookie. Protected pages wait for `/me`, then route unauthenticated customers to sign in.
3. **Persistent shopping:** cart and wishlist operations are authenticated and owner-scoped. Server stock is rechecked at cart preview and again during order creation.
4. **Delivery:** address book routes store and delete customer-owned addresses. Checkout chooses one owned address, which is copied into the eventual order as a delivery snapshot.
5. **Review and payment:** `/orders/preview` calculates item prices, coupon discount, tax, delivery and availability. The user explicitly approves the displayed preview. `/orders` checks that exact preview again, decrements stock and increments coupon use within a serializable transaction, writes order/items/demo payment, and clears cart. An idempotency key prevents a retry from making a duplicate order.
6. **After order:** order history/detail are owner scoped. Cancellation locks and conditionally transitions the order, refunds the simulated payment, and restores inventory in the same transaction; repeat cancellation does not re-stock twice.
7. **Independent agent:** `/agent` routes require a private machine key and scope; account data and order actions also require that customer's JWT. They share the customer order service and approval checks.

## Source map

- `frontend/src/main.tsx` — React root, BrowserRouter and global stylesheet.
- `frontend/src/App.tsx` — visual components and routes; API-backed product/category search, comparisons, reviews, auth, cart, wishlist, addresses, checkout, orders and profile.
- `frontend/src/api.ts` — Axios base URL, credentials, envelope helper and session-expiry handling.
- `frontend/src/style.css` — existing storefront layout, colors, responsive rules and form/error states.
- `backend/src/index.ts` — Express middleware, auth, schemas, routes, transaction handlers, logs, health checks and graceful shutdown.
- `backend/prisma/schema.prisma` — PostgreSQL relational model.
- `backend/prisma/migrations/` — initial schema and additive integrity migration.
- `backend/prisma/seed.ts` — repeatable catalogue/categories/demo coupon/review seed; importantly, it does not reset inventory on rerun.
- `frontend/src/App.test.tsx` — mocked API component smoke test.
- `tests/specs/shop.spec.ts` — Playwright HTTP-mocked UI flows; not a substitute for PostgreSQL integration checks.

## Environment and setup

Copy `.env.example` to `backend/.env`; edit it before starting. `DATABASE_URL` must connect to an existing PostgreSQL 14+ database. `JWT_SECRET` must be a private random value of at least 32 characters and must not contain the placeholder. Keep `AGENT_API_KEY` private; `AGENT_API_SCOPES` controls agent permissions. `FRONTEND_URL` must exactly match the browser origin; `PORT` defaults to 4001. `VITE_API_URL` is optional and may only contain a public API URL, never a secret.

```powershell
npm install
npm run db:generate -w backend
npm run db:migrate -w backend
npm run db:seed -w backend
npm run dev
```

Health: `http://localhost:4001/api/health`. Customer UI: `http://localhost:5174`.

## Deployment

Set `NODE_ENV=production`, serve the frontend and API over HTTPS, configure strong secrets and an exact `FRONTEND_URL`, and use managed PostgreSQL with backups. From the backend folder, use `npx prisma migrate deploy`; do not use `migrate dev` against production. Build with `npm run build`, run the API using `npm start` in `backend`, and serve `frontend/dist` with an SPA fallback to `index.html`. The demo payment path must remain marked simulated unless a separately authorized payment integration is implemented.
