# ProShop — Architecture Document

This document describes the complete architecture of the ProShop eCommerce platform: technology stack, structure, data models, APIs, authentication, frontend flows, deployment, and operational aspects.

---

## 1. Executive Summary

ProShop is a full-stack eCommerce application built with the **MERN** stack (MongoDB, Express.js, React, Node.js). It provides:

- **Customer**: Product browsing, search, pagination, cart, checkout (shipping, payment), order history, product reviews, and user profile.
- **Admin**: Product CRUD, user management, order list, mark orders as delivered, and product image upload.

The application runs as a **single repository** with a separate backend (Node/Express) and frontend (Create React App). In production, the Express server serves both the API and the static frontend build; in development, frontend and backend run concurrently with a proxy from the React dev server to the API.

---

## 2. High-Level Architecture

### 2.1 Top-Level Layout

```
proshop/
├── backend/           # Node.js Express API
├── frontend/          # React SPA (CRA)
├── uploads/           # Static file uploads (product images)
├── docs/              # Documentation (this file)
├── package.json       # Root scripts, backend deps, Heroku build
├── Procfile           # Heroku process definition
└── README.md
```

- **Not a monorepo**: No npm workspaces; `frontend` has its own `package.json` and is built separately.
- **Single process in production**: One Node process runs `backend/server.js`, serves `/api/*` and static files from `frontend/build`, and handles SPA fallback.

### 2.2 Request Flow (Production)

```
Client (browser)
    → Same-origin request to /
    → Express: static file from frontend/build or index.html (SPA)
Client (browser)
    → Same-origin request to /api/*
    → Express: JSON API (auth, products, orders, users, upload, config)
    → MongoDB (read/write)
```

### 2.3 Request Flow (Development)

```
React dev server (port 3000)
    → Serves UI; "proxy" in frontend/package.json forwards /api/* to http://127.0.0.1:5000
Backend (port 5000)
    → Serves only API and GET / → "API is running...."
```

---

## 3. Technology Stack

### 3.1 Backend

| Concern        | Technology        | Version (from package.json) |
|----------------|-------------------|-----------------------------|
| Runtime        | Node.js           | ESM (`"type": "module"`)    |
| Framework      | Express           | ^4.17.1                     |
| Database       | MongoDB           | Via Mongoose ^5.10.6        |
| ODM            | Mongoose          | ^5.10.6                     |
| Auth           | JWT               | jsonwebtoken ^8.5.1         |
| Password hash  | bcryptjs          | ^2.4.3                      |
| File upload    | multer            | ^1.4.2                      |
| Async errors   | express-async-handler | ^1.1.4                  |
| Logging        | morgan            | ^1.10.0 (dev only)          |
| Env            | dotenv            | ^8.2.0                      |
| Dev            | nodemon           | ^2.0.4                      |

- **No API versioning**: All routes under `/api/` (no `/api/v1`).
- **No TypeScript**: Backend is plain JavaScript with ESM (`.js` imports must include `.js` extension).

### 3.2 Frontend

| Concern          | Technology           | Version (from frontend/package.json) |
|------------------|----------------------|--------------------------------------|
| Framework        | React                | ^16.13.1                             |
| Build            | Create React App     | react-scripts 3.4.3                  |
| Routing          | React Router DOM     | ^5.2.0                               |
| State            | Redux + redux-thunk  | ^4.0.5, ^2.3.0                       |
| HTTP client      | Axios                | ^0.20.0                              |
| UI               | React Bootstrap      | ^1.3.0                               |
| PayPal           | react-paypal-button-v2 | ^2.6.2                             |
| Meta/SEO         | react-helmet         | ^6.1.0                               |
| Testing (present)| Jest + React Testing Library | CRA default (no tests in repo) |

- **No Next.js**: Classic SPA with client-side routing only.

### 3.3 Database

- **MongoDB** only; no Redis, no separate cache.
- **Mongoose** for schemas and queries; no formal migrations (schema-only evolution).

### 3.4 External Services

- **PayPal**: Client-side SDK (react-paypal-button-v2); backend exposes `PAYPAL_CLIENT_ID` via `GET /api/config/paypal`. Payment confirmation is recorded by `PUT /api/orders/:id/pay` (no server-side PayPal webhook or server-side SDK in this codebase).

---

## 4. Backend Architecture

### 4.1 Entry Point and Bootstrap

