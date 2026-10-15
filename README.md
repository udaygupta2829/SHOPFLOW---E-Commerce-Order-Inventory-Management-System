# ShopFlow – E-Commerce Order & Inventory Management System

ShopFlow is an enterprise-grade full-stack e-commerce platform architected for high transaction integrity, resilient inventory consistency, and role-based access control. Built with **Spring Boot 3**, **Spring Security 6**, **JWT Authentication**, **JPA / Hibernate**, and **React**, it demonstrates production backend patterns including pessimistic concurrency locking to prevent overselling, transactional order rollbacks, and centralized exception handling.

---

## 1. Project Overview

In standard tutorial-grade shopping carts, order creation and inventory updates are decoupled, exposing systems to race conditions, inventory drift, and double allocations during concurrent traffic spikes. 

**ShopFlow** addresses these core distributed-systems challenges through:
- **Pessimistic Concurrency Locking (`@Lock(LockModeType.PESSIMISTIC_WRITE)`)**: Guarantees mutually exclusive stock decrements and prevents overselling.
- **Strict ACID Transactions (`@Transactional`)**: Ensures the 11-step checkout process (validation, stock reservation, line item recording, cart clearing, and order creation) executes atomically or rolls back cleanly with HTTP 409 Conflict.
- **Bi-Directional Inventory Auditing**: Order cancellations automatically revert inventory counts in an isolated transactional boundary, only permitted when orders are in `PLACED` or `CONFIRMED` states.
- **Role-Based Access Control (RBAC)**: Fine-grained security separating `CUSTOMER` workflows from `ADMIN` platform oversight.

---

## 2. Tech Stack

### Backend
- **Language**: Java 17+ (Compiled & verified on OpenJDK 25)
- **Framework**: Spring Boot 3.3.4
- **Security**: Spring Security 6, JJWT (io.jsonwebtoken 0.12.6), BCrypt Password Hashing
- **Persistence**: Spring Data JPA, Hibernate ORM
- **Database**: MySQL 8.0+ (with zero-configuration H2 MySQL-compatibility mode for instant local development)
- **Validation**: Jakarta Bean Validation (`@NotBlank`, `@DecimalMin`, `@Min`, `@Email`)
- **API Documentation**: Springdoc OpenAPI 3.0 / Swagger UI
- **Testing**: JUnit 5, Mockito, Spring Security Test, Spring Boot Test (MockMvc)
- **Build Tool**: Apache Maven 3.9+

### Frontend
- **Library**: React 19 / JavaScript (ES6+)
- **Build Tool**: Vite 8 / Rolldown
- **Routing**: React Router 7
- **HTTP Client**: Axios with automated request/response interceptors
- **Icons**: Lucide React
- **Design System**: Responsive CSS with CSS Custom Properties, flexbox/grid architectures, and mobile drawers

---

## 3. System Architecture

ShopFlow adheres to a strict multi-tier layered architecture ensuring clean separation of concerns, high testability, and adherence to Object-Oriented Design (OOP) principles.

