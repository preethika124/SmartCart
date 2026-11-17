# SmartCart REST API

Base URL: `http://localhost:4001/api`. Responses use `{success,data,error,requestId}`; failures have the same envelope and a safe message. Customer browser sessions use an HttpOnly cookie; API clients and the independent agent use short-lived JWT bearer tokens. A machine `x-agent-key` is required for `/agent` routes and does not replace customer identity.

## Public catalogue

- `GET /health` — checks API and PostgreSQL availability.
- `GET /categories` — catalogue categories and product counts.
- `GET /products/search?q=&category=&brand=&minPrice=&maxPrice=&minRating=&specName=&specValue=&sort=&limit=&cursor=` — filtered catalogue page. Sort values: `price_asc`, `price_desc`, `rating`, `popular`, `newest`; response data is `{items,nextCursor}`.
- `GET /products/:id` — product, specifications, images, inventory and recent reviews.
- `GET /products/:id/stock` — available count (`quantity - reserved`).
- `POST /products/compare` — `{ "ids": ["id1", "id2"] }`, 2–4 distinct existing products.

## Authentication and customer profile

- `POST /auth/register` — `{email,name,password}`. Password needs 10+ characters with a letter and digit.
- `POST /auth/login` — `{email,password}`.
- `POST /auth/logout` — clears the HttpOnly session cookie.
- `GET /me`, `PATCH /me` — current customer and profile fields (`name`, `phone`, `preferredCategory`).

Registration and login set `smartcart_session` (HttpOnly, SameSite Strict, 12-hour expiry) and return a short-lived JWT in the API response for API clients. Browser code uses the cookie and never stores tokens in localStorage. Passwords use bcrypt.

## Cart, wishlist, reviews and addresses (customer JWT/session required)

- `GET /cart`; `POST /cart/items` with `{productId,quantity}` (sets the quantity, range 1–10); `DELETE /cart/items/:cartItemId`.
- `GET /wishlist`; `POST /wishlist/:productId`; `DELETE /wishlist/:productId`.
- `POST /products/:id/reviews` with `{rating,title,comment}`. Only a customer with a confirmed purchase can review; one review per customer/product.
- `GET /addresses`; `POST /addresses` with `{name,phone,line1,city,state,postalCode,isDefault?}`; `DELETE /addresses/:id`.

All changes are scoped to the authenticated customer. Product/catalogue errors use 404, invalid input 422, stock/coupon conflicts 409.

## Coupons and order flow

`POST /coupons/validate` accepts `{code}` and checks it against the caller's current cart. `POST /orders/preview` accepts `{couponCode?,addressId?}` and does not write an order. It validates stock, coupon availability and address ownership, and calculates current prices, discount, tax and delivery from database records.

After the customer reviews the full preview, create the order with `Idempotency-Key` (16–128 characters) and:

```json
{
  "confirmed": true,
  "paymentApproval": "DEMO_APPROVED",
  "couponCode": "SMART10",
  "addressId": "customer-address-id",
  "expectedPreview": { "...": "the exact preview response" }
}
```

The API rechecks the exact preview and performs inventory decrement, coupon-use increment, order/item/payment creation and cart clearing inside a serializable transaction. Stock decrement is conditional, concurrent conflicts roll back, and idempotency keys are bound to customer plus reviewed request content. `DEMO_APPROVED` creates a simulated payment record only. It is not a gateway credential.

- `GET /orders`; `GET /orders/:id` — owner-scoped history/detail.
- `POST /orders/:id/cancel` — atomically changes confirmed order to cancelled, marks simulated payment refunded, and restores stock once. Repeated cancellation is idempotent.

## Independent AgentPlatform routes

All `/agent` paths require `x-agent-key` and an allowed `AGENT_API_SCOPES` scope. Private customer operations additionally require the customer's bearer JWT.

- `GET /agent/health` (`products:read`)
- `GET /agent/products/search`, `GET /agent/products/:id`, `GET /agent/products/:id/stock`, `POST /agent/products/compare` (`products:read`)
- `GET /agent/cart`, `POST /agent/cart/items` (`cart:write`, customer JWT)
- `POST /agent/orders/preview`, `POST /agent/orders` (`orders:write`, customer JWT; create uses the same confirmation, exact-preview and idempotency checks)
- `GET /agent/orders/:id` (`orders:read`, customer JWT)
- `POST /agent/orders/:id/cancel` (`orders:write`, customer JWT)

The key and scopes are server-side environment variables only. Give AgentPlatform its own machine key and only consented customer JWTs. Rotate credentials and use TLS in deployment.
