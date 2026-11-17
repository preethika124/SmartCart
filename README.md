# SmartCart

SmartCart is an independent electronics marketplace. The React/Vite storefront uses a separate Express/TypeScript API backed by PostgreSQL and Prisma. Product catalogue, stock, customer cart, wishlist, profile, addresses, reviews and orders are served from the API. Checkout is simulated only: no card number, OTP or UPI PIN is requested or processed.

## Start locally

Requirements: Node.js 20+, npm and PostgreSQL 14+.

```powershell
cd C:\Users\penta\Documents\PROJECTS\final_project\SmartCart
Copy-Item .env.example backend\.env
```

Edit `backend\.env`: set `DATABASE_URL`, replace `JWT_SECRET` with at least 32 random characters, set a private `AGENT_API_KEY`, and set the browser origin/port. Create the named PostgreSQL database before migration. A secret can be generated with:

```powershell
node -e "console.log(require('node:crypto').randomBytes(48).toString('base64url'))"
```

Then run:

```powershell
npm install
npm run db:generate -w backend
npm run db:migrate -w backend
npm run db:seed -w backend
npm run dev
```

Storefront: http://localhost:5174 · API: http://localhost:4001 · Health: http://localhost:4001/api/health

## Main workflow

Search catalogue → inspect a product and verified reviews → sign in → persist items in cart/wishlist → save a delivery address → preview server-calculated price/coupon/tax/delivery → explicitly confirm simulated payment → inspect order history or cancel. Order creation and cancellation are transactional, owner scoped, stock-aware and idempotent.

See [docs/API.md](docs/API.md) for endpoint bodies and [docs/WORKFLOW.md](docs/WORKFLOW.md) for setup, architecture and production procedures. `docs/ARCHITECTURE.md` and `database/README.md` describe the separation and persistence model.

## Checks

```powershell
npm run build
npm run test -w frontend
npm run test:e2e -w tests
```

The Playwright checks mock HTTP responses to verify browser interactions; use PostgreSQL-backed manual/API verification after configuring `backend\.env` and applying migrations. Install Chromium once if needed: `npx playwright install chromium`.

Production deployment needs HTTPS, a managed PostgreSQL database, unique strong secrets, `NODE_ENV=production`, and `FRONTEND_URL` set to the exact storefront origin. Apply migrations with `cd backend; npx prisma migrate deploy`. To allow the clearly labeled simulated checkout in production, set `PAYMENT_MODE=demo` and `ALLOW_SIMULATED_CHECKOUT_IN_PRODUCTION=true` in the backend environment. It remains disabled without both values. This records a demo payment only; it does not charge customers or replace a real payment integration.