```
                    ┌─────────────────────────┐
                    │    React Frontend UI    │
                    │ (Vite : Port 5173/Host) │
                    └────────────┬────────────┘
                                 │ HTTP / JSON (Axios + JWT)
                                 ▼
                    ┌─────────────────────────┐
                    │ Spring Security Filter  │
                    │   JwtAuthenticationFilter│
                    └────────────┬────────────┘
                                 │
     ┌───────────────────────────┴───────────────────────────┐
     ▼                                                       ▼
┌─────────────────────────┐                             ┌─────────────────────────┐
│   Public / Customer     │                             │   Admin Operations      │
│   REST Controllers      │                             │   REST Controllers      │
└────────────┬────────────┘                             └────────────┬────────────┘
             │                                                       │
             ▼                                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                             Service Layer                                       │
│        (AuthService, ProductService, CartService, OrderService, InventoryService)│
│                     • Business Logic & Validation                               │
│                     • ACID @Transactional Orchestration                         │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Repository Layer                                     │
│     (UserRepository, ProductRepository, CartRepository, OrderRepository, ...)  │
│                     • Spring Data JPA & Derived Queries                         │
│                     • Pessimistic Row-Level Write Locks                         │
└────────────────────────────────────────┬────────────────────────────────────────┘
                                         │
                                         ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Database Persistence Tier (MySQL / H2)                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Backend Package Layout
```
com.shopflow/
├── config/             # OpenAPI Swagger config, DataInitializer seed runner
├── controller/         # REST Controllers (Auth, Product, Cart, Order, Admin)
├── dto/                # Data Transfer Objects & Jakarta Bean Validation
├── entity/             # JPA Entities (User, Category, Product, Inventory, Cart, Order)
├── exception/          # Custom exceptions & Centralized @RestControllerAdvice
├── repository/         # Spring Data JPA repositories with query specifications
├── security/           # JWT Filter, UserDetailsService, SecurityFilterChain
└── service/            # Business logic and transaction managers
```

---

## 4. Database Schema

The relational schema implements foreign keys, constraints, cascade deletion policies, and indexes on frequently queried search vectors.

```
┌──────────────┐          ┌────────────────┐          ┌─────────────────┐
│    USERS     │ 1      1 │      CART      │ 1      * │   CART_ITEMS    │
├──────────────┤──────────├────────────────┤──────────├─────────────────┤
│ id (PK)      │          │ id (PK)        │          │ id (PK)         │
│ name         │          │ user_id (FK)   │          │ cart_id (FK)    │
│ email (UQ)   │          │ created_at     │          │ product_id (FK) │
│ password     │          │ updated_at     │          │ quantity        │
│ role         │          └────────────────┘          └────────┬────────┘
│ created_at   │                                               │
│ updated_at   │                                               │
└──────┬───────┘                                               │
       │                                                       │
       │ 1                                                     │ *
       │                                                       │
       │ *                                                     ▼
┌──────┴───────────────┐  1      * ┌─────────────────┐ *     1 ┌───────────────┐
│       ORDERS         │───────────│   ORDER_ITEMS   │─────────│   PRODUCTS    │
├──────────────────────┤           ├─────────────────┤         ├───────────────┤
│ id (PK)              │           │ id (PK)         │         │ id (PK)       │
│ user_id (FK)         │           │ order_id (FK)   │         │ name          │
│ total_amount         │           │ product_id (FK) │         │ description   │
│ status               │           │ quantity        │         │ price         │
│ shipping_address     │           │ price           │         │ category_id(FK)
│ created_at           │           └─────────────────┘         │ image_url     │
│ updated_at           │                                       │ created_at    │
└──────────────────────┘                                       └───────┬───────┘
                                                                       │ 1
                                                                       │
                                                                       │ 1
                                                              ┌────────┴────────┐
                                                              │    INVENTORY    │
                                                              ├─────────────────┤
                                                              │ id (PK)         │
                                                              │ product_id (FK) │
                                                              │ quantity        │
                                                              │ updated_at      │
                                                              └─────────────────┘
