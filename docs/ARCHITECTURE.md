# SmartCart architecture

```mermaid
flowchart LR
 Browser[React + Vite storefront] -->|JSON REST + customer JWT| API[Independent Express API]
 Agent[External future agent] -->|x-agent-key + scope + customer JWT| API
 API --> Prisma[Prisma ORM]
 Prisma --> DB[(SmartCart PostgreSQL)]
 API --> Pay[Simulated payment approval]
```

SmartCart is independently deployable. It contains no AI agent, travel, or event booking services. The API validates identity, scope, coupons, stock, order confirmation and idempotency. Order creation runs in a database transaction. Payment is simulated only.
