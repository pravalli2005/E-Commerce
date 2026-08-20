# Multi-Tenant E-Commerce API

REST API built with Node.js, Express.js, Mongoose, and MongoDB.

## Tenant Storage

This API uses one shared MongoDB database with logical tenant isolation. Every tenant-owned model (`User`, `Product`, `Category`, `Cart`, `Order`, `Payment`, and `Review`) contains:

```js
tenantId: {
	type: mongoose.Schema.Types.ObjectId,
	ref: 'Tenant',
	required: true,
	index: true
}
```

Authenticated requests establish `req.user.tenantId`. Controllers and services must include that value in every tenant-owned filter and write. A missing tenant filter is a data-isolation defect, not an optional optimization.

## Product Request Chain

```text
Request
	-> Auth Middleware
	-> Role Middleware
	-> Tenant Middleware
	-> Product Controller
	-> Product Service
	-> Mongoose
	-> MongoDB
```

Tenant middleware always derives `req.tenantId` from `req.user.tenantId`. Any `tenantId` sent by the client is ignored, so clients cannot select another tenant's data.

## Request Flow

```text
User
	-> Register / Login
	-> Express REST API
	-> Authentication Middleware
			 - Verify JWT and active user
			 - Identify role
	-> Tenant Middleware
	-> Tenant-isolated Controller
	-> Service / Business Logic
	-> Mongoose Model
	-> MongoDB
	-> JSON REST Response
```

### Login Flow

```text
POST /api/auth/login
	-> Validate email and password
	-> Find active user
	-> Compare password with passwordHash
	-> Generate JWT authentication token
	-> Return token and user details
```

### Vendor Registration Flow

```text
POST /api/auth/register
	-> Validate vendor and tenant data
	-> Create Tenant (MongoDB generates tenantId)
	-> Hash password
	-> Create User with role = vendor and tenantId
	-> Return JWT, tenantId, and vendor details
	-> Vendor uses the token for the dashboard
```

The vendor dashboard data is available from `GET /api/analytics/summary`. It requires the returned bearer token and is automatically scoped to that vendor's `tenantId`.

Authentication and tenant isolation are separate responsibilities. Authentication establishes `req.user` and `req.tenantId`; tenant middleware requires that context before a protected controller runs. Controllers and services must include `tenantId` in every tenant-owned query and write.

## Roles

- `super_admin`: manages the SaaS platform.
- `vendor`: manages one store's products, categories, and orders.
- `customer`: can browse products and manage their own orders.

Platform administration should use explicit platform-only routes; it must not bypass tenant filters on normal store endpoints.

## Run

1. Install Node.js 18+ and MongoDB.
2. Start the MongoDB service before starting the API. On Windows, run PowerShell as Administrator and use `Start-Service MongoDB` if MongoDB was installed as a service.
3. Copy `.env.example` to `.env` and set `MONGO_URI` and a strong `JWT_SECRET`.
4. Run `npm install`, then `npm run dev`.

If startup reports `ECONNREFUSED 127.0.0.1:27017`, MongoDB is not listening on the local default port. Start the service, run `mongod` directly, or replace `MONGO_URI` with a reachable MongoDB Atlas connection string.

The API runs on `http://localhost:4000` by default.

## Endpoints

- `GET /health`
- `POST /api/auth/register` creates a tenant and its vendor.
- `POST /api/auth/login`
- `POST /api/products` (vendor only)
- `GET /api/products` (authenticated users)
- `GET /api/products/:id` (authenticated users)
- `PUT /api/products/:id` (vendor or super admin)
- `DELETE /api/products/:id` (vendor or super admin)
- `GET|POST /api/categories`
- `GET /api/cart` (customer cart)
- `POST /api/cart/items` (customer adds a product)
- `PUT /api/cart` (customer replaces cart items)
- `POST /api/orders/checkout` (customer converts cart to order)
- `GET /api/tenants/current`
- `GET /api/vendors`
- `PATCH|DELETE /api/products/:id`
- `GET|POST /api/orders`
- `PATCH /api/orders/:id/status`
- `POST /api/payments/create` (customer starts payment)
- `POST /api/payments/webhook` (provider callback with webhook secret)
- `GET /api/analytics/summary`

Payment creation is isolated behind the payment service:

```text
POST /api/payments/create
	-> Payment Controller
	-> Payment Service
	-> Stripe Checkout
	-> Payment metadata in MongoDB
	-> Payment result updates Payment and Order
```

The request accepts an `orderId` and optional `currency`. The API never accepts or stores raw card numbers, CVV, or other card credentials. Payment documents store only `tenantId`, `orderId`, `amount`, `currency`, `status`, `provider`, and `transactionReference`.

Sales analytics are available at `GET /api/analytics/sales`. Its aggregation starts with `{ $match: { tenantId: req.user.tenantId, paymentStatus: 'paid' } }` through tenant middleware, then returns totals, revenue, orders by date, and best products for that vendor only.

## Complete Request Lifecycle

```text
HTTP Request
	-> Express Router
	-> Auth Middleware
	-> Role Middleware
	-> Tenant Middleware
	-> Controller
	-> Service
	-> Mongoose Model
	-> MongoDB
	-> JSON Response
```

Vendor product creation requires a bearer token and accepts:

```http
POST /api/products
Authorization: Bearer <vendor-token>
Content-Type: application/json
```

```json
{
	"name": "Laptop",
	"price": 55000,
	"stock": 20,
	"category": "Electronics"
}
```

The API generates the product slug, stores `stock` as `inventory`, and assigns the authenticated vendor's `tenantId`.

Customer shopping uses `POST /api/cart/items` with `{ "productId": "...", "quantity": 1 }`, then `POST /api/orders/checkout` with an optional shipping address. The cart stores each item's price and recalculates `totalAmount`; stock is checked within the authenticated tenant before adding an item. The checkout response includes the created order and the payment checkout URL.

Orders begin as `pending`, become `confirmed` after successful payment, then move through `processing`, `shipped`, and `delivered`. A failed payment uses `payment_failed`; payment status is recorded separately as `unpaid`, `paid`, `failed`, or `refunded`.

Send the login token with every subsequent protected request:

```http
Authorization: Bearer <token>
```

Example:

```bash
curl http://localhost:4000/api/products \
	-H "Authorization: Bearer <token>"
```

Every tenant-owned query is scoped by the authenticated user's `tenantId`; roles are `super_admin`, `vendor`, and `customer`.