```

---

## 5. Authentication & Authorization

ShopFlow implements stateless JWT authentication powered by Spring Security:
1. **Password Hashing**: User credentials are encrypted using `BCryptPasswordEncoder` (strength 10). Passwords are never stored in plain text and are omitted from all DTO responses.
2. **Access Token Generation**: On successful authentication via `POST /api/auth/login` or `POST /api/auth/register`, a signed HMAC-SHA256 JWT containing user subject and role claims is issued.
3. **Role Enforcement**:
   - `ROLE_CUSTOMER`: Permitted to browse products, manage active cart, perform checkout, and track/cancel orders. Blocked with **HTTP 403 Forbidden** if attempting to access `/api/admin/**`.
   - `ROLE_ADMIN`: Access to dashboard analytics, product catalog mutations, category management, warehouse stock overrides, and platform orders.
4. **Error Responses**:
   - `401 Unauthorized`: Missing, expired, or malformed JWT token.
   - `403 Forbidden`: Authenticated user lacks privileged role.

---

## 6. How Order Processing Works

Order placement is the critical transaction in ShopFlow. The implementation in `OrderService.createOrder()` executes in an atomic `@Transactional(isolation = Isolation.REPEATABLE_READ)` block:

1. **Authentication**: Resolves current user from the `SecurityContext`.
2. **Cart Retrieval**: Loads the user's active persistent shopping cart.
3. **Cart Validation**: Checks that the cart is not empty; otherwise throws `BadRequestException`.
4. **Catalog Verification**: Validates that all items still exist in active inventory.
5. **Concurrency-Safe Stock Lock**: Acquires row-level write locks via `InventoryRepository.findByProductIdWithLock()`. If any requested item quantity exceeds available stock:
   - Throws `InsufficientInventoryException`.
   - Returns **HTTP 409 Conflict** with message `"Insufficient inventory for product: <Product Name>"`.
   - Rollback ensures zero items are deducted.
6. **Total Amount Calculation**: Derives total purchase value server-side (never trusting client totals).
7. **Order Creation**: Instantiates `Order` entity with status `PLACED`.
8. **Line Item Snapshots**: Creates `OrderItem` records persisting historical purchase prices so future catalog price changes do not distort historical receipts.
9. **Inventory Reduction**: Decrements available stock for each locked inventory row.
10. **Cart Cleanup**: Clears cart items in the database.
11. **Order Confirmation**: Returns populated `OrderResponse`.

---

## 7. Order Cancellation & Stock Restoration

Customers and administrators can cancel orders under strict business invariants:
- **Eligibility**: Cancellation is only permitted when order status is `PLACED` or `CONFIRMED`.
- **Invariants**: Orders in `SHIPPED`, `OUT_FOR_DELIVERY`, or `DELIVERED` status throw `BadRequestException` ("Cannot cancel order in status: SHIPPED").
- **Transactional Restoration**: Upon valid cancellation:
  - Order status transitions to `CANCELLED`.
  - For each `OrderItem`, row locks are acquired on the corresponding `Inventory` record.
  - The exact ordered quantity is restored back to active inventory.
  - Changes are committed atomically.

---

## 8. API Documentation

Swagger OpenAPI 3 is integrated at:
- **Swagger UI**: `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON Spec**: `http://localhost:8080/api-docs`

### Core Endpoints Matrix

| Module | Method | Endpoint | Access | Description |
|---|---|---|---|---|
| **Auth** | `POST` | `/api/auth/register` | Public | Register user & receive JWT |
| **Auth** | `POST` | `/api/auth/login` | Public | Authenticate & receive JWT |
| **Auth** | `GET` | `/api/profile` | Authenticated | Retrieve authenticated user profile |
| **Catalog** | `GET` | `/api/products` | Public | Paginated search, filter & sort |
| **Catalog** | `GET` | `/api/products/{id}` | Public | Product details & live stock |
| **Catalog** | `GET` | `/api/categories` | Public | List categories |
| **Cart** | `GET` | `/api/cart` | Customer/Admin | View active shopping cart |
| **Cart** | `POST` | `/api/cart/items` | Customer/Admin | Add product with quantity to cart |
| **Cart** | `PUT` | `/api/cart/items/{id}`| Customer/Admin | Update cart item quantity |
| **Cart** | `DELETE`| `/api/cart/items/{id}`| Customer/Admin | Remove item from cart |
| **Orders** | `POST` | `/api/orders` | Customer/Admin | Atomic 11-step checkout |
| **Orders** | `GET` | `/api/orders` | Customer/Admin | Paginated user order history |
| **Orders** | `GET` | `/api/orders/{id}` | Customer/Admin | View order details & tracking |
| **Orders** | `PUT` | `/api/orders/{id}/cancel` | Customer/Admin | Transactional order cancel & restock |
| **Admin** | `GET` | `/api/admin/dashboard`| Admin Only | Platform revenue & metrics |
| **Admin** | `POST` | `/api/admin/products` | Admin Only | Create catalog product |
| **Admin** | `PUT` | `/api/admin/products/{id}`| Admin Only | Update catalog product |
| **Admin** | `DELETE`| `/api/admin/products/{id}`| Admin Only | Remove product |
| **Admin** | `GET` | `/api/admin/inventory`| Admin Only | Audit warehouse stocks |
| **Admin** | `PUT` | `/api/admin/inventory/{productId}`| Admin Only | Manually update stock count |
| **Admin** | `GET` | `/api/admin/orders` | Admin Only | View all platform orders |
| **Admin** | `PUT` | `/api/admin/orders/{id}/status` | Admin Only | Update order fulfillment status |
| **Admin** | `GET` | `/api/admin/users` | Admin Only | Directory of registered users |

---

## 9. Testing & Quality Verification

Comprehensive unit and integration test coverage is implemented with **JUnit 5** and **Mockito**:
- `AuthServiceTest`:
  - `testRegisterSuccess()`: Validates token issuance and cart creation.
  - `testRegisterDuplicateEmail_ThrowsBadRequestException()`: Verifies rejection on duplicate email.
  - `testLoginValid()`: Verifies credential validation.
  - `testLoginInvalid_ThrowsUnauthorizedException()`: Verifies 401 handling on bad credentials.
- `ProductServiceTest`:
  - `testCreateProduct_Success()`: Tests category association and initial stock linkage.
  - `testGetProductById_Success()`: Verifies response formatting.
  - `testGetProductById_NotFound_ThrowsResourceNotFoundException()`: Tests 404 handling.
- `CartServiceTest`:
  - `testAddToCart_Success()`: Tests item addition within stock limit.
  - `testAddToCart_ExceedsInventory_ThrowsInsufficientInventoryException()`: Verifies bounds checking.
  - `testUpdateCartItemQuantity_Success()`: Tests quantity stepper logic.
  - `testRemoveCartItem_Success()`: Tests item deletion.
- `OrderServiceTest`:
  - `testCreateOrder_Success()`: Verifies atomic checkout, stock deduction, and cart clearing.
  - `testCreateOrder_InsufficientInventory_ThrowsInsufficientInventoryException()`: Verifies conflict rollback.
  - `testCancelOrder_Success_RestoresInventory()`: Verifies automated stock restoration.
  - `testCancelOrder_InvalidStatus_ThrowsBadRequestException()`: Rejects cancellation if already shipped.
- `SecurityAuthorizationTest`:
  - `testCustomerCannotAccessAdminApi_ReturnsForbidden()`: Verifies HTTP 403.
  - `testAdminCanAccessAdminApi_ReturnsOk()`: Verifies privileged endpoint access.
  - `testUnauthenticatedUserCannotAccessAdminApi_ReturnsUnauthorized()`: Verifies HTTP 401.

### Running Backend Tests
```bash
cd backend
mvn test
```
All 18 tests pass with 0 failures and 0 errors.

---

## 10. How to Run Locally

### Prerequisites
- Java JDK 17 or higher
- Node.js 20+ and npm
- Maven 3.8+ (or Maven wrapper)
- (Optional) MySQL 8.0+ if running with production database profile

### Step 1: Start Backend
```bash
cd backend

# Option A: Zero-setup dev mode (in-memory H2 in MySQL mode + auto seed data)
mvn spring-boot:run

# Option B: Run with external MySQL server
mvn spring-boot:run -Dspring-boot.run.profiles=mysql
```
Backend will start on `http://localhost:8080`.

### Step 2: Start Frontend
```bash
cd frontend
npm install
npm run dev
```
Frontend will start on `http://localhost:5173`.

---

## 11. Demo Credentials

The backend seeds demo accounts on initial startup:

| Role | Email | Password | Permissions |
|---|---|---|---|
| **Admin** | `admin@shopflow.com` | `Admin@123` | Full Administrative & Inventory Controls |
| **Customer** | `customer@shopflow.com` | `Customer@123` | Browsing, Cart, Checkout, Order Tracking |

---

## 12. Environment Variables

| Variable | Default Value | Description |
|---|---|---|
| `DB_HOST` | `localhost` | MySQL Database Host |
| `DB_PORT` | `3306` | MySQL Port |
| `DB_NAME` | `shopflow_db` | MySQL Database Name |
| `DB_USER` | `root` | MySQL Username |
| `DB_PASSWORD` | `root` | MySQL Password |
| `JWT_SECRET` | `404E6352...` | 256-bit Secret Key for signing JWTs |
| `VITE_API_BASE_URL` | `http://localhost:8080/api` | Frontend API Gateway URL |

---

## 13. Future Improvements

1. **Distributed Caching with Redis**: Cache product catalog queries and category taxonomies with TTL invalidation upon admin mutations.
2. **Asynchronous Order Processing (Kafka / RabbitMQ)**: Decouple checkout confirmation from notification services and invoice generation.
3. **Stripe Payment Gateway Integration**: Webhook verification for two-phase commit payment authorization and capture.
4. **Elasticsearch Product Search**: Fuzzy matching, autocomplete suggestions, and faceted filtering for large-scale catalogs.

---

## 14. Author

Engineered as a showcase portfolio project for **Amazon SDE-I / Software Development Engineer** technical evaluation, demonstrating end-to-end full-stack software development, ACID transactions, concurrency locking, and clean architecture.
