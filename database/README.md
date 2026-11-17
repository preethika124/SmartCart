# Database

Prisma schema: `backend/prisma/schema.prisma`. Migrations are additive and tracked under `backend/prisma/migrations/`. The integrity migration adds customer preferences, order address snapshots/idempotency hashes, review uniqueness, and database checks for product prices, inventory counts, coupon discount/use counts and review scores.

For local development, create a PostgreSQL database named by `DATABASE_URL`, then run `npm run db:migrate -w backend` and `npm run db:seed -w backend`. The seed uses unique SKUs/category slugs and does not overwrite stock quantities when re-run. Orders preserve item prices and delivery address snapshots; inventory is decremented/restored transactionally. Do not run development migrations against production: use `npx prisma migrate deploy` there.