- **Entry**: `backend/server.js`
- **Bootstrap order**:
  1. `dotenv.config()`
  2. `connectDB()` (MongoDB; `process.exit(1)` on failure)
  3. Create Express app
  4. Middleware: `morgan('dev')` (development only), `express.json()`
  5. Mount routes (see below)
  6. PayPal config: `GET /api/config/paypal`
  7. Static: `/uploads` → `uploads/` directory
  8. Production only: `express.static('frontend/build')`, then `GET *` → `frontend/build/index.html`
  9. Development only: `GET /` → "API is running...."
  10. `notFound`, `errorHandler`
  11. `app.listen(PORT)` — default port **5000** (`process.env.PORT`).

### 4.2 Database Configuration

- **File**: `backend/config/db.js`
- **Connection**: `mongoose.connect(process.env.MONGO_URI, { useUnifiedTopology: true, useNewUrlParser: true, useCreateIndex: true })`
- **Env**: `MONGO_URI` must be set; no central config module; no connection pooling tuning.

### 4.3 Data Models (Mongoose)

All models live in `backend/models/` and use Mongoose schemas with `timestamps: true`.

#### User (`userModel.js`)

| Field      | Type    | Required | Default | Notes                    |
|------------|---------|----------|---------|--------------------------|
| name       | String  | yes      | —       |                          |
| email      | String  | yes      | —       | unique                   |
| password   | String  | yes      | —       | hashed pre-save (bcrypt) |
| isAdmin    | Boolean | yes      | false   |                          |
| createdAt  | Date    | —        | auto    |                          |
| updatedAt  | Date    | —        | auto    |                          |

- **Methods**: `matchPassword(enteredPassword)` — bcrypt compare.
- **Pre-save**: If `password` is modified, hash with `bcrypt.genSalt(10)` and replace.

#### Product (`productModel.js`)

| Field        | Type     | Required | Default | Notes                    |
|--------------|----------|----------|---------|--------------------------|
| user         | ObjectId | yes      | —       | ref: 'User' (creator)    |
| name         | String   | yes      | —       |                          |
| image        | String   | yes      | —       | path (e.g. /uploads/…)   |
| brand        | String   | yes      | —       |                          |
| category     | String   | yes      | —       |                          |
| description  | String   | yes      | —       |                          |
| reviews      | [Review] | —        | []      | sub-document array       |
| rating       | Number   | yes      | 0       | aggregate                |
| numReviews   | Number   | yes      | 0       |                          |
| price        | Number   | yes      | 0       |                          |
| countInStock | Number   | yes      | 0       |                          |
| createdAt    | Date     | —        | auto    |                          |
| updatedAt    | Date     | —        | auto    |                          |

**Review sub-schema**: `name`, `rating`, `comment`, `user` (ref: User), `timestamps`.

#### Order (`orderModel.js`)

| Field            | Type     | Required | Default | Notes                    |
|------------------|----------|----------|---------|--------------------------|
| user             | ObjectId | yes      | —       | ref: 'User'              |
| orderItems       | [Item]   | —        | —       | see below                |
| shippingAddress  | Object   | yes      | —       | address, city, postalCode, country |
| paymentMethod    | String   | yes      | —       | e.g. PayPal              |
| paymentResult    | Object   | —        | —       | id, status, update_time, email_address (optional) |
| taxPrice         | Number   | yes      | 0       |                          |
| shippingPrice    | Number   | yes      | 0       |                          |
| totalPrice       | Number   | yes      | 0       |                          |
| isPaid           | Boolean  | yes      | false   |                          |
| paidAt           | Date     | —        | —       |                          |
| isDelivered      | Boolean  | yes      | false   |                          |
| deliveredAt      | Date     | —        | —       |                          |
| createdAt        | Date     | —        | auto    |                          |
| updatedAt        | Date     | —        | auto    |                          |

**Order item**: `name`, `qty`, `image`, `price`, `product` (ref: Product).

**Note**: The order controller sends `itemsPrice` in the body; the Order schema does not define `itemsPrice`. Mongoose may store it if strict mode allows; consider aligning schema and API.

### 4.4 API Routes Summary

| Mount              | File              | Routes |
|--------------------|-------------------|--------|
| `/api/products`    | productRoutes.js  | See table below |
| `/api/users`       | userRoutes.js     | See table below |
| `/api/orders`      | orderRoutes.js    | See table below |
| `/api/upload`      | uploadRoutes.js   | POST / (single file, no auth) |
| (server.js)        | —                 | GET /api/config/paypal |

#### Product routes

| Method | Path             | Auth        | Controller / behavior        |
|--------|------------------|------------|------------------------------|
| GET    | /                | none       | getProducts (list, pagination, keyword) |
| POST   | /                | protect, admin | createProduct            |
| GET    | /top             | none       | getTopProducts               |
| GET    | /:id             | none       | getProductById               |
| PUT    | /:id             | protect, admin | updateProduct            |
| DELETE | /:id             | protect, admin | deleteProduct            |
| POST   | /:id/reviews     | protect    | createProductReview          |

