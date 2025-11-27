# Shopify Multi-Tenant Data Ingestion & Insights — MVP

This repository implements a production-minded, end-to-end MVP for a multi-tenant Shopify Data Ingestion & Insights service. It is built as a single monorepo with a Node.js (Express + Sequelize) backend and a React dashboard frontend.

Key features
- Multi-tenant isolation using `tenantId`
- Tenant onboarding API with secure admin user creation
- Shopify REST Admin ingestion (Customers, Products, Orders) per-tenant
- Webhook endpoint for `orders/create` with HMAC verification (configurable)
- Hourly cron sync that updates each tenant sequentially
- Analytics endpoints: summary, orders-by-date, top-customers
- React dashboard: email login, JWT-based auth, KPI cards, trend chart, top-customers table, date-range filter
- Ready for demo: clear logging, structured JSON errors, and a small migration SQL to create the schema

Assumptions
- Shopify access tokens are provided at tenant onboarding (this MVP uses the access token directly). For production, use Shopify OAuth.
- Single MySQL instance accessible to the backend.
- HMAC secret for webhooks is optional but strongly recommended — set `SHOPIFY_WEBHOOK_SECRET` to enable verification.
- Demo scale (few tenants) — cron runs serially; production should use queues and rate limiting.

ASCII Architecture Diagram
```
+----------------------+       +------------------+       +------------------+
|  Shopper / Shopify   | <---> |  Tenant (Shopify) | <---> | Backend API (Node)|
+----------------------+       +------------------+       +------------------+
                                                            |  Sequelize (MySQL)
                                                            |
                                                      +-----------+
                                                      | Frontend  |
                                                      | React UI  |
                                                      +-----------+
```

API List (major endpoints)
- POST /api/auth/register
  - Create tenant + admin user (provide storeDomain and accessToken)
- POST /api/auth/login
  - Tenant-scoped login (returns JWT)
- POST /api/shopify/sync-now
  - Trigger immediate sync for authenticated tenant
- POST /api/shopify/webhooks/orders-create
  - Webhook receiver for order created (HMAC verification optional)
- GET /api/analytics/summary
  - Totals: revenue, orders count, avg order value, customers count
- GET /api/analytics/orders-by-date?startDate=&endDate=
  - Returns day-by-day counts and revenue in range
- GET /api/analytics/top-customers
  - Top 5 customers by spend

MySQL schema (high-level)
- tenants (id, name, storeDomain, accessToken, timezone, metadata, timestamps)
- users (id, tenantId, email, passwordHash, role, timestamps)
- customers (id, tenantId, shopifyId, email, firstName, lastName, phone, raw JSON, timestamps)
- products (id, tenantId, shopifyId, title, sku, price, raw JSON, timestamps)
- orders (id, tenantId, shopifyId, orderNumber, totalPrice, currency, customerId, raw JSON, createdAtShopify, timestamps)

Local setup (quick)
1. Start MySQL and create DB using migration:
   - mysql -u root -p < migrations/001_init_schema.sql
2. Backend
   - cd backend
   - cp .env.example .env (edit credentials)
   - npm install
   - npm start
3. Frontend
   - cd frontend
   - npm install
   - npm run dev
4. Register a tenant (example):
   POST /api/auth/register
   {
     "tenantName":"Demo Store",
     "storeDomain":"demo-shop.myshopify.com",
     "accessToken":"shopify-admin-token",
     "email":"admin@example.com",
     "password":"StrongPass123"
   }
5. Login via dashboard with tenantId from register result.

Known limitations
- Shopify pagination: implemented using Link header; works for standard Admin API but more robust retry/backoff and rate-limit handling is needed for production.
- No automated migrations tooling (we provide SQL). Use Flyway / Liquibase in production.
- Background sync is sequential and simple; for scale, replace with a worker queue (Bull, RabbitMQ) and per-tenant rate limiting.
- HMAC verification is optional. For security, always set `SHOPIFY_WEBHOOK_SECRET`.

How to improve
- Implement full Shopify OAuth flow for install and token refresh
- Add per-tenant role-based access control
- Add metrics & tracing (Prometheus, OpenTelemetry)
- Use migrations and CI for schema changes
- Add unit / integration tests and end-to-end flows

Checklist (confirm after running)
  1. MySQL migration SQL executed  
  2. npm install & npm start runs backend  
  3. npm run dev runs React dashboard  
  4. Summary metrics show correct totals  
  5. Webhook inserts new order + updates revenue  
  6. Tenant data is never mixed  
  7. No unfinished or placeholder logic anywhere

Enjoy the demo-quality MVP. If you'd like, I can add the OAuth flow and CI pipeline next.