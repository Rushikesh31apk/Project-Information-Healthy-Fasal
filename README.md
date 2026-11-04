# Healthy Fasal – Fresh Fruits & Vegetables Delivery Platform

**Handover document.** It explains what the system does, how it is built, how a request travels through the code, every business rule, how to run it, how to change it, and what to check before going live. A new developer should be able to read this top to bottom and be productive in about an hour.

---
# Link: [https://healthy-fasal.wuaze.com/](https://healthy-fasal.wuaze.com/)

## Contents

1. [What this project is](#1-what-this-project-is)
2. [Technology and requirements](#2-technology-and-requirements)
3. [Run it on your computer (Windows XAMPP or any PHP setup)](#3-run-it-on-your-computer)
4. [Test accounts and sample data](#4-test-accounts-and-sample-data)
5. [The three roles and what each can do](#5-the-three-roles-and-what-each-can-do)
6. [Folder structure – what every file is for](#6-folder-structure)
7. [How a request works (architecture)](#7-how-a-request-works)
8. [Business rules in detail](#8-business-rules-in-detail)
9. [Status flows (order, payment, delivery)](#9-status-flows)
10. [Database reference](#10-database-reference)
11. [Configuration reference (`.env` and Admin → Settings)](#11-configuration-reference)
12. [Email, OTP and password reset](#12-email-otp-and-password-reset)
13. [Security model](#13-security-model)
14. [Performance, caching and scaling](#14-performance-caching-and-scaling)
15. [Automated tests](#15-automated-tests)
16. [Everyday maintenance – "how do I change X?"](#16-everyday-maintenance)
17. [Deploying to a live server and go-live checklist](#17-deploying-and-go-live-checklist)
18. [Troubleshooting](#18-troubleshooting)
19. [Known limitations and ideas for the future](#19-known-limitations-and-future-ideas)
20. [Glossary](#20-glossary)
---

## 1. What this project is

Healthy Fasal is a web shop for a local fruit and vegetable business. Customers who live in **housing societies** order from their phone. The shop prepares the order, a **delivery partner** brings it, and the customer **pays at the doorstep** (cash or UPI). There is no online payment gateway.

It contains three separate interfaces in one application:

| Interface | Who uses it | Address |
|---|---|---|
| **Storefront + customer area** | Customers | `/` , `/shop`, `/cart`, `/checkout`, `/orders`, `/account` |
| **Admin panel** | Shop owner / staff | `/admin` |
| **Delivery partner app** (mobile-first) | Delivery boys | `/partner` |

Main features:

* Catalog of products and categories with prices, discounts, stock and images, all managed from the admin panel (no code changes needed to add products).
* **Multi-society service**: each society has its own delivery fee, minimum order, free-delivery limit, order cut-off time, delivery slots and delivery partners.
* Two ordering modes: **Order Fresh Now** (same day) and **Preorder for Tomorrow** (next eligible delivery day).
* Cart, coupons and automatic offers, server-side price checking, duplicate-order protection.
* Stock reservation so two customers cannot buy the last item.
* Delivery partner workflow with **cash/UPI collection records** and admin **reconciliation**.
* Dashboard, charts, reports and CSV exports built from real order data.
* Email OTP at signup and for password reset (via SMTP).
* Security basics: hashed passwords, CSRF protection, role checks on the server, login throttling, safe image uploads, audit log.

Size of the codebase: about 1,300 lines of controllers, 1,000 lines of services, 1,600 lines of views, 30 database tables, 111 routes, and 276 automated test checks.

---

## 2. Technology and requirements

| Layer | Technology |
|---|---|
| Backend | **PHP 8.2+** (no framework, no Composer packages), own small MVC |
| Database | **MySQL 8+** or MariaDB 10.5+, accessed with **PDO prepared statements** |
| Frontend | HTML5, CSS3, **Bootstrap 5.3**, vanilla JavaScript (`fetch`), Bootstrap Icons, **Chart.js** (admin charts) |
| Fonts / libraries | Loaded from public CDNs (jsDelivr, Google Fonts). Visitors need internet access. |
| Sessions | PHP sessions (file by default, optional database sessions) |
| Mail | Built-in SMTP client (PHP sockets), works with Gmail, Brevo, SendGrid, Zoho, hosting mailboxes, etc. |

Required PHP extensions: `pdo_mysql`, `mbstring`, `fileinfo`, `gd` (WebP support recommended), `openssl` (needed for SMTP over TLS), `curl` is not required.

> **Windows/XAMPP:** extensions are switched on in `C:\xampp\php\php.ini` by removing the `;` in front of the line (for example `extension=gd`, `extension=openssl`, `extension=fileinfo`), then restarting Apache. `public/check-gd.php` is a small helper page that tells you whether GD is active; **delete it before going live.**

---

## 3. Run it on your computer

### 3.1 First-time installation

1. Put the project in `C:\xampp\htdocs\healthy-fasal` (or any folder).
2. Start **Apache** and **MySQL** in the XAMPP Control Panel.
3. Create the database and load everything with **one command** (from the project folder):

   ```powershell
   C:\xampp\php\php.exe bin\setup.php --db=healthy_fasal --user=root --pass= --env=local --seed --demo-orders=150
   ```
   * `--user=root --pass=` is the XAMPP default (empty password). Use your own values if you changed them.
   * `--seed` adds sample societies, products, offers and **test accounts** (never use on a live site).
   * `--demo-orders=150` adds 150 sample orders so the dashboard has data.
   * The command also creates the `.env` file and generates `APP_KEY`.
4. Open **http://localhost/healthy-fasal/public/** (or run the built-in server below).

Alternative without the command line: create an empty database in phpMyAdmin, import `database/schema.sql`, then `database/seed.sql`, copy `.env.example` to `.env` and edit the `DB_*` values.

### 3.2 Built-in PHP server (optional, simplest)

```powershell
C:\xampp\php\php.exe -S localhost:8000 -t public public\router.php
```
Open http://localhost:8000. (`public/router.php` is a development-only helper so static files and the front controller both work.)

### 3.3 Where the web root should point

The only folder that should be reachable from the internet is **`public/`**. On a live server, point the domain's document root at `public/`. If a host cannot do that, the `.htaccess` in the project root redirects everything into `public/` and blocks `app/`, `config/`, `database/`, `storage/`, `views/` and so on.

### 3.4 Command-line tools (`bin/`)

| Command | What it does |
|---|---|
| `php bin/setup.php --db=… --user=… --pass=… [--env=local\|production] [--seed] [--demo-orders=N] [--force]` | Creates the database tables, writes `.env`, generates `APP_KEY`. `--force` wipes and reinstalls. Refuses `--seed` in production mode. |
| `php bin/create-admin.php "Owner Name" 9876543210 [email]` | Creates an admin account (asks for the password; Linux/Mac terminals hide it). |
| `php bin/migrate.php` | Applies new files from `database/migrations/` once each (use after updating the code). |
| `php bin/test-mail.php you@example.com` | Sends a test email and prints the mail server's exact answer. |

---

## 4. Test accounts and sample data

Created only when you install with `--seed`. **Password for all: `Password@123`.** Delete them before launch.

| Role | Mobile (login) | Notes |
|---|---|---|
| Admin | `9000000001` | `/admin/login` |
| Delivery partner | `9000000002` | serves Green Valley + Sunrise |
| Delivery partner | `9000000003` | serves Sunrise + Maple |
| Customer | `9000000010` | Green Valley Residency |
| Customer | `9000000011` | Sunrise Heights |

Sample societies: Green Valley Residency, Sunrise Heights, Maple Gardens (active) and Lotus Enclave (inactive). 19 products across Vegetables, Fruits, Leafy Vegetables, Seasonal Produce (plus an empty hidden Daily Essentials category). Coupons: `WELCOME10` (10 %, min ₹200, max ₹100, once per customer), `FRESH50` (₹50 off ₹500+), and an automatic 5 % off fruits. Delivery slots: Morning 7–10, Late Morning 10–1, Evening 5–8.

---

## 5. The three roles and what each can do

### Customer
Register (society + flat/floor/building/landmark/PIN), log in, browse and search, add to cart, apply a coupon, choose **Fresh Now** or **Preorder**, pick an address and delivery slot, place the order, track it, cancel while allowed, view invoices, manage several addresses and a default address. Customers can only see their own data.

### Admin
Dashboard (KPIs and charts), orders (filters, status changes, assign partner, print packing slip, payment dispute/refund), **preorders** (grouped by date, demand/purchase list, confirm, CSV), products (add/edit/stock/images), categories, offers and coupons, customers, delivery partners, societies, delivery slots, holidays, settings, testimonials, reports with CSV export, email test.

### Delivery partner
Sees only orders assigned to them. For each order: full address, tap-to-call, Google Maps link, items, amount to collect. Actions: accept, start preparing, picked up, out for delivery, **record payment** (cash/UPI, must tick "I verified"), mark delivered, report a problem (customer unavailable, address issue, item shortage, failed delivery), add a note.

Roles are **enforced on the server** in `Router::guard()`, not only by hiding buttons (see §13).

---

## 6. Folder structure

```
healthy-fasal/
├─ .env                      Your private settings (NOT in Git, never share). Created by bin/setup.php.
├─ .env.example              Template listing every setting.
├─ .htaccess                 For hosts whose web root is the project folder: forwards to /public, blocks private folders.
├─ README.md                 This document.
│
├─ public/                   ← the ONLY folder the web server should expose
│  ├─ index.php              Front controller: every page request enters here.
│  ├─ router.php             Dev-only helper for `php -S`.
│  ├─ .htaccess              URL rewriting, gzip, 1-year caching for static files.
│  ├─ check-gd.php           Diagnostic page (DELETE before going live).
│  ├─ assets/
│  │  ├─ css/app.css         Storefront styles (design tokens are CSS variables at the top).
│  │  ├─ css/admin.css       Admin + partner styles.
│  │  ├─ js/app.js           Storefront JS: cart, quick view, filters, checkout slots, tracking poll.
│  │  ├─ js/admin-dashboard.js  Chart.js dashboard charts.
│  │  └─ images/             logo.png, hero.png, products/*.webp, categories/*.webp, banners/, placeholder.svg
│  └─ uploads/               Images uploaded from the admin panel (PHP execution disabled by .htaccess).
│
├─ app/
│  ├─ bootstrap.php          Loads config, timezone, autoloader, error settings.
│  ├─ controllers/           One class per area; thin: read input → call a service → render a view.
│  │   HomeController, ShopController, CartController, AuthController, AccountController,
│  │   CheckoutController, CustomerOrderController, PartnerController,
│  │   AdminController (dashboard), AdminCatalogController (products/categories),
│  │   AdminOrderController (orders/preorders), AdminPeopleController (partners/customers),
│  │   AdminBusinessController (offers/societies/slots/settings/holidays/testimonials/mail test),
│  │   AdminReportController (reports/CSV)
│  ├─ services/              ALL business logic lives here (this is the part to read first).
│  │   CartService      cart for guests (session) and customers (database)
│  │   PricingService   subtotal, offers, coupons, delivery fee, minimum order (integer paise)
│  │   ScheduleService  delivery dates, holidays, preorder cut-off, slot availability/capacity
│  │   InventoryService reserve / release / sell stock + stock ledger
│  │   OrderService     place order, status changes, assign partner, deliver, availability workflow
│  │   PaymentService   partner collections, admin dispute/refund
│  │   ReportService    KPIs, chart data, date-range reports
│  │   OtpService       6-digit email codes (issue/verify/limits)
│  │   UploadService    image validation + re-encoding
│  ├─ middleware/Auth.php   Login, logout, role lookup, login throttling
│  ├─ models/Catalog.php    Storefront read queries (products, societies, offers)
│  └─ helpers/              Router, Db (PDO wrapper), Session, Settings, Cache, Audit, Mailer (SMTP),
│                           View, Controller (base class), UserError, functions.php (global helpers)
│
├─ routes/web.php            Every URL, which controller method handles it, and which role may use it.
├─ config/config.php         Reads .env (real environment variables override it).
├─ views/                    PHP templates
│  ├─ layouts/               main (storefront), admin, partner, auth (staff login)
│  ├─ partials/              product-card, cart-body, flash messages, pager, footer, login-form-style
│  ├─ home/ shop/ cart/ auth/ customer/ admin/ delivery/ errors/
│
├─ database/
│  ├─ schema.sql             Full current schema (new installs). Every table has a comment.
│  ├─ seed.sql               Sample data + test accounts (LOCAL ONLY).
│  └─ migrations/            Incremental upgrade scripts run by bin/migrate.php.
├─ bin/                      Command-line tools (see §3.4).
├─ tools/generate_images.py  Python script that renders the product illustrations (see §16).
├─ tests/                    Automated tests (see §15).
└─ storage/
   ├─ logs/                  php-error.log (and mail.log in local mode)
   └─ cache/                 Cache files (safe to delete anytime)
```

Every private folder also contains an `.htaccess` with `Require all denied` as a second line of defence.

---

## 7. How a request works

```
Browser ──► public/index.php
              │  1. loads app/bootstrap.php (config, timezone, autoloader, helpers)
              │  2. sets security headers and the global error handler
              │  3. Session::start()  (secure cookie settings)
              │  4. routes/web.php registers every route into the Router
              ▼
          Router::dispatch(method, uri)
              │  finds the matching route
              │  guard():  POST ⇒ CSRF token must match (else 419)
              │            route has roles ⇒ must be logged in (else redirect to login / 401 JSON)
              │                               role must match (else 403)
              │            must_change_password ⇒ forced to /change-password
              ▼
          Controller method        (reads input, validates)
              │
              ▼
          Service                  (business rules, DB transactions, audit log)
              │
              ▼
          View + Layout            (PHP template → HTML)    or    JSON for /api/* calls
```

Key conventions:

* **Controllers stay thin.** If you are writing an `if` about money, stock, dates or statuses inside a controller, it belongs in a service.
* **`UserError`** is an exception whose message is safe to show to the person. Services throw it ("Only 3 left"); controllers catch it and show it as a flash message or JSON error.
* **Everything that changes several tables runs in `Db::tx()`** (a transaction with automatic retry on deadlock).
* **Money is calculated in integer paise** (`paise()` / `rupees()` helpers), stored as `DECIMAL(10,2)`, and shown with Indian grouping by `inr()` (₹12,34,567).
* **Output is escaped** with `e()` in every view.
* **AJAX endpoints** live under `/api/...` and return JSON; they send the CSRF token in the `X-CSRF-Token` header.
* **Read-only polling routes** are marked `ro` in the route file so PHP releases the session lock early.
* The MySQL session time zone is set to the business time zone (`TIMEZONE`, default `Asia/Kolkata`) so `NOW()` in SQL and `date()` in PHP agree.

---

## 8. Business rules in detail

### 8.1 Browsing and the cart

* A product shows only if the product **and** its category are active.
* Price shown = `products.price`. If `products.mrp` is higher, a "% OFF" badge and the struck-through original price appear.
* **Guests** keep the cart in the session; after login it is merged into the database cart (`CartService::mergeGuest`).
* "Add to cart" re-checks the product on the server. In *Fresh Now* mode it also refuses quantities above the sellable stock. In *Preorder* mode out-of-stock items **can** be added (you buy fresh for tomorrow).
* Maximum quantity per product: setting `cart_max_qty` (default 20).
* Cart prices are never trusted from the browser. The server re-reads prices at every step.

### 8.2 Offers, coupons and totals (`PricingService`)

1. `subtotal` = sum of price × quantity (paise).
2. **Discount**: if the customer entered a coupon, only that coupon's offer is considered; otherwise the **best automatic offer** (an offer with no coupon code) is applied. Only one discount applies per order.
3. An offer can be a percentage (with optional maximum amount) or a fixed amount; it can apply to the whole cart, selected categories or selected products; it can have a minimum order value, start/end date, a society restriction, a total usage limit and a per-customer limit.
4. **Delivery fee** comes from the society. It becomes free when `subtotal − discount ≥ free_delivery_above` (if that is above 0).
5. **Minimum order** (society `min_order`) is checked against `subtotal − discount`.
6. `total = subtotal − discount + delivery fee`.

Coupon errors are specific: unknown, expired, not started, wrong society, usage limit reached, already used by this customer, minimum not reached, not applicable to the cart.

### 8.3 Fresh Now vs Preorder (`ScheduleService`)

| | Order Fresh Now | Preorder for Tomorrow |
|---|---|---|
| Delivery date | Today | Next eligible dates (server calculated) |
| Stock | Must be available; **reserved** at order time | Not reserved by default (setting `reserve_stock_for_preorders`); admin confirms availability |
| Slot | Must still be open today | Any open slot on the chosen date |
| Status at start | `pending` ("Order Placed") | `pending` shown as "Pending Confirmation" |

**Preorder date calculation** (`ScheduleService::preorderDates`):

1. Take the society's cut-off time (`societies.preorder_cutoff`, e.g. 20:00).
2. Ordering **before** the cut-off today ⇒ earliest delivery is **tomorrow**. Ordering **after** it ⇒ the **day after tomorrow**.
3. Skip **non-delivery weekdays** (Admin → Settings) and **holidays** (Admin → Settings → Holidays).
4. Offer the next *N* eligible dates (`preorder_days_offered`, default 3). Anything else is rejected on the server.

**Slots**: a society that has its own slots uses only those; otherwise the shared default slots (those with no society) apply. For today, a slot closes `slot_lead_minutes` (default 30) before it starts. A slot can have a daily capacity (`max_orders`, 0 = unlimited) enforced under a database row lock.

**Cancelling by the customer**: a pending order can always be cancelled (Fresh Now). A confirmed Fresh Now order can be cancelled for `cancel_window_minutes` after placing it. A preorder can be cancelled until the cut-off time on the day before delivery. Otherwise the customer must call the shop.

### 8.4 Placing an order (`OrderService::place`)

Runs in **one transaction**:

1. Idempotency: the checkout page includes a one-time token (`idem`). If an order with the same `(user, token)` already exists (double tap, refresh, back button), that order is returned and nothing is created twice. A unique database key backs this up even for simultaneous requests.
2. Per-customer limit: at most `orders_per_hour_limit` (default 10) orders per hour.
3. The chosen address must belong to the customer, and its society must be active.
4. Product rows are **locked** (`SELECT … FOR UPDATE`, in id order so deadlocks cannot happen) and checked: active, category active, quantity limits, stock (for reserving orders).
5. Delivery date and slot are validated (including capacity).
6. Payment method must be enabled.
7. The quote is recomputed from the locked database rows. A coupon that became invalid blocks the order; if the total differs from what the customer saw, they are asked to confirm again.
8. Rows are written: `orders` (with a full **address snapshot** and unique order number `HF` + yymmdd + 5 characters), `order_items` (name, unit, price **snapshots**), stock reservations, offer redemption, status history. The cart is emptied.

Because the address and prices are copied into the order, later edits to a customer's profile or a product's price never change old orders.

### 8.5 Stock model (`InventoryService`)

* `sellable = stock_qty − reserved_qty`.
* **Place order** ⇒ `reserved_qty += qty` using an atomic conditional `UPDATE … WHERE stock_qty − reserved_qty ≥ qty`.
* **Cancel / mark unavailable** ⇒ reservation released.
* **Delivered** ⇒ `stock_qty` and `reserved_qty` both reduced (the goods left the shop).
* **Admin stock change** (purchase or adjustment) ⇒ `stock_qty` changes, never below zero.
* Every change writes a row to `inventory_movements` (type, amounts, order, user, note), so stock can always be explained.

### 8.6 Preorder preparation and unavailable items

Admin → **Preorders** shows, for a delivery date and optional society:

* **Purchase list**: for each product the quantity needed by all preorders, the stock on hand and **how much to buy**.
* Orders grouped by society and slot, with bulk **Confirm**.
* **CSV export** of the purchase list and order list.

If an item cannot be supplied, the admin marks that line **unavailable** (the customer is never charged for it and nothing is substituted silently). The admin phones the customer, then either removes the unavailable lines (totals, discount and delivery fee are recalculated and a history note is written) or cancels the order.

### 8.7 Assignment and delivery

* The admin assigns a partner to a **confirmed** (or later) order. The partner must serve that society (`partner_societies`). The assignment is stored in `delivery_assignments` (history of reassignments is kept).
* The partner can only see and act on orders where `orders.partner_id` is their own id; others return 404.
* Partner steps: Accept → Start preparing → **Picked up** (status becomes `assigned`) → **Out for delivery** → record payment → **Mark delivered**.
* Problems: customer unavailable / address issue / failed delivery ⇒ status `failed`. Item shortage ⇒ status `unavailable`. The admin then reassigns, re-confirms or cancels.

### 8.8 Payments (`PaymentService`)

Payment status is **separate** from order status. Statuses: `pending`, `partial`, `paid`, `disputed`, `refunded`.

* Methods are switched on in Admin → Settings: Cash on Delivery, UPI (paid to the business), Other offline.
* The partner records a payment only when the order is **Out for delivery**.
* The amount must equal the amount still due (partial payments only if enabled).
* The partner must tick **"I verified"**: cash counted, or the UPI credit seen in the business UPI app. A customer-typed UPI id is never treated as proof; the optional reference is just a note.
* Each collection is a row in `payment_collections` with partner, method, amount, time. **There is no edit or delete code** for these rows.
* Delivery is blocked until the order is `paid`, unless the admin enabled "Allow delivery without payment" **and** the partner writes a reason (stored on the order).
* The admin can only add a **dispute / resolve / refund** with a reason. Each change is written to `payment_status_history` and `audit_logs`.

### 8.9 Reports and dashboard (`ReportService`)

Everything is computed from real rows, and **cancelled orders are always excluded** from sales figures.

| Figure | Meaning |
|---|---|
| Gross order value | Sum of `subtotal + delivery_fee` of non-cancelled orders |
| Discounts | Sum of `discount` |
| Refunds | Sum of `refund_amount` |
| **Net sales** | Gross − discounts − refunds |
| Collected | Sum of `amount_collected` (from verified collections) |
| Delivered, unpaid | Delivered orders whose payment is still pending/partial/disputed |
| Not yet delivered | Value of open orders |

Reports use the **order date** (`placed_at`). Dashboard cards: Today's Orders, Pending Orders (pending → assigned, unavailable), Active Preorders, Delivered Today, Today's Sales (booked, not cash), Outstanding Collections, Active Customers, Active Delivery Partners. Charts: daily and monthly sales, revenue by society, bestsellers, category performance, cash vs UPI, today's deliveries, preorder demand.

---

## 9. Status flows

### 9.1 Order status

```
 pending ──► confirmed ──► preparing ──► ready ──► assigned ──► out_for_delivery ──► delivered
    │            │             │          │          │                │
    │            └─────────────┴──────────┴──────────┴────────────────┴──► cancelled
    └──► unavailable ◄─────────(item shortage)                        └──► failed ──► ready (admin reopens) / cancelled
```

Who may make which change is defined in `OrderService::TRANSITIONS`:

| From | Admin may go to | Partner may go to | Customer may go to |
|---|---|---|---|
| pending | confirmed, unavailable, cancelled | – | cancelled (policy applies) |
| confirmed | preparing, ready, unavailable, cancelled | preparing, assigned, unavailable | cancelled (policy applies) |
| preparing | ready, unavailable, cancelled | assigned, unavailable | – |
| ready | unavailable, cancelled | assigned, out_for_delivery, unavailable | – |
| assigned | ready, unavailable, cancelled | out_for_delivery, failed, unavailable | – |
| out_for_delivery | failed, cancelled (and *delivered* via override) | failed, unavailable (and *delivered* via the deliver step) | – |
| unavailable | confirmed, cancelled | – | – |
| failed | ready, cancelled | – | – |
| delivered / cancelled | final | final | final |

`delivered` is **never** set by a plain status change: it goes through `OrderService::deliver()` which checks payment, deducts stock and closes the assignment.

Customer-facing timeline (7 steps): Order Placed → Order Confirmed → Preparing Fresh Produce → Ready for Delivery → Delivery Partner Assigned → Out for Delivery → Delivered. The page refreshes itself every 20 seconds. There is **no live GPS tracking**.

### 9.2 Payment status
`pending` → `partial` → `paid`; admin may set `disputed` (then resolve back) or `refunded`.

### 9.3 Assignment status
`assigned` → `accepted` → `completed`; or `rejected`, `failed`, `reassigned`.

Every status change is stored with who did it, when and why in `order_status_history`; partner events (picked up, issue, note) in `delivery_status_history`.

---

## 10. Database reference

30 tables (a 31st, `schema_migrations`, is created by `bin/migrate.php`). `database/schema.sql` contains a comment for every table and important column.

**People**

| Table | Purpose |
|---|---|
| `users` | All accounts. `role` = admin / customer / partner. Unique `mobile` and `email`. `password_hash`, `is_active`, `must_change_password`, `email_verified_at`. |
| `customer_addresses` | Saved addresses with society, flat, floor, building, landmark, PIN, instructions, `is_default`, soft-delete flag. |
| `delivery_partner_profiles` | Extra partner details (vehicle, joining date, notes). |
| `partner_societies` | Which societies each partner serves. |

**Catalog and stock**

| Table | Purpose |
|---|---|
| `categories` | Storefront categories (`sort_order`, `is_active`, image). |
| `products` | Name, SKU, unit, `price`, `mrp`, `cost_price`, `stock_qty`, `reserved_qty`, `low_stock_threshold`, flags (featured / bestseller / seasonal / active), image. |
| `inventory_movements` | Append-only stock ledger. |

**Cart and offers**

| Table | Purpose |
|---|---|
| `carts`, `cart_items` | Persistent cart for logged-in customers (primary key prevents duplicate lines). |
| `offers` | Discount rules (type, value, cap, minimum, dates, society, limits). |
| `offer_products`, `offer_categories` | Which products/categories an offer applies to. |
| `coupons` | A code that unlocks an offer. Offers without a coupon apply automatically. |
| `offer_redemptions` | Who used which offer on which order. |

**Service area**

| Table | Purpose |
|---|---|
| `societies` | Name, address, PIN, delivery instructions, `min_order`, `delivery_fee`, `free_delivery_above`, `preorder_cutoff`, `is_active`. |
| `delivery_slots` | Slots (global when `society_id` is NULL), time range, capacity. |
| `holidays` | No-delivery dates. |

**Orders and money**

| Table | Purpose |
|---|---|
| `orders` | Header: unique `order_number`, type, delivery date/slot, `status`, `payment_status`, money fields, `partner_id`, **address snapshot (`ship_*`)**, `idempotency_key`. |
| `order_items` | Line snapshots: name, unit, qty, `unit_price`, `line_total`, `reserved_qty`, `availability` (ok/unavailable). |
| `order_status_history` | Every status change (who, role, when, note). |
| `delivery_assignments` | Assignment history for each order. |
| `delivery_status_history` | Partner-side events. |
| `payment_collections` | **Immutable** doorstep collections. |
| `payment_status_history` | Audit trail of payment status changes. |

**System**

| Table | Purpose |
|---|---|
| `app_settings` | Business rules edited in Admin → Settings (key/value). |
| `audit_logs` | Admin/partner/payment actions with IP address. |
| `login_attempts` | Throttling for logins and reset codes. |
| `email_otps` | Hashed 6-digit email codes (purpose `register` or `reset`). |
| `sessions` | Only used when `SESSION_DRIVER=db`. |
| `testimonials` | Real customer feedback entered by the admin (the homepage section hides when empty). |

**Main relationships**

```
users ─┬─< customer_addresses >── societies ──< delivery_slots
       ├─< carts ─< cart_items >── products >── categories
       ├─< orders ─┬─< order_items >── products
       │           ├─< order_status_history
       │           ├─< delivery_assignments >── users(partner)
       │           ├─< delivery_status_history
       │           ├─< payment_collections
       │           └─< payment_status_history
       └─(partner)─< partner_societies >── societies
offers ─┬─< coupons      ├─< offer_products      └─< offer_categories
```

Design points: foreign keys everywhere, `DECIMAL` for money, indexes for the common filters (status + date, society + date, partner + status, user + placed_at), unique keys for order numbers, idempotency, SKU, mobile and email.

---

## 11. Configuration reference

### 11.1 `.env` (server settings, secrets)

Real environment variables override `.env`. **Never commit `.env` or send it to anyone.** After editing it you do not need to restart Apache.

| Key | Meaning |
|---|---|
| `APP_ENV` | `local` or `production`. Production hides errors, marks cookies `Secure`, and refuses the `log` mail driver. |
| `APP_DEBUG` | `1` shows error details (only honoured when `APP_ENV=local`). |
| `APP_URL` | Optional full site URL. Leave empty to auto-detect (subfolder installs work). |
| `TIMEZONE` | Default `Asia/Kolkata`. |
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS` | Database connection. |
| `SESSION_DRIVER` | `file` (default) or `db` (several web servers). |
| `SESSION_LIFETIME` | Seconds before an idle session may be deleted (default 86400). |
| `MAIL_DRIVER` | `smtp` (real sending), `mail` (PHP `mail()`), `log` (local testing only). |
| `MAIL_HOST`, `MAIL_PORT`, `MAIL_ENCRYPTION` | SMTP server, port (587 or 465), `tls` / `ssl` / `none`. |
| `MAIL_USERNAME`, `MAIL_PASSWORD` | SMTP login and password / App Password / SMTP key. |
| `MAIL_FROM`, `MAIL_FROM_NAME` | Sender shown to customers. |
| `MAIL_VERIFY_TLS` | Keep `1`. |
| `MAIL_CAFILE` | Windows/XAMPP certificate bundle path, only if you get certificate errors. |
| `OTP_REGISTRATION` | `1` = customers must verify their email with a code at signup. |
| `APP_KEY` | Random secret used to protect OTP codes in the database. Generate with `php -r "echo bin2hex(random_bytes(24));"`. **Not** your email password. |
| `BUSINESS_PHONE`, `BUSINESS_EMAIL`, `BUSINESS_ADDRESS` | Shown in the footer. |
| `SMS_PROVIDER` | Reserved; SMS OTP is not implemented. |

### 11.2 Admin → Settings (business rules, stored in `app_settings`)

| Setting | Default | Meaning |
|---|---|---|
| Payment methods | cash, UPI | Which offline methods customers and partners may use. |
| No-delivery weekdays | none | Days of the week with no delivery (skipped for preorders). |
| `preorder_enabled` | on | Switch next-day preorders on/off. |
| `preorder_days_offered` | 3 | Number of upcoming dates a customer can choose. |
| `slot_lead_minutes` | 30 | Same-day slot closes this many minutes before it starts. |
| `cancel_window_minutes` | 30 | How long a customer may cancel a confirmed Fresh Now order. |
| `cart_max_qty` | 20 | Max quantity of one product. |
| `orders_per_hour_limit` | 10 | Anti-abuse limit per customer. |
| `reserve_stock_for_preorders` | off | Also reserve stock for preorders. |
| `allow_partial_payment` | off | Let partners record part payments. |
| `allow_unpaid_delivery` | off | Allow delivery without payment (reason required). |
| `show_partner_contact` | on | Show partner name/phone to the customer while delivering. |

Also in Settings: **Holidays**, **Testimonials**, and the **Email & OTP** status panel with a "Send test" button (the mail password is never displayed).

Other admin-managed data: societies (rules, partners), delivery slots, products/categories, offers/coupons.

---

## 12. Email, OTP and password reset

### 12.1 What it does

* **Signup** (when `OTP_REGISTRATION=1` and mail works): the form collects the details, a 6-digit code is emailed, and the account is created **only after the code is entered**. The form data waits in the session (30 minutes), not in the database.
* **Forgot password** (always): the customer enters mobile or email, a code is emailed, then they enter the code and a new password. The screen is identical whether or not the account exists (no account guessing). Customers without an email must phone the shop. Delivery partners and admins are reset by an admin (Admin → Delivery partners → Reset password gives a one-time temporary password that must be changed at first login).
* Code rules (`OtpService`): 10 minutes validity, 5 wrong tries then the code is void, a new code possible after 60 seconds, at most 5 codes per hour per address, a new code voids the old one, only an HMAC hash is stored.

### 12.2 Setting it up (Gmail example)

Edit `.env` in the project root:

```ini
MAIL_DRIVER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_ENCRYPTION=tls
MAIL_USERNAME=youraddress@gmail.com
MAIL_PASSWORD=your-16-letter-app-password
MAIL_FROM=youraddress@gmail.com
MAIL_FROM_NAME="Healthy Fasal"
OTP_REGISTRATION=1
APP_KEY=<random value from the command above>
```

Gmail requires an **App Password** (Google Account → Security → 2-Step Verification → App passwords). If the password contains `#` or spaces, put it in double quotes.

| Provider | Host | Port / encryption | Username | Password |
|---|---|---|---|---|
| Gmail / Workspace | `smtp.gmail.com` | 587 / tls | full address | App Password |
| Brevo | `smtp-relay.brevo.com` | 587 / tls | SMTP login from Brevo | SMTP key |
| SendGrid | `smtp.sendgrid.net` | 587 / tls | the word `apikey` | API key (`SG.…`) |
| Zoho | `smtp.zoho.in` (or `.com`) | 465 / ssl | full address | app password |
| Hosting mailbox (cPanel, Hostinger…) | `mail.yourdomain.com` | 465 / ssl | mailbox address | mailbox password |
| Amazon SES | `email-smtp.<region>.amazonaws.com` | 587 / tls | SES SMTP user | SES SMTP password |

### 12.3 Test it

1. `php bin/test-mail.php you@example.com` (note: the address is required). Output shows the driver, server and either `SUCCESS` or the server's exact error.
2. Admin → Settings → Email & OTP: status should say **Ready**.

### 12.4 Windows / XAMPP checklist

1. `extension=openssl` enabled in `C:\xampp\php\php.ini`, Apache restarted.
2. If you see *certificate verify failed*: add `MAIL_CAFILE=C:\xampp\php\extras\ssl\cacert.pem` to `.env`.
3. Firewall/antivirus must allow outgoing connections on port 587 (or 465).

### 12.5 Common mail errors

| Message | Cause / fix |
|---|---|
| "E-mail is not configured…" | `MAIL_DRIVER` is `log` while `APP_ENV=production`. Set `MAIL_DRIVER=smtp`. |
| `535 … not accepted` | Wrong username/password. Gmail needs an App Password. |
| "Cannot connect to host:port" | Wrong host/port/encryption, or outgoing SMTP is blocked. |
| "TLS handshake failed" / certificate error | Try `ssl` + 465, or set `MAIL_CAFILE`. |
| Lands in spam | Verify the sender/domain with your provider (SPF/DKIM). |

In **local mode** with `MAIL_DRIVER=log`, nothing is sent; codes are shown on screen and written to `storage/logs/mail.log` for testing. This never happens in production.

---

## 13. Security model

| Area | How it is handled |
|---|---|
| Passwords | `password_hash()` / `password_verify()` (auto re-hash). Minimum 8 characters with letters and numbers. Plain passwords are never stored or shown. |
| Login | Separate login pages per role (`/login`, `/admin/login`, `/partner/login`); a customer's credentials do not work on the admin page. Throttling: 5 failures per account and 25 per IP in 15 minutes. |
| Sessions | HttpOnly, SameSite=Lax, `Secure` in production, strict mode, ID regenerated on login, destroyed on logout. |
| Roles | `Router::guard()` checks the role for every protected route on every request; user status and role are re-read from the database each time, so disabling an account takes effect immediately. |
| Data ownership | SQL always includes `user_id` / `partner_id`, so another person's order or address returns 404. |
| CSRF | Every POST needs a valid token (form field `_csrf` or header `X-CSRF-Token`), otherwise HTTP 419. |
| SQL injection | PDO prepared statements everywhere. |
| XSS | All output escaped with `e()`. |
| Uploads | MIME and image-content checks, size and dimension limits, image **re-encoded** to WebP (or PNG) under a random name, PHP execution disabled in `public/uploads`. |
| Checkout | Idempotency key, row locks, server-side recalculation of every price. |
| Payments | Append-only collection records, verification tick, admin-only dispute/refund with reasons. |
| Email OTP | Hashed codes, expiry, attempt limits, resend cool-down, hourly cap, generic responses. |
| Auditing | `audit_logs` records admin actions, status changes, payments, logins, password changes. |
| Errors | Production never shows stack traces; details go to `storage/logs/php-error.log`. |
| Files | `.env`, `app/`, `config/`, `database/`, `storage/`, `views/`, `tests/`, `bin/` blocked from the web. Security headers (`X-Frame-Options`, `nosniff`, referrer policy) are set. |
| CSV | Cells starting with `= + - @` are neutralised (formula injection). |

---

## 14. Performance, caching and scaling

* All list/filter queries are indexed. The storefront fetches `limit + 1` rows instead of running `COUNT(*)`.
* `Cache` (APCu if installed, otherwise files in `storage/cache`) holds shared storefront data for 30–60 seconds (categories, featured/bestseller lists, offers, societies, testimonials) and dashboard figures (KPIs 20 s, charts 60 s). **Any POST under `/admin` clears the whole cache automatically** (`Cache::bump()`), so admin changes show immediately. Stock is always re-checked live when adding to the cart and at checkout, so cached product cards can never cause overselling.
* Settings are read once per request (and cached in APCu if available).
* `.htaccess` enables gzip and one-year caching for images, CSS and JS.
* Images are WebP.
* Optional `SESSION_DRIVER=db` for several servers.

Measured on a weak single-core test machine (PHP built-in server, no OPcache): 40 simultaneous checkouts for 10 units produced exactly 10 orders; 60,000 orders did not slow customer queries (about 1 ms); admin reports took 0.4–0.5 s; the storefront served roughly 230–640 requests per second.

When you outgrow one server: enable OPcache and APCu, move MySQL to its own server, use `SESSION_DRIVER=db`, put a CDN in front of `/assets` and `/uploads`, add read replicas for reports, and add a nightly summary table for very large report ranges.

---

## 15. Automated tests

All tests are plain PHP scripts (no Composer). **They wipe their database, so use a separate one whose name contains `test`** (`tests/run.php` refuses to run otherwise).

Create an empty database named `healthy_fasal_test` (phpMyAdmin or `CREATE DATABASE`), then in PowerShell:

```powershell
$env:DB_NAME="healthy_fasal_test"; $env:DB_USER="root"; $env:DB_PASS=""
C:\xampp\php\php.exe tests\run.php           # business logic
C:\xampp\php\php.exe tests\concurrency.php   # parallel checkouts
C:\xampp\php\php.exe tests\mail.php          # SMTP client

# HTTP tests need a running server on the same test database:
Start-Process C:\xampp\php\php.exe -ArgumentList "-S","127.0.0.1:8081","-t","public","public\router.php"
$env:BASE_URL="http://127.0.0.1:8081"; C:\xampp\php\php.exe tests\http.php

# Email-OTP flows need a local-mode server with OTP and the log driver:
$env:APP_ENV="local"; $env:OTP_REGISTRATION="1"; $env:MAIL_DRIVER="log"
Start-Process C:\xampp\php\php.exe -ArgumentList "-S","127.0.0.1:8082","-t","public","public\router.php"
$env:BASE_URL="http://127.0.0.1:8082"; C:\xampp\php\php.exe tests\http_otp.php
```
(On Linux/Mac use `DB_NAME=… php tests/run.php` and `&` for background servers.)

| Suite | Checks | Covers |
|---|---|---|
| `tests/run.php` | 118 | login and throttling, password rules, cart totals, offers/coupons, delivery fee, preorder dates/holidays/slots, order placement, address snapshots, idempotency, stock, cancel/deliver, assignment rules, payments, refunds, unpaid-delivery rule, cancellation policy, reports, email OTP rules |
| `tests/concurrency.php` | 11 | 40 parallel buyers of 10 units (no overselling), 10 parallel identical checkouts (one order) |
| `tests/mail.php` | 31 | SMTP client against a fake mail server: plain, STARTTLS, SSL, wrong password, closed port, certificate checks, header injection |
| `tests/http.php` | 92 | The six acceptance tests (customer, preorder, partner, admin, security, mobile heuristics) over real HTTP |
| `tests/http_otp.php` | 24 | Signup with email code, wrong/reused codes, password reset by code |

Helper files: `tests/bootstrap.php` (mini test framework), `tests/client.php` (HTTP client with cookies/CSRF), `tests/fake_smtp.php` (fake mail server).

Limits of testing: the SMTP client is tested against a fake server, not a live Gmail account (use `bin/test-mail.php` once with real credentials). Layouts were checked in a headless browser at phone and desktop widths, but real-phone testing is still recommended.

---

## 16. Everyday maintenance

| I want to… | Do this |
|---|---|
| Add a product / change price / stock | Admin → Products. Stock changes (purchase/adjustment) are logged. |
| Replace a product photo | Admin → Products → Edit → Image (JPG/PNG/WebP up to 4 MB). Or drop a file named like the product in `public/assets/images/products/`. |
| Add a category | Admin → Categories (name, order, image, show/hide). |
| Add a society | Admin → Societies → Add (fee, minimum order, cut-off, partners). Then optional own slots in Delivery slots. |
| Change delivery slots | Admin → Delivery slots (global defaults or per society, with capacity). |
| Close deliveries on a day | Admin → Settings → Holidays (a date) or "No delivery on these weekdays". |
| Add a delivery partner | Admin → Delivery partners → Add; tick their societies. A temporary password is shown once. |
| Create a coupon / offer | Admin → Offers & coupons. Blank code = automatic offer. |
| Change business rules | Admin → Settings (see §11.2). |
| Change the logo | Replace `public/assets/images/logo.png` (the code looks for `logo.png`, then `logo.webp`, `logo.jpg`, then `logo.svg` if that file exists). |
| Change the homepage banner | Replace `public/assets/images/banners/hero.png` (the home view `views/home/index.php` references it). |
| Change brand colours / fonts | `public/assets/css/app.css` (CSS variables at the top). Your login page and OTP screens use their own blue styles in `views/auth/login.php` and `views/partials/login-form-style.php`. |
| Change footer contact details | `.env` (`BUSINESS_*`). |
| Change policy text | `views/home/page.php` (template text, have it reviewed). |
| Add a setting | Add a default in `Settings::DEFAULTS`, add a field in `views/admin/settings.php`, save it in `AdminBusinessController::settingsSave`, read it with `Settings::get()/bool()/int()`. |
| Add a page | Add a route in `routes/web.php` (with its roles), a controller method, a view in `views/`. |
| Change the database | Edit `database/schema.sql` for new installs **and** add a numbered file in `database/migrations/` for existing ones; run `php bin/migrate.php`. |
| Add a new order status | Update the `status` enum (schema + migration), `ORDER_STATUSES` in `functions.php`, `OrderService::TRANSITIONS`, and the timeline. |
| Regenerate the product illustrations | `python3 tools/generate_images.py` (needs Pillow and numpy). **Careful:** this overwrites the default product images, including any you customised. |
| Clear the cache | Delete files in `storage/cache/`, or save anything in the admin panel. |
| See errors | `storage/logs/php-error.log`. |
| See who did what | Table `audit_logs`, and the history sections on each order page. |

**Backups:** regularly run `mysqldump -u USER -p healthy_fasal > backup.sql` and copy `public/uploads/` and your `.env`.

---

## 17. Deploying and go-live checklist

**Deploying to a live server**

1. Upload the project; point the domain's document root at `public/`; enable HTTPS.
2. Create the MySQL database and user. Run `php bin/setup.php --db=… --user=… --pass=… --env=production` (no `--seed`), or import `database/schema.sql` and create `.env` by hand with `APP_ENV=production`, `APP_DEBUG=0`.
3. Create the first admin: `php bin/create-admin.php "Owner Name" 9876543210 owner@mail.com`.
4. Make sure the web server can write to `storage/logs`, `storage/cache`, `public/uploads/products`, `public/uploads/categories`, and nothing else.
5. Configure mail (§12) and run `php bin/test-mail.php`.
6. Nginx users: serve `public/`, route unknown URLs to `index.php`, deny `.env`, block PHP in `/uploads/`.

**Upgrading an existing install:** back up, upload the new files (keep `.env` and `public/uploads`), run `php bin/migrate.php`.

**Go-live checklist**

- [ ] `APP_ENV=production`, `APP_DEBUG=0`, HTTPS on, strong DB password, secrets only in `.env`
- [ ] `public/check-gd.php` **deleted**
- [ ] No seed/test accounts; your own admin created; password changed
- [ ] Real logo, business phone/email/address, societies, slots, prices, stock, product photos
- [ ] Admin → Settings reviewed (payment methods, off days, cancel window, unpaid delivery rule)
- [ ] Policy pages reviewed by the owner
- [ ] Email tested (`bin/test-mail.php`), `OTP_REGISTRATION=1` set, new App Password used (never share passwords in chat or Git)
- [ ] `APP_KEY` set to its own random value
- [ ] Backups scheduled (database and uploads)
- [ ] OPcache enabled on the server
- [ ] Full flow tried on a real Android phone and an iPhone: register, order, preorder, admin confirm/assign, partner collect/deliver

---

## 18. Troubleshooting

| Problem | Likely cause and fix |
|---|---|
| Blank page / error 500 | Look at `storage/logs/php-error.log`. For development set `APP_ENV=local` and `APP_DEBUG=1`. |
| "Could not connect" / database error | Check `DB_*` in `.env` and that MySQL is running. |
| "Table … doesn't exist" | Schema not imported. Run `bin/setup.php` or import `database/schema.sql`; after updates run `bin/migrate.php`. |
| "Session expired" on every action | Cookies blocked, or the PHP session folder is not writable. |
| Image upload fails | `gd` or `fileinfo` extension disabled (check `public/check-gd.php`), or `public/uploads/*` not writable, or file over 4 MB. |
| Redirect after an action shows a blank page / "headers already sent" | Something printed output before the page (usually a blank line before `<?php` in a PHP file). Remove it. |
| Emails not sent | See §12.5. Check Admin → Settings → Email & OTP. |
| Wrong times | Set `TIMEZONE` in `.env`. |
| Pages look unstyled | The CDN (Bootstrap/Fonts) is blocked or offline. |
| Site in a subfolder | Works automatically; or set `APP_URL=https://example.com/shop`. |
| Changes in admin do not show on the storefront | Cache is cleared automatically on admin saves; if you changed the database directly, delete `storage/cache/*`. |
| `php` is not recognised in PowerShell | Use the full path `C:\xampp\php\php.exe` or add `C:\xampp\php` to PATH. |

---

## 19. Known limitations and future ideas

**Not included (by design or not yet built)**

* **SMS OTP / WhatsApp** (needs a provider account; in India also DLT registration). Email OTP is implemented.
* **Online payments** (no gateway; payments are collected at the door).
* **Live GPS tracking** (status timeline only).
* **Order notification emails/SMS** (only OTP and reset emails are sent today).
* **GST tax invoice** (the invoice page is an order summary).
* One image per product, no variants (sizes), no returns/exchange module, no stock alert emails, no multi-language or multi-currency.
* Product photos are rendered illustrations unless you upload real photos.
* Reports over very large date ranges run live queries (0.4–0.5 s at 60,000 orders).
* No browser-based automated tests in the repository (they were run manually during development).

**Good next steps**

1. Order confirmation / out-for-delivery notifications (email, then WhatsApp/SMS).
2. Real product photography and multiple images per product.
3. Wallet / loyalty points, subscriptions (daily milk-style repeat orders).
4. A nightly summary table for faster reports.
5. A "rider route" view that groups a partner's orders by society and building.
6. Hindi/Marathi translations.

---

## 20. Glossary

| Term | Meaning |
|---|---|
| **Society** | A housing complex the shop delivers to. Has its own fee, minimum order, cut-off and slots. |
| **Slot** | A delivery time window (for example 7–10 AM). |
| **Cut-off** | The latest time on a day to order for the earliest next-day delivery. |
| **Preorder** | An order placed today for the next eligible delivery day. |
| **Fresh Now** | An order from today's stock for delivery today. |
| **Reserved stock** | Stock promised to open orders; not yet physically gone. |
| **Sellable stock** | `stock_qty − reserved_qty`. |
| **Snapshot** | A copy of address/price stored on an order so history never changes. |
| **Idempotency key** | One-time token that makes sure a repeated checkout request creates only one order. |
| **Collection** | A record that a partner received money at the doorstep. |
| **Reconciliation** | Comparing what partners collected with what the system expects. |
| **Paise** | 1/100 of a rupee; all maths is done in whole paise. |
| **OTP** | One-time password: the 6-digit code emailed to the customer. |
| **CSRF** | Attack where another website tricks a logged-in user's browser into submitting a form; blocked by the token on every POST. |
| **Audit log** | A permanent list of who did what and when. |

---

*Healthy Fasal – "Fresh from Our Shop to Your Doorstep."*

# Healthy Fasal – Manual Testing Guide (from scratch to sign-off)

Use this guide to test **every feature by hand** before handing the system to the client. It has **234 test cases** in 16 groups; **159 are priority P1** (must pass).

Record your results in the Excel file **`Healthy_Fasal_QA_Tracker.xlsx`** (same test IDs). The *Summary* sheet tells you automatically whether the project is **READY FOR HANDOVER**.

---

## Contents

1. [How testing works](#1-how-testing-works)
2. [Prepare everything from scratch](#2-prepare-everything-from-scratch)
3. [Test data cheat sheet](#3-test-data-cheat-sheet)
4. [Handy tools and quick fixes](#4-handy-tools-and-quick-fixes)
5. [Suggested schedule](#5-suggested-schedule)
6. [The test cases](#6-the-test-cases)
7. [Reporting bugs and retesting](#7-reporting-bugs-and-retesting)
8. [Exit criteria and sign-off](#8-exit-criteria-and-sign-off)

---

## 1. How testing works

* Go through the groups in order (PRE, HOME, SHOP, … DEP). Each test has **Steps** and an **Expected result**.
* Do the steps exactly. If what you see is different from *Expected*, that is a **Fail**: write down what you saw, take a screenshot, and log a bug (section 7).
* **Priority:** **P1** = must pass before handover · **P2** = should pass · **P3** = nice to have.

| Result | Meaning |
|---|---|
| **Not run** | Not tested yet (default). |
| **Pass** | Behaved exactly as 'Expected result'. |
| **Fail** | Behaved differently, or showed an error. Write a Bug ID and note what you saw. |
| **Blocked** | Could not test because something else is broken. Say what in Notes. |
| **N/A** | Does not apply (for example, a feature switched off). Say why in Notes. |

**Automated tests already cover** business rules, security and concurrency (276 checks, see README section 15). Manual testing is different: it checks what a *human* sees and feels: layout, wording, phone behaviour, real emails, real devices, and the full story between customer, admin and delivery boy. So focus on **looking and using**, not on reading code.

---

## 2. Prepare everything from scratch

### 2.1 What you need

| Item | Why |
|---|---|
| Windows PC with XAMPP (Apache + MySQL running) | Runs the system |
| Chrome, plus Edge or a second Chrome profile, plus Incognito | Be logged in as **admin**, **partner** and **customer** at the same time without clashes |
| An Android phone (and an iPhone if you can) on the same Wi-Fi | Real mobile testing. Open `http://<your-PC-IP>/healthy-fasal/public/` (find the IP with `ipconfig`; allow Apache in Windows Firewall) |
| phpMyAdmin open in a tab (`http://localhost/phpmyadmin`) | To check data and run the integrity queries |
| Two real email addresses you can open (Gmail lets you use `yourname+1@gmail.com`, `+2` … all arrive in one inbox) | To test OTP signup many times |
| The Excel tracker + a screenshot tool (Windows: Win+Shift+S) | Record results and evidence |
| A calculator | Verify prices and totals |

### 2.2 Build a clean test copy

1. Start **Apache** and **MySQL** in the XAMPP Control Panel.
2. In phpMyAdmin delete the database `healthy_fasal` (if it exists).
3. Open PowerShell in the project folder and run (no demo orders, so every number is predictable):

```powershell
C:\xampp\php\php.exe bin\setup.php --db=healthy_fasal --user=root --env=local --seed --force
```
4. Open `.env` and set the email lines (README section 12) and:

```ini
APP_ENV=local
MAIL_DRIVER=smtp          # real emails; use 'log' only if you cannot send mail yet
OTP_REGISTRATION=1
```
5. Run the email check: `C:\xampp\php\php.exe bin\test-mail.php youraddress@gmail.com` → must say **SUCCESS**.
6. Open the site: `http://localhost/healthy-fasal/public/` (or the address you use). You should see the home page with products.
7. Optional safety net: run the automated tests first (README section 15). All must say `Failed: 0`.

> **Start over any time:** repeat step 3 (it wipes and reinstalls). Do this before each full test round so old data does not confuse results.

### 2.3 Set up your three "people"

| Browser window | Log in as | Address |
|---|---|---|
| Chrome (normal) | **Admin** `9000000001` | `/admin/login` |
| Edge / second profile | **Delivery partner** `9000000002` | `/partner/login` |
| Chrome Incognito + your phone | **Customer** `9000000010` or a new account you register | `/login`, `/register` |

### 2.4 Testing things that depend on time (without changing the Windows clock)

Do **not** change the computer clock (it breaks logins). Change the *settings* instead:

* **Today's delivery slot open/closed:** Admin → Delivery slots → edit a slot's start time (a slot closes 30 minutes before it starts).
* **Preorder cut-off:** Admin → Societies → Edit → *Preorder cut-off* (set it a few minutes in the past or future).
* **No-delivery days:** Admin → Settings (weekdays) and Holidays.
* **Cancel window / limits:** Admin → Settings.

---

## 3. Test data cheat sheet

From the sample data (`database/seed.sql`). If your data was changed, adjust the numbers.

### Accounts (password for all: `Password@123`)

| Who | Login (mobile) | Page | Notes |
|---|---|---|---|
| Admin | `9000000001` | `/admin/login` | Password@123 |
| Delivery partner 1 (Ravi Kumar) | `9000000002` | `/partner/login` | Password@123 (serves Green Valley, Sunrise Heights) |
| Delivery partner 2 (Suresh Patil) | `9000000003` | `/partner/login` | Password@123 (serves Sunrise Heights, Maple Gardens) |
| Customer 1 (Anita Sharma) | `9000000010` | `/login` | Password@123 (Green Valley Residency) |
| Customer 2 (Rahul Deshmukh) | `9000000011` | `/login` | Password@123 (Sunrise Heights) |

### Societies

| Society | Minimum order | Delivery fee | Free delivery above | Preorder cut-off | Status |
|---|---|---|---|---|---|
| Green Valley Residency | ₹150 | ₹20 | ₹400 | 8:00 PM | Active |
| Sunrise Heights | ₹200 | ₹25 | ₹500 | 9:00 PM | Active |
| Maple Gardens | ₹100 | ₹15 | ₹350 | 7:30 PM | Active |
| Lotus Enclave (coming soon) | ₹200 | ₹30 | ₹600 | 8:00 PM | Inactive |

### Delivery slots (default for all societies)

| Slot | Time |
|---|---|
| Morning | 7:00 AM – 10:00 AM |
| Late Morning | 10:00 AM – 1:00 PM |
| Evening | 5:00 PM – 8:00 PM |

### Offers

| Code | Rule |
|---|---|
| `WELCOME10` | 10% off, minimum ₹200, maximum ₹100, once per customer |
| `FRESH50` | ₹50 off, minimum ₹500 |
| `(automatic)` | 5% off Fruits, no minimum |

### Products (price per unit, original price, opening stock)

| Product | Unit | Price | Original | Stock |
|---|---|---|---|---|
| Tomato | 1 kg | ₹32 | ₹40 | 120 |
| Potato | 1 kg | ₹28 | ₹32 | 200 |
| Onion | 1 kg | ₹30 | ₹36 | 180 |
| Cauliflower | per piece | ₹40 | ₹50 | 60 |
| Cabbage | per piece | ₹35 | ₹42 | 50 |
| Carrot | 500 g | ₹24 | ₹30 | 90 |
| Cucumber | 500 g | ₹20 | ₹25 | 80 |
| Capsicum | 250 g | ₹22 | ₹28 | 70 |
| Spinach (Palak) | per bunch | ₹15 | ₹20 | 60 |
| Coriander (Dhania) | per bunch | ₹10 | ₹12 | 100 |
| Green Chilli | 250 g | ₹18 | ₹22 | 60 |
| Lemon | 250 g | ₹25 | ₹30 | 75 |
| Apple | 1 kg | ₹160 | ₹190 | 60 |
| Banana | per dozen | ₹55 | ₹65 | 70 |
| Orange | 1 kg | ₹90 | ₹110 | 55 |
| Pomegranate | 1 kg | ₹170 | ₹200 | 40 |
| Grapes | 500 g | ₹70 | ₹85 | 45 |
| Papaya | per piece | ₹60 | ₹75 | 30 |
| Mango (Seasonal) | 1 kg | ₹140 | ₹170 | 35 |

### Worked example (use it to verify the cart)

Society **Green Valley**, cart **3 × Tomato + 2 × Apple**:

| Step | Calculation | Result |
|---|---|---|
| Subtotal | 3×32 + 2×160 | ₹416 |
| Automatic fruit offer | 5% of apples (₹320) | − ₹16 |
| Delivery | after discount ₹400 is not below ₹400 | Free |
| **Total** | | **₹400** |
| With coupon `WELCOME10` instead | 10% of ₹416 = ₹41.60; delivery ₹20 (₹374.40 is below ₹400) | **₹394.40** |

---

## 4. Handy tools and quick fixes

Run these in phpMyAdmin → SQL tab (database `healthy_fasal`). **Only on the test copy.**

| I want to… | SQL |
|---|---|
| Clear the login lock (5 wrong tries) | `DELETE FROM login_attempts;` |
| Allow coupons to be used again | `DELETE FROM offer_redemptions; UPDATE offers SET used_count=0;` |
| Skip the 60 second wait between OTP emails | `DELETE FROM email_otps;` |
| Set a product's stock quickly | `UPDATE products SET stock_qty=3 WHERE name='Apple';` (or use Admin → Product → Stock) |
| See what happened | `SELECT * FROM audit_logs ORDER BY id DESC LIMIT 30;` |
| Read errors | open `storage\logs\php-error.log` |

---

## 5. Suggested schedule

| Day | Groups | Notes |
|---|---|---|
| 1 | PRE, HOME, SHOP, CART, AUTH, OTP, FRG | Visitor and signup side |
| 2 | ACC, CHK, first half of ADM (products, categories, orders, preorders) | Core selling flow |
| 3 | Second half of ADM (people, offers, societies, slots, settings, reports), PRT | Back office and delivery |
| 4 | E2E, SEC, UX, CONC, DB | Whole stories, security, phones |
| 5 | Fix, **retest every failed test**, run all automated tests again, DEP | Handover prep |

Two testers working in parallel (one customer side, one admin/partner side) halve the time.

---

## 6. The test cases

### 0. Environment and smoke test

*Do this first. If anything here fails, stop and fix it before testing features.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **PRE-01** | P1 | Fresh install works | 1. Stop XAMPP Apache/MySQL if running, start them again.<br>2. In phpMyAdmin drop database `healthy_fasal` (or use a new name).<br>3. Run: C:\xampp\php\php.exe bin\setup.php --db=healthy_fasal --user=root --pass= --env=local --seed<br>4. Open the site in Chrome. | Setup prints 'Schema imported', 'Sample data and test accounts imported', '.env written' and the test logins. The home page opens with logo, hero image and products. No PHP warnings on screen. | ☐ Pass ☐ Fail |
| **PRE-02** | P1 | Automated tests pass (first gate) | 1. Create an empty database `healthy_fasal_test`.<br>2. Follow README section 15 and run tests\run.php, concurrency.php, mail.php, http.php, http_otp.php. | Every suite ends with 'Failed: 0' (276 checks in total). | ☐ Pass ☐ Fail |
| **PRE-03** | P1 | Error log is clean | 1. After the smoke steps open storage\logs\php-error.log. | File is empty or has no new errors (warnings about your own tests are not expected). | ☐ Pass ☐ Fail |
| **PRE-04** | P1 | Email sending works (real SMTP) | 1. Fill the MAIL_* lines in .env (README section 12).<br>2. Run: php bin\test-mail.php youraddress@gmail.com<br>3. Open Admin > Settings > Email & OTP and press Send test. | Command prints SUCCESS and the email arrives (check Spam). The admin panel shows Status = Ready. | ☐ Pass ☐ Fail |
| **PRE-05** | P1 | Tools ready | 1. Prepare: Chrome (admin), Edge or Chrome profile 2 (partner), Chrome incognito (customer), one Android phone, one iPhone if possible, phpMyAdmin open in a tab, this tracker open, a second email address. | You can be logged in as admin, partner and customer at the same time without sessions clashing. | ☐ Pass ☐ Fail |
| **PRE-06** | P2 | Time zone is correct | 1. Place any order and open its detail page. | Order time matches your real local time (IST). | ☐ Pass ☐ Fail |

### 1. Home page and general site

*Public pages, navigation, footer. Test as a logged-out visitor.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **HOME-01** | P1 | Home page desktop layout | 1. Open / on a laptop (1440 px wide). | Header shows logo, search bar, society button, Log in, cart icon; menu: Fruits, Vegetables, Seasonal, Offers, Preorder. Hero says 'Freshness Delivered to Your Society.' with buttons 'Shop Fresh Now' and 'Preorder for Tomorrow'. | ☐ Pass ☐ Fail |
| **HOME-02** | P1 | Home page on phone | 1. Open / on a phone (or DevTools 390 px).<br>2. Scroll the whole page. | No sideways scrolling. Bottom bar shows Home, Categories, Cart, Orders, Profile. Search box visible. Text readable. | ☐ Pass ☐ Fail |
| **HOME-03** | P1 | Choose a society | 1. Tap the society button in the header.<br>2. Choose Green Valley Residency. | Button now shows the society name. Home shows 'Delivering to Green Valley Residency' with the next delivery date and 'Order before 8:00 PM'. | ☐ Pass ☐ Fail |
| **HOME-04** | P2 | Only active societies are listed | 1. Open the society list. | Green Valley, Sunrise Heights, Maple Gardens are listed. 'Lotus Enclave' (inactive) is not. | ☐ Pass ☐ Fail |
| **HOME-05** | P1 | All home sections appear | 1. Scroll the home page. | Categories (Vegetables, Fruits, Leafy Vegetables, Seasonal Produce; no Daily Essentials), Today's Fresh Picks, Bestselling Products, Special Offers, Preorder for Tomorrow banner, How it works (4 steps), Serving your society, Why choose, FAQ, footer. Testimonials section is hidden when none exist. | ☐ Pass ☐ Fail |
| **HOME-06** | P2 | FAQ accordion | 1. Click each FAQ question. | Opens and closes smoothly; only one open at a time. | ☐ Pass ☐ Fail |
| **HOME-07** | P1 | Category cards | 1. Click 'Fruits' card. | Opens /shop showing only fruits. | ☐ Pass ☐ Fail |
| **HOME-08** | P1 | Hero buttons | 1. Click 'Shop Fresh Now'. Go back. Click 'Preorder for Tomorrow'. | First opens the shop in Fresh Now mode. Second opens the shop with a green strip 'Preorder for Tomorrow mode is on' and the earliest delivery date if a society is selected. | ☐ Pass ☐ Fail |
| **HOME-09** | P2 | Footer | 1. Check footer contact details and links. | Phone, email, address come from .env (BUSINESS_*). Policy links open. 'Delivery partner login' and 'Staff' open the right login pages. | ☐ Pass ☐ Fail |
| **HOME-10** | P2 | Policy pages | 1. Open /page/terms, /page/privacy, /page/delivery-policy, /page/refund-policy. | Each opens with a heading and the 'template text' notice. (Owner must review the wording before launch.) | ☐ Pass ☐ Fail |
| **HOME-11** | P2 | 404 page | 1. Open /this-page-does-not-exist. | Friendly 404 page with 'Go to home' button. No stack trace or file paths. | ☐ Pass ☐ Fail |
| **HOME-12** | P1 | Images | 1. Look at logo, hero, category tiles, product images on Home and Shop. | No broken image icons. Your own logo and hero image show. | ☐ Pass ☐ Fail |
| **HOME-13** | P3 | Society chips in 'Serving your society' | 1. Click a society chip. | That society becomes the selected society. | ☐ Pass ☐ Fail |

### 2. Shop, search, filters, quick view

*Product browsing as a visitor.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **SHOP-01** | P1 | Product list and Load more | 1. Open /shop.<br>2. Count cards. Press 'Load more' until it disappears. | First 12 products show. Load more adds the rest (19 sample products in total) without reloading the page, then the button disappears. | ☐ Pass ☐ Fail |
| **SHOP-02** | P1 | Category filter | 1. Tap each chip: All, Vegetables, Fruits, Leafy Vegetables, Seasonal Produce. | Only products of that category show; the list updates without a full reload. | ☐ Pass ☐ Fail |
| **SHOP-03** | P1 | Search | 1. Search 'tom'.<br>2. Search 'zzz'.<br>3. Search <script>alert(1)</script> | Tomato appears. 'zzz' shows 'No products match'. The script text is shown as plain text; no popup. | ☐ Pass ☐ Fail |
| **SHOP-04** | P2 | Price filter | 1. Set Min 50 and Max 100. | Only products priced from ₹50 to ₹100 show. | ☐ Pass ☐ Fail |
| **SHOP-05** | P2 | Sorting | 1. Sort by Price low to high, high to low, Biggest discount, Name A-Z. | Order changes correctly each time. | ☐ Pass ☐ Fail |
| **SHOP-06** | P2 | Filters combine and survive refresh | 1. Choose Fruits + Max 100 + Price low to high.<br>2. Refresh the page. | Same filters are still applied; the address bar contains the filters. | ☐ Pass ☐ Fail |
| **SHOP-07** | P2 | Mobile filter panel | 1. On a phone press 'Price & sort'. | Price and Sort fields open and close. Header search works. | ☐ Pass ☐ Fail |
| **SHOP-08** | P1 | Product card content | 1. Look at Tomato and Apple cards. | Shows image, category, name, unit, current price, struck-through original price and badge: Tomato 20% OFF (₹32 / ₹40), Apple 16% OFF (₹160 / ₹190). | ☐ Pass ☐ Fail |
| **SHOP-09** | P1 | Quick view | 1. Tap a product image or name.<br>2. Try Add to cart inside it.<br>3. Close with X and with the Escape key. | A window shows name, category, unit, price, discount, stock status, description. Add works. It closes both ways. | ☐ Pass ☐ Fail |
| **SHOP-10** | P2 | Low-stock badge | 1. In Admin set a product so only 3 are sellable (stock 3).<br>2. Open the shop. | Card shows 'Only 3 left'. | ☐ Pass ☐ Fail |
| **SHOP-11** | P1 | Out of stock behaviour | 1. In Admin set a product's stock to 0.<br>2. View it in Fresh Now mode, then Preorder mode. | Fresh Now: 'Out of stock' badge and a disabled 'Out of stock today' button. Preorder mode: 'Preorder only' badge and an enabled 'Preorder' button. | ☐ Pass ☐ Fail |
| **SHOP-12** | P1 | Deactivated product disappears | 1. Admin > Products > Deactivate a product.<br>2. Refresh Shop and Home, search for it. | It is gone everywhere within a few seconds. | ☐ Pass ☐ Fail |
| **SHOP-13** | P2 | Hidden category hides its products | 1. Admin > Categories > Hide 'Fruits'. | Fruits chip and every fruit product disappear from Shop and Home. Show it again afterwards. | ☐ Pass ☐ Fail |
| **SHOP-14** | P2 | Offers page | 1. Open /offers. | Active offers listed with codes (WELCOME10, FRESH50) or 'Applied automatically', and valid-till dates. 'Discounted products' shows only discounted items. | ☐ Pass ☐ Fail |
| **SHOP-15** | P2 | Seasonal link | 1. Click Seasonal in the menu. | Shows seasonal products (Mango). | ☐ Pass ☐ Fail |
| **SHOP-16** | P1 | Mode switch | 1. Toggle 'Order Fresh Now' and 'Preorder for Tomorrow' on the shop page, then open Home and Cart. | The chosen mode stays selected on other pages; the green preorder strip shows only in Preorder mode. | ☐ Pass ☐ Fail |

### 3. Cart, coupons and totals

*Verify the arithmetic with a calculator. Use sample data: Tomato ₹32, Apple ₹160. Society Green Valley (min order ₹150, delivery ₹20, free above ₹400).*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **CART-01** | P1 | Add to cart from a card | 1. Select Green Valley.<br>2. Tap Add on Tomato. | Button becomes a − 1 + stepper; cart badge shows 1; toast 'Added to cart'; on phones a green 'View cart' bar appears. | ☐ Pass ☐ Fail |
| **CART-02** | P1 | Stepper | 1. Press + twice, then − three times. | Quantity changes and badge updates each time; at 0 the Add button returns. | ☐ Pass ☐ Fail |
| **CART-03** | P1 | Cart page works without reload | 1. Open Cart.<br>2. Use +, − and Remove on lines. | Quantities, line totals and the order summary update immediately. | ☐ Pass ☐ Fail |
| **CART-04** | P1 | Totals with automatic offer | 1. Cart: 3 × Tomato + 2 × Apple. Society Green Valley. No coupon. | Subtotal ₹416, Discount ₹16 (Fruit festival 5% on apples), Delivery Free (₹400 or more after discount), Total ₹400. | ☐ Pass ☐ Fail |
| **CART-05** | P1 | Totals with coupon WELCOME10 | 1. Same cart. Apply WELCOME10. | Coupon replaces the automatic offer: Discount ₹41.60, Delivery ₹20, Total ₹394.40. | ☐ Pass ☐ Fail |
| **CART-06** | P1 | Invalid coupon | 1. Enter ABCD123. | Message 'This coupon code is not valid.' Cart total unchanged. | ☐ Pass ☐ Fail |
| **CART-07** | P1 | Coupon minimum order | 1. Cart worth less than ₹500, apply FRESH50.<br>2. Then make cart 4 × Apple (₹640) and apply FRESH50. | First: 'Add items worth ₹… more to use this coupon.' Second: accepted, Discount ₹50, Delivery Free, Total ₹590. | ☐ Pass ☐ Fail |
| **CART-08** | P1 | Remove coupon | 1. Click Remove on the applied coupon. | Coupon disappears; totals return to the automatic offer. | ☐ Pass ☐ Fail |
| **CART-09** | P1 | Society minimum order | 1. Cart: only 1 Tomato (₹32). | Warning 'Minimum order for Green Valley Residency is ₹150. Add ₹118 more.' and 'Proceed to checkout' is disabled. | ☐ Pass ☐ Fail |
| **CART-10** | P2 | Free delivery hint | 1. Cart worth about ₹300. | Shows 'Add ₹… more for free delivery'. | ☐ Pass ☐ Fail |
| **CART-11** | P2 | No society chosen | 1. Log out, clear society (new incognito window), add an item, open Cart. | Delivery shows 'Choose society' and an info message; no crash. | ☐ Pass ☐ Fail |
| **CART-12** | P2 | Maximum quantity | 1. Press + on one product more than 20 times (or try 25). | Quantity stops at 20 (setting 'Max qty per product'). | ☐ Pass ☐ Fail |
| **CART-13** | P1 | Stock limit in Fresh Now | 1. Admin: set a product stock to 3.<br>2. In Fresh Now mode try to add a 4th. | Toast 'Only 3 of … available today.' Quantity stays 3. | ☐ Pass ☐ Fail |
| **CART-14** | P2 | Preorder allows out-of-stock | 1. Set a product stock to 0, switch to Preorder mode, add it. | It can be added to the cart. | ☐ Pass ☐ Fail |
| **CART-15** | P1 | Guest cart is kept after login | 1. As a visitor add 2 items and apply a coupon.<br>2. Log in (or register). | Items (and coupon) are still in the cart after login. | ☐ Pass ☐ Fail |
| **CART-16** | P2 | Cart follows the account | 1. Add an item on desktop while logged in.<br>2. Open the site on a phone with the same account. | Same cart on both devices. | ☐ Pass ☐ Fail |
| **CART-17** | P1 | Item deactivated while in cart | 1. Put a product in the cart.<br>2. Admin deactivates it.<br>3. Refresh the cart. | Line shows 'No longer available. Please remove.'; it is excluded from totals. | ☐ Pass ☐ Fail |
| **CART-18** | P2 | Price change is applied | 1. Put Tomato in the cart.<br>2. Admin changes Tomato's price to ₹35.<br>3. Refresh the cart. | Cart shows the new price. (Prices always come from the server.) Change it back. | ☐ Pass ☐ Fail |
| **CART-19** | P3 | Empty cart | 1. Remove all items. | Friendly empty message with 'Start shopping'. | ☐ Pass ☐ Fail |

### 4. Registration, login, password

*Create accounts and test every validation. Use a mobile number you do not already have registered.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **AUTH-01** | P1 | Registration page | 1. Open /register. | Form shows name, mobile, email, passwords, society, flat, floor, building, street, PIN, landmark, instructions. Society list has only active societies. | ☐ Pass ☐ Fail |
| **AUTH-02** | P1 | Required-field validation | 1. Submit with one problem at a time: empty name; mobile 12345; mobile starting with 5; invalid email; password 'abc'; password with no number; passwords not matching; no society; no flat number; PIN 12345. | Each shows a clear message (e.g. 'Enter a valid 10-digit Indian mobile number.', 'Password must be at least 8 characters.', 'The two passwords do not match.'). Typed values stay in the form. | ☐ Pass ☐ Fail |
| **AUTH-03** | P1 | Successful registration | 1. Fill the form correctly (society Green Valley, flat C-12, floor 3, building Wing C) and submit. | You are logged in on the home page with 'Welcome to Healthy Fasal'. /account shows your name. Address page shows the address as Default. | ☐ Pass ☐ Fail |
| **AUTH-04** | P2 | Mobile number formats | 1. Register with +91 98765-43210 style input. | Accepted and stored as 10 digits. | ☐ Pass ☐ Fail |
| **AUTH-05** | P1 | Duplicate mobile or email | 1. Register again with the same mobile, then with the same email. | 'This mobile number is already registered. Please log in.' and the same for email. | ☐ Pass ☐ Fail |
| **AUTH-06** | P1 | Login with mobile and with email | 1. Log out. Log in with mobile + password. Log out. Log in with email + password. | Both work and land on /account. | ☐ Pass ☐ Fail |
| **AUTH-07** | P1 | Wrong password | 1. Log in with a wrong password. | 'Incorrect mobile/email or password.' The mobile field keeps its value; the cursor moves to password. | ☐ Pass ☐ Fail |
| **AUTH-08** | P2 | Show/hide password and empty submit | 1. Use the eye icon. Press Log in with empty fields. | Password toggles visible/hidden. Empty form is not sent. | ☐ Pass ☐ Fail |
| **AUTH-09** | P1 | Login throttling | 1. Enter a wrong password 5 times for one account, then the correct password. | Message 'Too many attempts. Please wait 15 minutes and try again.' even with the right password. (To reset quickly: in phpMyAdmin run DELETE FROM login_attempts;) | ☐ Pass ☐ Fail |
| **AUTH-10** | P1 | Roles cannot cross-login | 1. Admin mobile on /login.<br>2. Customer mobile on /admin/login.<br>3. Partner mobile on /login and on /admin/login. | All are refused with the same generic message. | ☐ Pass ☐ Fail |
| **AUTH-11** | P1 | Protected pages need login | 1. Logged out, open /account, /orders, /checkout. | Redirected to login with 'Please log in to continue.' After logging in you return to the page you wanted. | ☐ Pass ☐ Fail |
| **AUTH-12** | P1 | Logout | 1. Log out as customer, admin, partner. Then press the browser Back button and refresh. | Session ends; private pages ask for login again. | ☐ Pass ☐ Fail |
| **AUTH-13** | P2 | Already logged in | 1. While logged in open /login. | You are sent to your dashboard. | ☐ Pass ☐ Fail |
| **AUTH-14** | P1 | Change password | 1. Open Change password: wrong current password; weak new password; mismatching; then a correct change. | Errors for the first three; success on the last. Old password no longer works, new one does. | ☐ Pass ☐ Fail |
| **AUTH-15** | P1 | Deactivated customer cannot log in | 1. Admin deactivates a customer without open orders.<br>2. Customer tries to log in. | 'This account is switched off…'. If the customer was already logged in, their next click logs them out. | ☐ Pass ☐ Fail |

### 5. Email OTP at signup (OTP_REGISTRATION=1)

*Set OTP_REGISTRATION=1 and MAIL_DRIVER=smtp in .env. Use a real email you can open. These need real emails.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **OTP-01** | P1 | Email becomes required | 1. Open /register. | Email field is required with the hint 'We will email a 6-digit code to verify it.' The button says 'Continue – send me a code'. | ☐ Pass ☐ Fail |
| **OTP-02** | P1 | Code is emailed | 1. Complete the form and submit. | A 'Check your email' screen shows the email partly hidden (e.g. pr••••••••@gmail.com). An email 'Your Healthy Fasal code: 123456' arrives within a minute (check Spam). | ☐ Pass ☐ Fail |
| **OTP-03** | P1 | No account before verification | 1. Before entering the code, try to log in with that mobile. | Login fails because the account does not exist yet. | ☐ Pass ☐ Fail |
| **OTP-04** | P1 | Correct code creates account | 1. Type the 6 digits. | The form submits by itself after the 6th digit, the account is created, you are logged in, a default address exists. | ☐ Pass ☐ Fail |
| **OTP-05** | P1 | Wrong code | 1. Enter a wrong code 4 times, then the right code. | Each wrong try says 'That code is not correct. N tries left.' counting down. After 5 wrong tries even the right code is refused ('Too many wrong tries') until a new code is requested. | ☐ Pass ☐ Fail |
| **OTP-06** | P1 | Resend | 1. Look at 'Resend code'.<br>2. Wait for the countdown, press it. | Button is disabled with a 60 second countdown, then works. A new code arrives and the old code stops working. | ☐ Pass ☐ Fail |
| **OTP-07** | P2 | Expired code | 1. Wait more than 10 minutes, then enter the code. | Message says the code expired; request a new one. | ☐ Pass ☐ Fail |
| **OTP-08** | P2 | Used code cannot be reused | 1. After a successful signup, try the same code again. | Refused. | ☐ Pass ☐ Fail |
| **OTP-09** | P2 | Hourly limit | 1. Request 6 codes for the same email within an hour (waiting 60 s between). | 6th request: 'Too many codes requested. Please try again in an hour.' | ☐ Pass ☐ Fail |
| **OTP-10** | P2 | Change details link | 1. On the code screen press 'Change my details'. | Returns to the form. Leaving the code screen for more than 30 minutes asks you to fill the form again. | ☐ Pass ☐ Fail |
| **OTP-11** | P2 | Email already registered | 1. Sign up using an email that already has an account. | Refused before any code is sent. | ☐ Pass ☐ Fail |
| **OTP-12** | P1 | Mail failure is handled | 1. Temporarily put a wrong MAIL_PASSWORD in .env and try to sign up. | Friendly 'We could not send the email right now…'. No account created, no crash. Restore the password afterwards. | ☐ Pass ☐ Fail |

### 6. Forgot password by email code

*Test with a customer that has an email address.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **FRG-01** | P1 | Request a code | 1. Open /forgot, enter the registered mobile (or email). | Screen 'Enter the code' with the email partly hidden; email with a 6-digit code arrives. | ☐ Pass ☐ Fail |
| **FRG-02** | P1 | Unknown account looks the same | 1. Request a reset for 9999999999. | Identical screen (no 'account not found'); no email is sent. | ☐ Pass ☐ Fail |
| **FRG-03** | P2 | Account without email | 1. Request a reset for an account that has no email. | Information message to phone the shop; nothing is sent. | ☐ Pass ☐ Fail |
| **FRG-04** | P1 | Validation | 1. Wrong code; weak new password; mismatching passwords. | Clear errors: 'not correct, N tries left', 'at least 8 characters', 'passwords do not match'. | ☐ Pass ☐ Fail |
| **FRG-05** | P1 | Reset succeeds | 1. Correct code + a new strong password. | Back to login with 'Password updated'. Old password fails; new password works. | ☐ Pass ☐ Fail |
| **FRG-06** | P2 | Code reuse and blocking | 1. Reuse the same code. Then submit 6 wrong codes. | Reused code refused. After repeated wrong codes: 'Too many attempts. Please wait 15 minutes…'. | ☐ Pass ☐ Fail |
| **FRG-07** | P3 | Resend | 1. Press Resend after the 60 second countdown. | A new code arrives; the old one no longer works. | ☐ Pass ☐ Fail |

### 7. Customer account and addresses

*Logged in as a customer.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **ACC-01** | P1 | Profile | 1. Open /account.<br>2. Change name and email, save.<br>3. Try an invalid email and an email used by another account. | Saved with a success message. Invalid and duplicate emails are refused. Mobile is read-only. | ☐ Pass ☐ Fail |
| **ACC-02** | P2 | Statistics | 1. Compare Orders and Spent on /account with the order list. | Numbers match (cancelled orders are not counted as spent). | ☐ Pass ☐ Fail |
| **ACC-03** | P1 | Add an address | 1. Open Addresses and add a second one (Office) with all fields. | It appears in the list; the first address stays Default. | ☐ Pass ☐ Fail |
| **ACC-04** | P1 | Edit, default, delete | 1. Edit an address.<br>2. Make the second one default.<br>3. Delete one (confirm). | Changes appear immediately. Deleting the default moves 'Default' to another address. | ☐ Pass ☐ Fail |
| **ACC-05** | P1 | Address validation | 1. Save with no society; no flat number; PIN 123. | Each is refused with a message. | ☐ Pass ☐ Fail |
| **ACC-06** | P2 | Inactive society address | 1. Admin deactivates the society of one address.<br>2. Open Addresses and Checkout. | Address is marked 'Not serviceable now' and cannot be selected at checkout. Reactivate afterwards. | ☐ Pass ☐ Fail |
| **ACC-07** | P3 | Return to checkout | 1. From checkout choose 'Add or edit address', save. | You come back to checkout. | ☐ Pass ☐ Fail |

### 8. Checkout, orders and cancellation

*The most important customer flow. Logged in as a customer with items worth more than the society minimum.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **CHK-01** | P1 | Checkout needs a cart | 1. Open /checkout with an empty cart. | Redirected to the cart with 'Your cart is empty.' | ☐ Pass ☐ Fail |
| **CHK-02** | P1 | Checkout page | 1. Add items and open Checkout. | Default address selected, Fresh Now / Preorder cards, slots load, payment methods, review panel totals equal the cart. | ☐ Pass ☐ Fail |
| **CHK-03** | P1 | Fresh Now order | 1. Make sure a slot is open today (Admin > Delivery slots: a slot starting at least 1 hour from now).<br>2. Choose it, payment Cash on Delivery, Place order. | Success page 'Order placed!' with order number HF…, items, total, payment method, delivery date and slot, address. | ☐ Pass ☐ Fail |
| **CHK-04** | P1 | Slot closes before it starts | 1. Edit a slot to start in 10 minutes (lead time is 30). | That slot shows 'Slot closed for today' and cannot be selected. | ☐ Pass ☐ Fail |
| **CHK-05** | P1 | No open slot today | 1. Make all slots closed (past times). | Message suggests Preorder for Tomorrow; Place order button is disabled. | ☐ Pass ☐ Fail |
| **CHK-06** | P1 | Preorder dates | 1. Choose Preorder for Tomorrow.<br>2. Look at the date list. | Three eligible dates are offered; the earliest is tomorrow when you order before the society's cut-off. | ☐ Pass ☐ Fail |
| **CHK-07** | P1 | Cut-off rule | 1. Admin: set the society's preorder cut-off to a time already passed today.<br>2. Reload checkout in Preorder mode.<br>3. Then set it to a time later today. | After the cut-off the earliest date becomes the day after tomorrow; before it, tomorrow. | ☐ Pass ☐ Fail |
| **CHK-08** | P1 | Off-days and holidays are skipped | 1. Admin > Settings: tick no delivery for the weekday of tomorrow (or add tomorrow as a holiday). | That date is not offered in the list. Undo afterwards. | ☐ Pass ☐ Fail |
| **CHK-09** | P1 | Preorder success | 1. Place a preorder. | 'Preorder placed!'. Order page shows 'Pending Confirmation' and the 'Preorder for Tomorrow' tag with the chosen date and slot. | ☐ Pass ☐ Fail |
| **CHK-10** | P2 | Slot capacity | 1. Admin: set a slot's max orders to 1 for that society.<br>2. Customer A orders it; Customer B opens checkout. | For B the slot shows 'Slot full' and is disabled. | ☐ Pass ☐ Fail |
| **CHK-11** | P1 | Payment methods follow settings | 1. Admin > Settings: turn off UPI. | Checkout shows only Cash on Delivery. Turn UPI back on. | ☐ Pass ☐ Fail |
| **CHK-12** | P2 | Order note | 1. Add a note (e.g. 'Ripe tomatoes please') and place the order. | Admin sees the note on the order page. | ☐ Pass ☐ Fail |
| **CHK-13** | P1 | Double tap protection | 1. Tap Place order twice quickly.<br>2. After success press the browser Back button and Place order again. | Only one order is created (check Orders). The button shows 'Placing order…'. The repeated request returns the same order or is refused. | ☐ Pass ☐ Fail |
| **CHK-14** | P2 | Tampered total is rejected | 1. On checkout open DevTools, change the hidden field expected_total_p to 5, then Place order. | Message 'Prices or offers changed… review the updated total'. No order is created. | ☐ Pass ☐ Fail |
| **CHK-15** | P1 | Coupon limit per customer | 1. Place an order using WELCOME10.<br>2. Try WELCOME10 again in a new cart. | 'You have already used this coupon.' | ☐ Pass ☐ Fail |
| **CHK-16** | P2 | Stock changes during checkout | 1. Open Checkout with 5 apples.<br>2. Admin sets apple stock to 2.<br>3. Press Place order. | Message 'Only 2 of Apple left…' and no order is created; the cart is still there. | ☐ Pass ☐ Fail |
| **CHK-17** | P2 | Minimum order enforced at the end | 1. Open checkout in tab 1, reduce the cart below the minimum in tab 2, press Place order in tab 1. | Refused with the society's minimum order message. | ☐ Pass ☐ Fail |
| **CHK-18** | P2 | Orders per hour limit | 1. Admin > Settings: set 'Max orders per customer / hour' to 2.<br>2. Place 3 orders. | Third is refused with a clear message. Restore to 10. | ☐ Pass ☐ Fail |
| **CHK-19** | P1 | Address snapshot | 1. After ordering, edit the address (change flat number).<br>2. Open the old order. | The order still shows the old flat number. | ☐ Pass ☐ Fail |
| **CHK-20** | P1 | Order history | 1. Open /orders: Current and Past tabs. | Each order card shows number, placed time, delivery date and slot, status badge, total, 'Preorder' tag. Paging works when there are many. | ☐ Pass ☐ Fail |
| **CHK-21** | P1 | Order detail and invoice | 1. Open an order, then 'Order summary / invoice'. | Timeline with 7 steps, items, totals, payment status, address. Invoice page prints and says it is not a GST invoice. | ☐ Pass ☐ Fail |
| **CHK-22** | P1 | Live status update | 1. Keep an order open on the customer phone.<br>2. Admin changes its status. | Within about 20 seconds the status badge changes (the timeline updates on refresh). | ☐ Pass ☐ Fail |
| **CHK-23** | P1 | Customer cancels a pending order | 1. Open a pending order and press Cancel order. | Status Cancelled with reason. In Admin the product's reserved quantity drops back (stock released). | ☐ Pass ☐ Fail |
| **CHK-24** | P1 | Cancel window | 1. Admin: set 'Customer cancel window' to 0.<br>2. Admin confirms a Fresh Now order. | Customer no longer sees the Cancel button for the confirmed order. Restore to 30 afterwards. | ☐ Pass ☐ Fail |
| **CHK-25** | P1 | Preorder cancel deadline | 1. Place a preorder for tomorrow (cancel button visible).<br>2. Admin sets the society cut-off to a time earlier than now. | Cancel button disappears (deadline = cut-off time on the day before delivery). | ☐ Pass ☐ Fail |
| **CHK-26** | P1 | Other customer's order | 1. Copy an order URL from customer A and open it as customer B. | 404 page; no data shown. | ☐ Pass ☐ Fail |

### 9. Admin panel

*Log in at /admin/login with 9000000001. Use a desktop browser and also check the menu on a phone.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **ADM-01** | P1 | Admin login and menu | 1. Log in. Check the sidebar; on a phone use the ☰ button. | Dashboard opens. Sidebar: Dashboard, Orders, Preorders, Reports, Products, Categories, Offers, Customers, Delivery partners, Societies, Delivery slots, Settings. | ☐ Pass ☐ Fail |
| **ADM-02** | P1 | Dashboard numbers | 1. With known data compare each card with the Orders list: Today's Orders, Pending, Active Preorders, Delivered Today, Today's Sales (booked), Outstanding Collections, Active Customers, Active Partners. | Every card matches the real orders. Cancelled orders are excluded from sales. | ☐ Pass ☐ Fail |
| **ADM-03** | P1 | Dashboard charts | 1. Scroll the dashboard; open browser console (F12). | All 8 charts draw (daily sales, deliveries, monthly, society, products, categories, cash vs UPI, preorder demand). No console errors. With no data the page still opens. | ☐ Pass ☐ Fail |
| **ADM-04** | P2 | Low-stock panel and alert | 1. Set a product's stock low.<br>2. Confirm an order without a partner. | Product shows in 'Low stock'. A yellow alert says orders are ready for a delivery partner. | ☐ Pass ☐ Fail |
| **ADM-05** | P1 | Products list | 1. Search by name/SKU; filter by category; filter Low stock and Inactive. | Results correct; stock column shows sellable, on hand and reserved. | ☐ Pass ☐ Fail |
| **ADM-06** | P1 | Add a product with image | 1. Products > Add product: Beetroot, SKU VEG-BEE, ₹30, original ₹36, 500 g, stock 40, upload a JPG. | Saved. Image preview appears. Product shows on the storefront within seconds with the image and 17% OFF. | ☐ Pass ☐ Fail |
| **ADM-07** | P1 | Product validation | 1. Try: empty name; SKU with spaces; duplicate SKU; price 0; original price lower than price; negative cost. | Each is refused with a clear message; nothing is saved. | ☐ Pass ☐ Fail |
| **ADM-08** | P1 | Image upload checks | 1. Upload: a PNG, a WebP, a file over 4 MB, a text file renamed .jpg, a PHP file renamed .png, an image smaller than 100 px. | PNG and WebP accepted. All others refused with a message. Uploaded file is saved under public/uploads/products with a random name. | ☐ Pass ☐ Fail |
| **ADM-09** | P1 | Edit price and flags | 1. Change Tomato price; tick Featured/Bestseller/Seasonal on another product. | Storefront reflects the new price and the sections within seconds. | ☐ Pass ☐ Fail |
| **ADM-10** | P1 | Stock adjustment and history | 1. Add +20 'Purchase', then −5 'Adjustment', then try −500. | Stock changes; −500 is refused ('Stock cannot go below zero'). Inventory history lists each change with who, when and note. | ☐ Pass ☐ Fail |
| **ADM-11** | P2 | Categories | 1. Add a category with image; edit order; hide and show. | Works; hidden category disappears from the storefront. | ☐ Pass ☐ Fail |
| **ADM-12** | P1 | Orders list and filters | 1. Use status chips (counts), date range, society, partner, payment method, type, and search by order number, name and mobile. | Each filter returns the right orders; pagination works; on a phone the table becomes cards. | ☐ Pass ☐ Fail |
| **ADM-13** | P1 | Order detail | 1. Open an order. | Shows items, totals, customer and delivery address, slot, payment, full history and delivery events. | ☐ Pass ☐ Fail |
| **ADM-14** | P1 | Confirm, prepare, ready | 1. Move a pending order Confirmed > Preparing > Ready. | Each change succeeds and appears in the history with your name; the customer's page reflects it. | ☐ Pass ☐ Fail |
| **ADM-15** | P1 | Cancel needs a reason | 1. Try to cancel with an empty reason, then with a reason. | Empty is refused ('Please enter a reason'); with reason the order is cancelled and reserved stock is released. | ☐ Pass ☐ Fail |
| **ADM-16** | P1 | Only valid next steps are offered | 1. Open a Delivered order and a Cancelled order. | No status change options are offered. | ☐ Pass ☐ Fail |
| **ADM-17** | P1 | Assign a delivery partner | 1. Try to assign a pending (unconfirmed) order, then a confirmed one. | Pending: 'Confirm the order before assigning…' (or not offered). Confirmed: only partners serving that society are listed; assignment succeeds and shows the partner name. | ☐ Pass ☐ Fail |
| **ADM-18** | P2 | No partner for a society | 1. Use a society with no assigned partner. | Warning 'No active partner serves…' with a link to partners. | ☐ Pass ☐ Fail |
| **ADM-19** | P2 | Reassign | 1. Assign a different partner to the same order. | Works; the new partner sees it, the old one no longer does. | ☐ Pass ☐ Fail |
| **ADM-20** | P1 | Packing slip | 1. Press Packing slip, then Print. | Printable page with items, quantities, address and 'Collect' amount. | ☐ Pass ☐ Fail |
| **ADM-21** | P1 | Preorders page | 1. Open Preorders, pick a date with orders. | Date chips show counts. Purchase list shows quantity needed, stock and 'To buy' (needed minus sellable stock) - check against the orders. | ☐ Pass ☐ Fail |
| **ADM-22** | P1 | Bulk confirm | 1. Tick two pending preorders, press Confirm selected. | They become Confirmed; already-confirmed or orders with unavailable items are skipped with a message. | ☐ Pass ☐ Fail |
| **ADM-23** | P1 | Unavailable item workflow | 1. On a pending preorder mark one item Unavailable with a reason.<br>2. Check the customer's order page.<br>3. Press 'Remove unavailable items & recalculate' and confirm. | Line is highlighted; customer sees it struck through as 'Unavailable'. After removal the total, discount and delivery fee are recalculated and a history note is written. If every item is unavailable the order is cancelled. | ☐ Pass ☐ Fail |
| **ADM-24** | P2 | Preorder CSV | 1. Export CSV and open it in Excel. | Contains the purchase list and order list; opens correctly. | ☐ Pass ☐ Fail |
| **ADM-25** | P1 | Add a delivery partner | 1. Add 'Mohan Test', mobile 9333333333, tick two societies. | Saved; a temporary password is shown once. Invalid or duplicate mobile is refused. | ☐ Pass ☐ Fail |
| **ADM-26** | P1 | Partner deactivate safeguard | 1. Deactivate a partner who has an Assigned or Out-for-delivery order, then one without. | First is refused ('still has N active deliveries'). Second works and that partner can no longer log in. | ☐ Pass ☐ Fail |
| **ADM-27** | P2 | Reset partner password | 1. Press Reset password. | New temporary password shown once; the partner is forced to change it at next login. | ☐ Pass ☐ Fail |
| **ADM-28** | P2 | Partner detail | 1. Open a partner. | Shows delivered/pending/problem counts, collections by cash/UPI, last 20 orders. | ☐ Pass ☐ Fail |
| **ADM-29** | P1 | Customers | 1. Search by name, mobile, email; filter by society; open a customer; try Deactivate with an open order. | Search works. Detail shows addresses and orders. Deactivating with open orders is refused; without open orders it works and can be undone. | ☐ Pass ☐ Fail |
| **ADM-30** | P1 | Create offers | 1. Create: a percentage offer with coupon and max discount; a fixed offer; an automatic offer (no code) on one category; a society-specific offer; one with a usage limit. | All save and show in the list as Live. | ☐ Pass ☐ Fail |
| **ADM-31** | P1 | Offer validation | 1. End date before start; percent over 100; duplicate coupon code; 'selected products' with none selected. | Each is refused with a message. | ☐ Pass ☐ Fail |
| **ADM-32** | P1 | Offers work on the storefront | 1. Test each offer in a cart. Also test one scheduled for the future, one expired and one turned off. | Live offers apply exactly as configured (minimum, cap, scope, society, limits). Future, expired and turned-off offers never apply and are not shown on /offers. | ☐ Pass ☐ Fail |
| **ADM-33** | P1 | Add a society | 1. Add 'Test Towers' (fee, minimum, free-above, cut-off) and assign a partner. | Appears in the registration list and the header selector. Deactivating it removes it from those lists and flags existing addresses. | ☐ Pass ☐ Fail |
| **ADM-34** | P1 | Society rules take effect | 1. Change a society's delivery fee, minimum order and free-above amount. | Cart and checkout use the new values immediately. | ☐ Pass ☐ Fail |
| **ADM-35** | P1 | Delivery slots | 1. Add a default slot; add a slot for one society only; edit; delete one. | A society with its own slots shows only those; others show the defaults. Invalid times (end before start) are refused. Old orders keep their slot name after deletion. | ☐ Pass ☐ Fail |
| **ADM-36** | P1 | Settings: each switch has an effect | 1. Change each setting and check its effect: payment methods, no-delivery weekdays, preorders on/off, preorder dates offered, slot lead minutes, cancel window, max qty, orders per hour, reserve stock for preorders, partial payments, unpaid delivery, show partner contact. | Every setting changes behaviour as described in README section 11.2. Preorders off: Preorder links disappear and preorders are refused. 'Reserve stock for preorders' on: a new preorder increases the product's reserved quantity. | ☐ Pass ☐ Fail |
| **ADM-37** | P2 | Holidays | 1. Add and remove a holiday. | Holiday date disappears from preorder dates while it exists. | ☐ Pass ☐ Fail |
| **ADM-38** | P2 | Testimonials | 1. Add a testimonial, check Home, delete it. | Section appears with the text while one exists and hides when none remain. | ☐ Pass ☐ Fail |
| **ADM-39** | P1 | Email & OTP panel | 1. Open Settings > Email & OTP.<br>2. Press Send test. | Shows Driver smtp, Status Ready, OTP at signup On. Test email arrives. With wrong settings a clear error is shown. | ☐ Pass ☐ Fail |
| **ADM-40** | P1 | Reports match reality | 1. Choose a date range and society; compare Gross, Discounts, Refunds, Net sales, Collected, Delivered-but-unpaid with a manual calculation from the orders. | Numbers match. Cancelled orders are excluded. Net = Gross − Discounts − Refunds. | ☐ Pass ☐ Fail |
| **ADM-41** | P2 | Report tables | 1. Check product-wise, society-wise and partner-wise tables. | Totals agree with the orders. | ☐ Pass ☐ Fail |
| **ADM-42** | P1 | Reconciliation | 1. Check 'Payment collection reconciliation' per partner, per day, cash vs UPI; check the 'delivered but not fully paid or disputed' list. | Amounts equal what partners actually recorded; the discrepancy list shows only real problems. | ☐ Pass ☐ Fail |
| **ADM-43** | P2 | CSV exports | 1. Export summary, products, societies, collections, partners. Create a customer named =1+1 and export again. | Files open in Excel. The cell with =1+1 is shown as text (an apostrophe prefix), not calculated. | ☐ Pass ☐ Fail |
| **ADM-44** | P2 | Dashboard refreshes | 1. Place an order and look at the dashboard after about a minute. | Figures update (they are cached up to 20-60 seconds). | ☐ Pass ☐ Fail |
| **ADM-45** | P2 | Admin change password and audit | 1. Change the admin password. Then run in phpMyAdmin: SELECT action, COUNT(*) FROM audit_logs GROUP BY action; | Password changes. The audit table lists logins, status changes, payments, product/offer changes. | ☐ Pass ☐ Fail |

### 10. Delivery partner (use a phone)

*Log in at /partner/login. Create a fresh order and have the admin confirm and assign it to this partner first.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **PRT-01** | P1 | First login with temporary password | 1. Admin creates a partner. Log in with the temporary password. | You are forced to /change-password; a weak password is refused; after changing you reach the dashboard. | ☐ Pass ☐ Fail |
| **PRT-02** | P1 | Dashboard tiles | 1. Check New assigned, Today's deliveries, Completed today, To collect. | Counts match the partner's real orders. | ☐ Pass ☐ Fail |
| **PRT-03** | P1 | Only own orders | 1. Log in as the other partner and compare lists. Copy an order URL from partner A and open it as partner B. | Each partner sees only their own orders. Another partner's order gives 404. | ☐ Pass ☐ Fail |
| **PRT-04** | P1 | Order card | 1. Read an order card and open the order. | Shows order number, customer, society, building, flat, floor, landmark, instructions, items, slot, payment method and the amount to collect. | ☐ Pass ☐ Fail |
| **PRT-05** | P1 | Call and Maps buttons | 1. Tap Call and Map on a phone. | Call opens the dialer with the customer's number. Map opens Google Maps with the society address. | ☐ Pass ☐ Fail |
| **PRT-06** | P2 | Accept assignment | 1. Press 'Accept this delivery'. | The NEW tag disappears. | ☐ Pass ☐ Fail |
| **PRT-07** | P1 | Delivery steps | 1. Press: Start preparing (if shown), Picked up from shop, Out for delivery. | Statuses change step by step; the admin order page and the customer page show them. Out for delivery is not available before pickup. | ☐ Pass ☐ Fail |
| **PRT-08** | P1 | Payment validation | 1. On an out-for-delivery order try: amount 1; amount larger than due; text 'abc'; UPI without the verification tick; UPI reference with symbols. | Each is refused with a clear message. (Partial amounts are refused when partial payments are off.) | ☐ Pass ☐ Fail |
| **PRT-09** | P1 | Record payment correctly | 1. Select UPI or Cash, keep the full amount, tick 'I verified', confirm. | 'Payment recorded in full.' appears and the 'Mark delivered' button is enabled. Admin shows the collection with your name. | ☐ Pass ☐ Fail |
| **PRT-10** | P2 | Partial payments setting | 1. Admin turns partial payments on.<br>2. Collect part of the amount, then the rest. | Status goes Partial then Paid; total collected equals the order total. | ☐ Pass ☐ Fail |
| **PRT-11** | P1 | Delivered requires payment | 1. Try 'Mark delivered' before recording payment. | Not possible (message/disabled). After payment it works; the order moves to Completed and the customer sees Delivered. | ☐ Pass ☐ Fail |
| **PRT-12** | P1 | Stock after delivery | 1. Check the product's stock before and after delivery in Admin. | On-hand stock decreases by the delivered quantity and reserved decreases too. | ☐ Pass ☐ Fail |
| **PRT-13** | P2 | Unpaid delivery rule | 1. Admin enables 'Allow delivery without payment'.<br>2. Deliver an unpaid order. | A reason is required; it is stored and shown to the admin on the order. | ☐ Pass ☐ Fail |
| **PRT-14** | P1 | Report a problem | 1. For separate orders report: customer unavailable, address problem, delivery failed, item shortage (each with a note). | First three: status 'Delivery Failed'. Item shortage: 'Item Unavailable'. A note is required. Admin sees the event and can reassign or cancel. | ☐ Pass ☐ Fail |
| **PRT-15** | P3 | Delivery note | 1. Save a note such as 'Left with security'. | Note appears in the order's history list. | ☐ Pass ☐ Fail |
| **PRT-16** | P2 | Completed list | 1. Open Completed. | Delivered orders with amounts collected, paged. | ☐ Pass ☐ Fail |
| **PRT-17** | P1 | Phone usability | 1. Use the whole flow with one hand on a 360 px phone. | Buttons are large, no sideways scrolling, buttons disable after a tap (no double submissions). | ☐ Pass ☐ Fail |

### 11. End-to-end stories (run each with three people or three browsers)

*These combine all roles. Tick the story only when every step behaves as described.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **E2E-01** | P1 | Story 1: Fresh Now order, delivered and paid | 1. Customer: add 3 Tomato + 2 Apple, WELCOME10, Cash, place order (Fresh Now).<br>2. Admin: confirm, prepare, ready, assign partner.<br>3. Partner: accept, picked up, out for delivery, collect cash, deliver.<br>4. Admin: open Reports, Reconciliation, Dashboard.<br>5. Customer: open order and invoice. | Customer total ₹394.40 throughout. Order ends Delivered/Paid. Stock reduced by 3 tomatoes and 2 apples. Report shows gross ₹416+₹20 delivery, discount ₹41.60, net ₹394.40; collections list shows the partner's cash entry; dashboard Delivered Today +1. | ☐ Pass ☐ Fail |
| **E2E-02** | P1 | Story 2: Preorder with an unavailable item | 1. Customer: preorder 3 items for tomorrow (UPI).<br>2. Admin: see it on Preorders, mark one item unavailable (reason), phone customer (simulate), remove the item, confirm, assign.<br>3. Partner: deliver and collect UPI.<br>4. Customer: check the final total. | Purchase list matched the order. After removal the total, discount and delivery were recalculated, the customer saw the unavailable line struck through, and the final collected amount equals the new total. | ☐ Pass ☐ Fail |
| **E2E-03** | P1 | Story 3: Cancellation releases stock | 1. Note a product's reserved quantity.<br>2. Customer places an order; reserved rises.<br>3. Customer (or admin with a reason) cancels it. | Reserved returns to the original value; the order shows Cancelled with the reason; reports exclude it. | ☐ Pass ☐ Fail |
| **E2E-04** | P1 | Story 4: Failed delivery and reassign | 1. Partner A reports 'Customer not available' on an out-for-delivery order.<br>2. Admin reopens it (Ready) and assigns Partner B.<br>3. Partner B delivers and collects. | Order moves Failed > Ready > Assigned > Delivered; Partner A can no longer see it; collection is recorded under Partner B. | ☐ Pass ☐ Fail |
| **E2E-05** | P1 | Story 5: Last item race (two phones) | 1. Admin: set a product's stock so exactly 1 is sellable.<br>2. Customer A and Customer B both add it to the cart.<br>3. Both press Place order at the same moment. | Exactly one order succeeds. The other sees 'out of stock today… place it as a preorder'. Reserved quantity is 1, never 2. | ☐ Pass ☐ Fail |
| **E2E-06** | P1 | Story 6: Refund and dispute | 1. Complete a paid delivery.<br>2. Admin marks payment Disputed with a reason, resolves it, then records a partial refund. | Each change is in the payment history with reason and your name; refund total appears in Reports; collected amount itself is unchanged. | ☐ Pass ☐ Fail |
| **E2E-07** | P2 | Story 7: New society onboarding | 1. Admin adds a society, a partner, a slot.<br>2. A new customer registers in that society and orders; the partner delivers. | Everything works without any code change. | ☐ Pass ☐ Fail |

### 12. Security checks (manual)

*Do these on the final server too.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **SEC-01** | P1 | Page access by role | 1. Try these URLs logged out, as customer, as partner, as admin: /admin, /admin/orders, /admin/reports, /partner, /account, /checkout. | Logged out: redirected to the right login. Wrong role: 403 'You do not have access'. Right role: opens. | ☐ Pass ☐ Fail |
| **SEC-02** | P1 | Other people's data | 1. Change numbers in URLs: /orders/ID, /orders/ID/invoice, /partner/orders/ID. | Someone else's record always returns 404; nothing is shown. | ☐ Pass ☐ Fail |
| **SEC-03** | P1 | Scripts typed into fields | 1. Use <script>alert(1)</script> as customer name, address, product name, order note, society name. | It appears as plain text everywhere (storefront, admin, partner, invoice, packing slip). No popups. | ☐ Pass ☐ Fail |
| **SEC-04** | P1 | Injection strings | 1. Put ' OR 1=1 -- in search, the login field and the coupon box. | No errors and no unexpected results. | ☐ Pass ☐ Fail |
| **SEC-05** | P1 | Private files blocked | 1. Open /.env, /app/helpers/Db.php, /database/schema.sql, /storage/logs/php-error.log, /tests/run.php, /bin/setup.php, /uploads/ (directory). | All blocked (403/404). No directory listing. | ☐ Pass ☐ Fail |
| **SEC-06** | P1 | Uploaded code is not executed | 1. Try to upload a .php file (see ADM image checks). If any file ever lands in /uploads, open its URL. | PHP never runs from the uploads folder. | ☐ Pass ☐ Fail |
| **SEC-07** | P1 | Session expiry protection | 1. Open an admin page in two tabs. Log out in tab 1. Submit a form in tab 2. | 'Your session expired…' page (419). The action is not performed. | ☐ Pass ☐ Fail |
| **SEC-08** | P2 | Cookies | 1. DevTools > Application > Cookies. | Session cookie is HttpOnly and SameSite=Lax; on the live HTTPS site also Secure. | ☐ Pass ☐ Fail |
| **SEC-09** | P1 | No technical details in errors | 1. Cause a 404, a 403 and an expired session. | Friendly pages only. No file paths, SQL or stack traces (with APP_ENV=production). | ☐ Pass ☐ Fail |
| **SEC-10** | P1 | Passwords are hashed | 1. In phpMyAdmin open table users. | password_hash begins with $2y$… for every user; no readable passwords. | ☐ Pass ☐ Fail |
| **SEC-11** | P1 | Price and quantity tampering | 1. With DevTools Network replay 'cart/set' with qty -5, 0 and 99999, and try changing prices in requests. | Quantity is limited; prices always come from the server. | ☐ Pass ☐ Fail |
| **SEC-12** | P2 | Disabled account is cut off | 1. Log in as a customer in one browser. Deactivate that customer in Admin. Click anything in the customer browser. | Customer is logged out at the next click. | ☐ Pass ☐ Fail |
| **SEC-13** | P1 | HTTPS (live server) | 1. Open http://yourdomain.com. | Redirects to https:// with a valid certificate and no mixed-content warnings. | ☐ Pass ☐ Fail |

### 13. Phones, browsers, accessibility and speed

*Quality of experience.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **UX-01** | P1 | Screen widths | 1. Check Home, Shop, Cart, Checkout, Login, Orders, Admin dashboard, Partner dashboard at 360, 390, 430, 768, 1024 and 1440 px. | No horizontal scrolling anywhere; text readable; nothing overlaps. | ☐ Pass ☐ Fail |
| **UX-02** | P1 | Touch targets | 1. Use the phone one-handed: stepper, Add, bottom bar, Place order. | Buttons are easy to tap (about 44 px); no mis-taps. | ☐ Pass ☐ Fail |
| **UX-03** | P1 | Real browsers | 1. Test the main customer flow on Chrome, Edge, Firefox, Safari on iPhone, Samsung Internet. | Same behaviour everywhere; fonts and icons show. | ☐ Pass ☐ Fail |
| **UX-04** | P2 | Keyboard use | 1. On desktop use only Tab, Shift+Tab, Enter on Login and Checkout. | Focus outline always visible; forms can be completed. | ☐ Pass ☐ Fail |
| **UX-05** | P3 | Zoom and rotation | 1. Zoom 200%; rotate a phone to landscape. | Pages stay usable. | ☐ Pass ☐ Fail |
| **UX-06** | P2 | Slow network | 1. DevTools > Network > Fast 3G; browse, add to cart, check out. | Loading states appear; no double orders; pages usable. | ☐ Pass ☐ Fail |
| **UX-07** | P2 | Network failure | 1. Switch to Offline, press Add to cart. | A message appears (not a frozen button); working again when back online. | ☐ Pass ☐ Fail |
| **UX-08** | P2 | Printing | 1. Print the invoice and the packing slip. | Clean one-page print without menus. | ☐ Pass ☐ Fail |
| **UX-09** | P3 | Accessibility basics | 1. Check images have alt text, form fields have labels, colours readable. | No unlabeled fields; text contrast is readable. | ☐ Pass ☐ Fail |
| **UX-10** | P2 | Speed check | 1. Run Chrome Lighthouse (mobile) on Home and Shop. | Performance 70+ on a normal connection and Accessibility 90+ (note scores in the Notes column). | ☐ Pass ☐ Fail |

### 14. Data integrity (run these queries in phpMyAdmin after testing)

*Each query should return 0 rows. Paste each into SQL tab on the healthy_fasal database.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **DB-01** | P1 | Reserved stock matches open orders | 1. Run: SELECT p.id,p.name,p.reserved_qty,COALESCE(x.r,0) AS from_orders FROM products p LEFT JOIN (SELECT oi.product_id, SUM(oi.reserved_qty) r FROM order_items oi JOIN orders o ON o.id=oi.order_id WHERE o.status NOT IN ('delivered','cancelled') GROUP BY oi.product_id) x ON x.product_id=p.id WHERE p.reserved_qty<>COALESCE(x.r,0); | 0 rows. | ☐ Pass ☐ Fail |
| **DB-02** | P1 | No negative stock | 1. Run: SELECT id,name,stock_qty,reserved_qty FROM products WHERE stock_qty<0 OR reserved_qty<0; | 0 rows. | ☐ Pass ☐ Fail |
| **DB-03** | P1 | Order totals add up | 1. Run: SELECT id,order_number FROM orders WHERE ROUND(subtotal-discount+delivery_fee,2)<>total; | 0 rows. | ☐ Pass ☐ Fail |
| **DB-04** | P1 | Collected money matches records | 1. Run: SELECT o.id,o.order_number FROM orders o LEFT JOIN (SELECT order_id,SUM(amount) a FROM payment_collections GROUP BY order_id) c ON c.order_id=o.id WHERE o.amount_collected<>COALESCE(c.a,0); | 0 rows. | ☐ Pass ☐ Fail |
| **DB-05** | P1 | Delivered but unpaid always has a reason | 1. Run: SELECT order_number FROM orders WHERE status='delivered' AND payment_status IN ('pending','partial') AND unpaid_delivery_reason IS NULL; | 0 rows. | ☐ Pass ☐ Fail |
| **DB-06** | P2 | Items add up to the subtotal | 1. Run: SELECT o.id,o.order_number FROM orders o JOIN (SELECT order_id,SUM(line_total) s FROM order_items WHERE availability='ok' GROUP BY order_id) i ON i.order_id=o.id WHERE ROUND(o.subtotal,2)<>ROUND(i.s,2); | 0 rows (rows are expected only for orders that still have items marked unavailable but not yet removed). | ☐ Pass ☐ Fail |
| **DB-07** | P2 | Every status change is logged | 1. Run: SELECT o.id FROM orders o LEFT JOIN order_status_history h ON h.order_id=o.id WHERE h.id IS NULL; | Every order has at least one history row. | ☐ Pass ☐ Fail |

### 15. Simultaneous use

*Use two or three devices at the same time.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **CONC-01** | P1 | Two customers, same coupon limit | 1. Create an offer with total usage limit 1.<br>2. Two customers apply it and place orders at the same moment. | Only one order gets the discount; the other is told the offer has reached its limit. | ☐ Pass ☐ Fail |
| **CONC-02** | P1 | Double submit everywhere | 1. Double-click: Place order, Record payment, Mark delivered, Save product. | Only one action happens each time. | ☐ Pass ☐ Fail |
| **CONC-03** | P2 | Two admins, one order | 1. Open the same order in two admin browsers.<br>2. Change its status in both. | The second change is either applied in a valid order or refused with a clear message; no corrupted state. | ☐ Pass ☐ Fail |
| **CONC-04** | P2 | Admin and partner together | 1. Admin cancels an order while the partner is about to collect payment. | Partner gets a clear message; no payment is recorded on a cancelled order. | ☐ Pass ☐ Fail |

### 16. Final handover and live-server checklist

*Do on the real server just before giving it to the client.*

| ID | P | Test | Steps | Expected result | Result |
|---|---|---|---|---|---|
| **DEP-01** | P1 | Clean production install | 1. Install on the live server with --env=production and WITHOUT --seed.<br>2. Create the real admin: php bin/create-admin.php. | No sample accounts exist (check table users). Admin can log in. | ☐ Pass ☐ Fail |
| **DEP-02** | P1 | Settings for production | 1. Open .env. | APP_ENV=production, APP_DEBUG=0, strong DB password, MAIL_DRIVER=smtp, APP_KEY set to its own random value, OTP_REGISTRATION as the client wants. | ☐ Pass ☐ Fail |
| **DEP-03** | P1 | Remove helper files | 1. Delete public/check-gd.php; make sure no test or backup files (.sql, .zip) are in public/. | Files gone; opening them gives 404. | ☐ Pass ☐ Fail |
| **DEP-04** | P1 | Real data entered | 1. Enter the client's real logo, societies, slots, products, prices, stock and photos; remove sample items. | Storefront shows only real business data. | ☐ Pass ☐ Fail |
| **DEP-05** | P1 | Live email test | 1. Sign up and run forgot-password with Gmail, Outlook and Yahoo addresses. | Codes arrive in the Inbox (not Spam) within a minute for all three. | ☐ Pass ☐ Fail |
| **DEP-06** | P1 | Backup and restore test | 1. Make a backup: mysqldump -u USER -p healthy_fasal > backup.sql and copy public/uploads.<br>2. Restore into a different empty database and point a test copy at it. | Restored site opens and logs in with all data and images. | ☐ Pass ☐ Fail |
| **DEP-07** | P1 | Backups scheduled | 1. Set a daily database backup and weekly upload backup (hosting panel or Windows Task Scheduler). | A backup file appears the next day. | ☐ Pass ☐ Fail |
| **DEP-08** | P1 | Full smoke test on the live domain | 1. Repeat these on the live site on a phone: register, order, admin confirm, partner deliver and collect, report. | Everything works over HTTPS with real email. | ☐ Pass ☐ Fail |
| **DEP-09** | P2 | Time and currency | 1. Check that times are IST and amounts show ₹ with Indian grouping (₹12,34,567). | Correct everywhere. | ☐ Pass ☐ Fail |
| **DEP-10** | P1 | Handover pack | 1. Give the client: admin login, README.md, this tracker (all P1 passed), backup instructions, who to call. | Client has everything needed to run and support the system. | ☐ Pass ☐ Fail |

---

## 7. Reporting bugs and retesting

For every **Fail**, add a row in the tracker's **Bug Log** sheet (and attach a screenshot). Good bug report:

1. **Title** – one line ("Coupon WELCOME10 accepted below ₹200").
2. **Test ID** – for example `CART-05`.
3. **Steps** – exactly how to see it again, with the account, society and product used.
4. **Expected** vs **Actual**.
5. **Severity** (below), and the browser/device.

| Severity | Meaning |
|---|---|
| **Critical** | Money, stock, security or data is wrong, or the site/page cannot be used. Blocks handover. |
| **High** | A main feature does not work, no workaround. Blocks handover. |
| **Medium** | Feature works with a workaround, or wrong message/layout on an important screen. |
| **Low** | Spelling, small visual issue, nice-to-have. |

**Retesting:** after a fix, repeat the failed test and also the tests around it (for example after a cart fix re-run all CART tests). Mark the bug *Closed* only when you saw it work. After the last fix, run the automated tests again and one full end-to-end story (group E2E).

---

## 8. Exit criteria and sign-off

The system is ready for the client when **all** of these are true (the tracker's Summary sheet checks the first three for you):

* Every **P1** test is **Pass** (or N/A with a reason).
* **No open Critical or High bugs.**
* All automated test suites report `Failed: 0`.
* Most **P2** tests pass and every remaining P2/P3 failure is a known, written-down Low/Medium issue the client accepts.
* The final group **DEP** is complete on the *live* server (real HTTPS, real email, sample accounts removed, backup restored once).

### Sign-off

| Role | Name | Signature | Date |
|---|---|---|---|
| Tester (QA) | | | |
| Developer | | | |
| Client | | | |

*Build / version tested: ____________________   Environment: ☐ Local  ☐ Staging  ☐ Live*