#### User routes

| Method | Path     | Auth        | Controller / behavior   |
|--------|----------|------------|--------------------------|
| POST   | /        | none       | registerUser             |
| POST   | /login   | none       | authUser                 |
| GET    | /profile | protect    | getUserProfile           |
| PUT    | /profile | protect    | updateUserProfile        |
| GET    | /        | protect, admin | getUsers             |
| GET    | /:id     | protect, admin | getUserById           |
| PUT    | /:id     | protect, admin | updateUser            |
| DELETE | /:id     | protect, admin | deleteUser            |

#### Order routes

| Method | Path          | Auth        | Controller / behavior   |
|--------|---------------|------------|--------------------------|
| POST   | /             | protect    | addOrderItems            |
| GET    | /myorders     | protect    | getMyOrders              |
| GET    | /             | protect, admin | getOrders            |
| GET    | /:id          | protect    | getOrderById             |
| PUT    | /:id/pay      | protect    | updateOrderToPaid        |
| PUT    | /:id/deliver  | protect, admin | updateOrderToDelivered |

### 4.5 Authentication and Authorization

- **Mechanism**: JWT only (no sessions, no OAuth).
- **Token generation**: `backend/utils/generateToken.js` — `jwt.sign({ id }, process.env.JWT_SECRET, { expiresIn: '30d' })`.
- **Middleware**: `backend/middleware/authMiddleware.js`:
  - **protect**: Reads `Authorization: Bearer <token>`, verifies with `JWT_SECRET`, loads user with `User.findById(decoded.id).select('-password')`, sets `req.user`. On missing/invalid token: 401, "Not authorized, token failed" or "Not authorized, no token".
  - **admin**: Ensures `req.user` exists and `req.user.isAdmin === true`; otherwise 401 "Not authorized as an admin". Must be used after `protect`.
- **Upload route**: No auth; anyone can POST to `/api/upload`. Frontend may send token for other reasons; backend does not validate it for upload.

### 4.6 File Upload

- **File**: `backend/routes/uploadRoutes.js`
- **Engine**: Multer with `diskStorage`; destination `uploads/`; filename `{fieldname}-{Date.now()}{ext}`.
- **Filter**: Allowed types jpg, jpeg, png (by extension and mimetype). Otherwise "Images only!".
- **Route**: `POST /`, `upload.single('image')`. Response: string path, e.g. `/${req.file.path}` (e.g. `/uploads/image-1234567890.jpg`).
- **No size limit** specified in code (Multer default applies).
- **Serving**: `GET /uploads/*` served by Express from `uploads/` directory.

### 4.7 Error Handling

- **File**: `backend/middleware/errorMiddleware.js`
- **notFound**: For any request that did not match a route, sets status 404 and passes `Error('Not Found - ' + req.originalUrl)` to next.
- **errorHandler**: Sets status to `res.statusCode` if not 200, else 500; responds with JSON: `{ message: err.message, stack: process.env.NODE_ENV === 'production' ? null : err.stack }`.
- **Async errors**: Controllers use `asyncHandler` so thrown errors are passed to the error handler.

### 4.8 Seeding

- **Script**: `node backend/seeder.js` (or `npm run data:import` / `npm run data:destroy` with `-d` to destroy).
- **Data**: `backend/data/users.js`, `backend/data/products.js`.
- **Behavior**: Deletes all Orders, Products, Users; inserts users and products (products reference first user). No migration system.

### 4.9 Environment Variables (Backend)

| Variable         | Purpose                    | Required |
|------------------|----------------------------|----------|
| NODE_ENV         | development / production   | No (affects logging, static, error stack) |
| PORT             | Server port                | No (default 5000) |
| MONGO_URI        | MongoDB connection string  | Yes      |
| JWT_SECRET       | JWT signing                 | Yes      |
| PAYPAL_CLIENT_ID | Exposed to frontend        | For PayPal UI |

- Loaded via `dotenv.config()` in `server.js`. No validation or `.env.example` in repo (README documents vars).

---

## 5. Frontend Architecture

### 5.1 Entry and Shell

- **Entry**: `frontend/src/index.js` — ReactDOM.render with `Provider` (Redux store), `App`, and global styles (`bootstrap.min.css`, `index.css`).
- **App**: `frontend/src/App.js` — `BrowserRouter`, global `Header` and `Footer`, `Container`, and route list (see below). No shared `PrivateRoute` or `AdminRoute` component; each screen checks auth and redirects.

