# AGENTS.md — toko-online (E-Commerce)

## Stack
**Laravel 12** (PHP 8.2+), **MySQL** (default) / SQLite (testing only), **Blade** + **Tailwind CSS v4** via Vite (`@tailwindcss/vite` plugin).  
No JS framework — vanilla JS in `resources/js/app.js`. Font Awesome 6 via CDN.  
Custom Tailwind v4 theme colors (`shopee`, `shopee-dark`, `shopee-light`, `shopee-bg`) + animation utilities in `resources/css/app.css`.  
`@source` directives in `app.css` tell Tailwind v4 which paths to scan for class usage.

## Quick start
```bash
composer install
npm install && npm run build
cp .env.example .env                    # if missing
php artisan key:generate                # required before migrate if .env is fresh
php artisan storage:link                # symlink for payment proofs + product images
php artisan migrate:fresh --seed        # tables + test accounts + sample products
composer dev                            # serve + queue worker + logs + Vite HMR (via concurrently)
```

## Test accounts (seeded)
| Role  | Email                 | Password |
|-------|-----------------------|----------|
| Admin | admin@tokoonline.com  | admin123 |
| User  | user@tokoonline.com   | user123  |

## Routes
All in `routes/web.php` (no API routes). Check `php artisan route:list` for exact URIs.

| Area | Prefix | Middleware |
|------|--------|------------|
| Home | `/` | — |
| Auth (login/register) | `/login`, `/register` | `guest` |
| Authenticated user | `/checkout`, `/orders`, `/payments/*`, `/wishlist`, `/reviews/*`, `/voucher/*` | `auth` |
| Admin login | `/admin/login` | — |
| Admin panel | `/admin/*` | `AdminMiddleware` (aliased `admin`, declared in `bootstrap/app.php`) |

## Models
- **User**: `role` is a plain string (`'admin'`/`'user'`). `isAdmin()` helper. Only model with a factory (`UserFactory`).
- **Product**: Scopes `active()`, `search()`. Accessors: `discounted_price`, `has_discount`, `discount_amount`, `image_url`. `is_active` is **boolean** cast. `discount_percentage` (int 0-100). `condition` enum `'baru'`/`'bekas'`. Images at `storage/app/public/products/`.
- **Order**: `items()` and `orderDetails()` are aliases for the same `hasMany(OrderDetail)` — use `items` in user views, `orderDetails` in admin views. Status: `pending`/`processing`/`completed`/`cancelled`. Payment status: `pending`/`paid`/`failed`/`refunded`. `invoice_number` format: `INV-YYYYMMDD-XXXX`.
- **Voucher**: `isValid($subtotal)`, `calculateDiscount($subtotal)`. Types: `percentage`/`nominal`.

## Cart
**Session‑based** (no DB table). `session('cart')` = `[product_id => ['price' => discounted, ...]]`.

## Payments (4 methods)
| Method | Behaviour |
|--------|-----------|
| `bank_transfer` | Upload proof → `pending` (admin confirms via `/admin/orders/{id}/payment` PUT) |
| `ewallet` | OVO/Dana/GoPay sim → `pending` |
| `gateway` | Simulated gateway → instantly `confirmed`, sets `payment_status=paid` |
| `midtrans` | **Midtrans Snap** real integration → webhook at `POST /api/payment/callback` (CSRF‑excluded) |

Proof images in `storage/app/public/payments/`.  
Midtrans config in `config/services.php` + `.env` keys.  
Snap token endpoint: `GET /payment/midtrans/snap/{orderId}` (auth).  
Webhook creates/updates `Payment` record with status `confirmed` on `capture`/`settlement`.

## Checkout
- Shipping cost is fixed at **Rp10.000** (hardcoded in `CheckoutController@process`).
- `buyNow` replaces the entire cart session with a single product, then redirects to checkout.

## Vouchers
CRUD at `/admin/vouchers`. Apply: POST `/voucher/apply`, remove: DELETE `/voucher/remove`.  
Stored on order (`discount_amount`, `voucher_code`); `used_count` incremented on order placement.

## Env quirks
- Session, queue, **and cache** all use **database** driver (not Laravel defaults).
- Custom Tailwind v4 theme defined via `@theme` in `resources/css/app.css`.
- Vite input: `resources/css/app.css` + `resources/js/app.js`.

## Views
Two layouts: `layouts.app` (user) and `layouts.admin` (admin panel). Both use `@yield('content')` + `@stack('scripts')`. Flash messages use `success`/`error` session keys, auto-dismiss after 5s.

## Testing
**PHPUnit** (not Pest). In-memory SQLite, array cache, sync queue (`phpunit.xml`).  
Feature tests need `DatabaseMigrations`/`RefreshDatabase` + `$this->seed()`.  
Run all: `php artisan test`  
Run single: `php artisan test --filter=test_name` or `.\vendor\bin\phpunit tests/Feature/ExampleTest.php`

## Code quality
Formatter: `./vendor/bin/pint` (Laravel Pint). EditorConfig: 4‑space indent, LF, UTF‑8.

## Known bugs fixed (don't reintroduce)
- `ProductController@search`: pass `$categories` + use `q` param (not `keyword`)
- User order views: `$order->items`, `$order->total_amount` (not `total`)
- Admin order views: `$order->orderDetails` (not `items`), `$order->invoice_number` (not `invoice`), `$order->total_amount` (not `total`)
- Admin order index: user name via `$order->user?->name` (not `$order->customer_name`)
- `$product->is_active` is boolean, not `status` string
- `CartController@update`: method must accept `$id` route param (not validate `product_id`)
- Cart session images need `asset('storage/')` prefix (bypass for external URLs)
- `Admin\ProductController`: `$request->boolean('is_active')` without `true` default (unchecked = false)
- `ReportController` revenue query needs `whereIn('payment_status', ['paid', 'confirmed'])`
- Midtrans webhook: pass `$request->getContent()` directly to avoid PHP input stream conflict