### 5.2 Routing (React Router v5)

| Path                              | Component        | Access / notes              |
|-----------------------------------|------------------|-----------------------------|
| /                                 | HomeScreen       | Public; pagination via /page/:pageNumber |
| /search/:keyword                  | HomeScreen       | Public                      |
| /page/:pageNumber                 | HomeScreen       | Public                      |
| /search/:keyword/page/:pageNumber | HomeScreen       | Public                      |
| /product/:id                      | ProductScreen    | Public                      |
| /cart/:id?                       | CartScreen       | Public (id optional)        |
| /login                            | LoginScreen      | Public                      |
| /register                         | RegisterScreen   | Public                      |
| /profile                          | ProfileScreen    | Protected                   |
| /shipping                         | ShippingScreen   | Protected (checkout)        |
| /payment                          | PaymentScreen    | Protected (checkout)        |
| /placeorder                       | PlaceOrderScreen | Protected (checkout)        |
| /order/:id                        | OrderScreen      | Protected                   |
| /admin/userlist                   | UserListScreen   | Admin                       |
| /admin/user/:id/edit              | UserEditScreen   | Admin                       |
| /admin/productlist                | ProductListScreen| Admin                       |
| /admin/productlist/:pageNumber    | ProductListScreen| Admin                       |
| /admin/product/:id/edit           | ProductEditScreen| Admin                       |
| /admin/orderlist                  | OrderListScreen  | Admin                       |

- **Route order**: More specific routes (e.g. `/order/:id`, `/admin/*`) are declared before the generic `/` so they match correctly.

### 5.3 State Management (Redux)

- **Store**: `frontend/src/store.js` — single store, `combineReducers`, `thunk`, `composeWithDevTools`.
- **Initial state from localStorage**:
  - `cartItems` → `cart.cartItems`
  - `userInfo` → `userLogin.userInfo`
  - `shippingAddress` → `cart.shippingAddress`
- **Reducers** (slices): productList, productDetails, productDelete, productCreate, productUpdate, productReviewCreate, productTopRated, cart, userLogin, userRegister, userDetails, userUpdateProfile, userList, userDelete, userUpdate, orderCreate, orderDetails, orderPay, orderDeliver, orderListMy, orderList.
- **Persistence**: Cart and user and shipping are read at boot; screens also write to localStorage (e.g. shipping, payment method) for checkout continuity.

### 5.4 API Client and Auth

- **HTTP**: Axios. No baseURL; relative paths `/api/...`. In development, `proxy` in `frontend/package.json` points to `http://127.0.0.1:5000`.
- **Auth header**: Protected requests add `Authorization: Bearer ${userInfo.token}` from Redux state.
- **PayPal**: Client ID fetched from `GET /api/config/paypal` and used by react-paypal-button-v2.

### 5.5 Auth Flows (Frontend)

- **Login**: POST `/api/users/login` → store `userInfo` (including token) in Redux and localStorage → redirect from `redirect` query or `/`.
- **Register**: POST `/api/users` → same storage and redirect.
- **Logout**: Clear `userInfo`, `cartItems`, `shippingAddress`, `paymentMethod` from localStorage and Redux; `document.location.href = '/login'`.
- **Protected screens**: Each screen that requires login (or admin) checks `userInfo` (and `userInfo.isAdmin`) and redirects to `/login` (e.g. `?redirect=shipping` for checkout). No reusable `<PrivateRoute>` or `<AdminRoute>` wrapper in `App.js`.

### 5.6 Checkout Flow

1. **Cart** (`/cart`) — adjust quantities, remove items, proceed to shipping.
2. **Shipping** (`/shipping`) — collect address; save to Redux and localStorage; continue to payment.
3. **Payment** (`/payment`) — select method (e.g. PayPal); save to Redux/localStorage; continue to place order.
4. **Place Order** (`/placeorder`) — POST `/api/orders` with order items, shipping, payment, prices; redirect to order detail `/order/:id`.
5. **Order** (`/order/:id`) — show order; if PayPal, show PayPal button; on success call `PUT /api/orders/:id/pay` to mark paid.

### 5.7 Key Components

- **Header**: Nav, cart link, user dropdown (profile, orders, logout), admin dropdown (users, products, orders).
- **Footer**: Static footer.
- **Meta**: React Helmet for per-page title/description.
- **Loader, Message**: Loading and error/success messaging.
- **FormContainer**: Layout for forms.
- **Rating**: Star display for product rating.
- **Product, ProductCarousel**: Product card and top-rated carousel.
- **SearchBox, Paginate**: Search and pagination for product list.
- **CheckoutSteps**: Visual steps for shipping → payment → place order.

### 5.8 Build and Static Assets

- **Build**: `npm run build` in frontend produces `frontend/build` (CRA default). Root `heroku-postbuild` runs `npm install --prefix frontend` and `npm run build --prefix frontend`.
- **Production**: Backend serves `frontend/build` as static and falls back to `index.html` for non-API routes.

---

## 6. Security Aspects

- **Passwords**: Bcrypt (salt rounds 10), never returned in API (user select `-password`).
- **JWT**: Server-side verification with `JWT_SECRET`; 30-day expiry; no refresh flow or revocation.
- **Upload**: Allowed types restricted to jpg/jpeg/png; no auth on upload route (anyone can upload).
- **Input validation**: No schema validation library (e.g. Joi/Zod) at API boundaries; controllers assume valid body.
- **CORS**: Not explicitly configured in `server.js`; same-origin in production; dev uses proxy.
- **Secrets**: JWT and DB credentials from env; README contains sample PayPal credentials (should not be committed in production).

---

## 7. Deployment and Operations

### 7.1 Heroku (Primary)

- **Procfile**: `web: node backend/server.js`
- **Build**: `heroku-postbuild` installs frontend deps and builds frontend; backend serves the built SPA.
- **Single dyno**: One web process; no separate worker or scheduler.

### 7.2 Other

- **Docker**: No Dockerfile or docker-compose in the repo.
- **CI/CD**: No GitHub Actions or other CI config.
- **Env**: No `.env.example`; README lists required variables.

### 7.3 Scripts (Root package.json)

| Script             | Command / behavior                          |
|--------------------|---------------------------------------------|
| start              | node backend/server                         |
| server             | nodemon backend/server                      |
| client             | npm start --prefix frontend                 |
| dev                | concurrently server + client                |
| data:import        | node backend/seeder                         |
| data:destroy       | node backend/seeder -d                      |
| heroku-postbuild   | install frontend deps + build frontend      |

---

## 8. Testing

- **Backend**: No test framework or test files (no Jest/Mocha, no `*.test.js` / `*.spec.js`).
- **Frontend**: CRA provides Jest and React Testing Library; no test files present in `frontend/src`.
- **E2E**: None (no Cypress, Playwright, or similar).

---

## 9. Diagrams (Conceptual)

### 9.1 System Context

```
+------------------+     HTTPS      +------------------+
|   Browser        | <------------> |  Express Server  |
|   (React SPA)    |   /api/*, /    |  (Node)          |
+------------------+                +--------+---------+
                                            |
                                            | MONGO_URI
                                            v
                                    +------------------+
                                    |  MongoDB         |
                                    +------------------+
```

### 9.2 Auth Flow

```
Client                Backend
  |                      |
  |  POST /api/users/login
  |  { email, password }  |
  |--------------------->|
  |                      | User.findOne, matchPassword
  |                      | generateToken(id)
  |  200 { _id, name, email, isAdmin, token }
  |<---------------------|
  |  Store token in Redux + localStorage
  |  Subsequent requests: Authorization: Bearer <token>
  |--------------------->|
  |                      | protect: jwt.verify, User.findById
  |                      | req.user set
```

### 9.3 Checkout and Payment

```
Cart -> Shipping (address) -> Payment (method) -> Place Order
                                                      |
                                                      v
                                              POST /api/orders
                                                      |
                                                      v
                                              Redirect to /order/:id
                                                      |
                                    User completes PayPal in browser
                                                      |
                                                      v
                                              PUT /api/orders/:id/pay
                                              (mark isPaid, paidAt)
```

---

## 10. Possible Improvements (Reference)

- **API**: Add request validation (e.g. Joi/Zod) at route/controller boundaries; align Order schema with payload (e.g. itemsPrice); consider `/api/v1` prefix.
- **Auth**: Add refresh tokens or shorter-lived access tokens; protect upload route; optional rate limiting.
- **Config**: Central config module with validation; fail fast on missing env; add `.env.example`.
- **Frontend**: Extract `<PrivateRoute>` and `<AdminRoute>`; consider TypeScript and shared types with backend.
- **Testing**: Unit tests for business logic; integration tests for API routes; E2E for checkout.
- **DevOps**: Dockerfile and docker-compose for local and deployment; CI pipeline (lint, test, build).
- **Security**: Remove or redact sensitive data from README; add CORS config explicitly if needed; enforce upload size limits.

---

## 11. Document History

| Date       | Change                    |
|------------|---------------------------|
| 2025-02-26 | Initial architecture doc  |

---

*This architecture document reflects the ProShop codebase as of the date above. For run instructions and env setup, see the root [README.md](../README.md).*
