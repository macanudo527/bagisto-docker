# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A Docker-based development environment for Bagisto (Laravel e-commerce). It orchestrates PHP-Apache, MySQL, Redis, PHPMyAdmin, Elasticsearch, Kibana, and Mailpit containers, with the actual Laravel application code living in `workspace/`.

## Initial Setup

```sh
sh setup.sh
```

This tears down existing containers, rebuilds, starts services, waits for MySQL, creates databases (`bagisto` and `bagisto_testing`), clones Bagisto into `workspace/bagisto/`, and runs the installer.

If your host UID is not `1000`, update `uid` in `docker-compose.yml` before running.

## Docker Commands

```sh
docker-compose build          # Build the PHP-Apache image
docker-compose up -d          # Start all services
docker-compose down -v        # Stop and remove volumes
```

## Shell Access

```sh
sh bin/php-apache-container.sh   # Opens bash in bagisto working dir (/var/www/html/bagisto)
sh bin/mysql-container.sh        # Opens bash in MySQL container
```

## Laravel Commands (run inside the PHP-Apache container)

```sh
php artisan <command>            # General artisan
./vendor/bin/pest                # Run all tests
./vendor/bin/pest --testsuite="Admin Feature Test"   # Run a specific suite
./vendor/bin/pint                # Fix code style (Laravel preset + aligned => operators)
php artisan bagisto:install --skip-env-check --skip-admin-creation
```

Test suites defined in `phpunit.xml`: `Admin Feature Test`, `Core Unit Test`, `DataGrid Unit Test`, `Shop Feature Test`. Tests live in `packages/Webkul/*/tests/`.

## Service Ports

| Service       | Port(s)      |
|---------------|--------------|
| Web (Apache)  | 80           |
| MySQL         | 3306         |
| Redis         | 6379         |
| PHPMyAdmin    | 8080         |
| Elasticsearch | 9200, 9300   |
| Kibana        | 5601         |
| Mailpit SMTP  | 1025         |
| Mailpit UI    | 8025         |

## Architecture

### Workspace Layout

- `workspace/sight-shopper/` — primary application, served at the Apache root (`/`)
- `workspace/bagisto/` — upstream Bagisto dev-master, accessible at `/bagisto`
- `workspace/bagisto-old/` — previous version kept for reference

The Apache vhost (`configs/apache.conf`) sets `sight-shopper/public` as `DocumentRoot` and aliases `/bagisto` to `bagisto/public`.

### Bagisto Package Architecture

Both `bagisto` and `sight-shopper` follow the Webkul monorepo pattern: all feature code lives in `packages/Webkul/<PackageName>/src/` and is autoloaded via a `packages/*/*` path repository (symlinked into `vendor/`). The `sight-shopper` fork adds two custom packages absent from upstream:

- `Webkul\TopThings` — `packages/Webkul/TopThings/src`
- `Webkul\SightListing` — `packages/Webkul/SightListing/src`

### Environment Files

`.configs/.env` and `.configs/.env.testing` are the source-of-truth env files copied into the container during setup. Key values:

- DB host: `bagisto-mysql` (Docker service name, not `localhost`)
- Redis host: `bagisto-redis`
- Mail host: `bagisto-mailpit` port `1025`
- Testing DB: `bagisto_testing`

### Image Details

- Base: `php:8.3-apache`
- PHP extensions: `bcmath calendar exif gd gmp intl mysqli pdo pdo_mysql zip`
- Node: 23 (copied from `node:23` image)
- Composer: 2.7
- Apache rewrite module enabled; runs as a non-root user matching the host UID

## Admin Login (after install)

- URL: `http://localhost/admin/login`
- Email: `admin@example.com`
- Password: `admin123`

---

## sight-shopper Application

`workspace/sight-shopper/` is a Bagisto fork that manages sightseeing/tour itineraries. Products represent activities; a cart is an itinerary structured by geographic area and day. All routing is delegated to the custom packages — `routes/web.php` is empty.

### Domain Concepts

- **SightListing product** — configurable product type with `area_id` (derived from the leaf-node category) and variants that carry a `time_required` attribute (minutes).
- **ItineraryAreaBlock** — a geographic area in the cart, with `block_order`, `days_allocated`, and `total_time`.
- **AreaDay** — an individual day inside an area block, with `day_number` and `total_time`.
- **CartItem** — linked to an `AreaDay` via `area_day_id`; ordered within the day by `order_in_day`.

### Database Schema

```
Cart  (total_time, name, share_token)
 └── ItineraryAreaBlock  (block_order, days_allocated, total_time)
      └── AreaDay  (day_number, total_time)
           └── CartItem  (order_in_day, area_day_id)
                └── Product  (area_id, type: sight_listing | sight_variant)
                     └── Variant  (time_required attribute)
```

**Cart row semantics:**
- `is_active = 1` — the working cart; never touched by save/copy operations
- `is_active = 0`, `name` set — a saved itinerary
- `is_active = 0`, `name` + `share_token` set — a shared itinerary (public read-only link)

### Key Classes — Webkul\SightListing

| Class | Path | Role |
|---|---|---|
| `SightListingServiceProvider` | `Providers/SightListingServiceProvider.php` | Bootstraps the package: binds Cart, Product, CartItemRepository, ProductResource; registers CartTotalsListener |
| `Cart` | `Cart.php` | Extends BaseCart; overrides `addProduct()` and `initCart()` for itinerary-aware cart creation |
| `CartController` (API) | `Http/Controllers/API/CartController.php` | REST endpoints for cart CRUD, item/day/area reordering, and coupon management |
| `OnepageController` (API) | `Http/Controllers/API/OnepageController.php` | Checkout flow: address → shipping → payment → order |
| `CartResource` | `Http/Resources/CartResource.php` | Serializes Cart with totals, itinerary blocks, and `total_time` formatted via TimeHelper |
| `ItineraryAreaBlockResource` | `Http/Resources/ItineraryAreaBlockResource.php` | Serializes area blocks with child days and time totals |
| `AreaDayResource` | `Http/Resources/AreaDayResource.php` | Serializes individual days with ordered items |
| `CartTotalsListener` | `Listeners/CartTotalsListener.php` | Fires on `checkout.cart.collect.totals.after`; recalculates `total_time` for every CartItem → AreaDay → ItineraryAreaBlock → Cart |
| `ItineraryAreaBlockRepository` | `Repositories/ItineraryAreaBlockRepository.php` | `moveArea()` reorders area blocks safely using temporary values to avoid unique constraint conflicts |
| `AreaDayRepository` | `Repositories/AreaDayRepository.php` | `moveDayNumber()` reorders days within an area transactionally |
| `CartItemRepository` | `Repositories/CartItemRepository.php` | `decrementOrderAfter()` / `incrementOrderFrom()` maintain `order_in_day` sequence |
| `SightListing` (type) | `Type/SightListing.php` | Custom configurable product type; `create()` sets `area_id` from category hierarchy; `prepareForCart()` packages variants |
| `Importer` | `Helpers/Importer.php` | Extends BaseImporter; `prepareProductsWithAreas()` extracts `area_id` from the leaf-node category during bulk import |
| `SightListingOption` | `Helpers/SightListingOption.php` | `getTimes()` maps variant IDs → formatted durations for the product configuration UI |
| `ItineraryController` (API) | `Http/Controllers/API/ItineraryController.php` | CRUD for saved itineraries (auth-gated): list, save, show, rename, delete |
| `SharedItineraryController` (API) | `Http/Controllers/API/SharedItineraryController.php` | Share management: generate/revoke token, public read, copy to own list |
| `ItineraryCloneService` | `Services/ItineraryCloneService.php` | Deep-clones a cart (cart → area_blocks → area_days → cart_items) for save and copy flows |
| `SavedItineraryResource` | `Http/Resources/SavedItineraryResource.php` | Serializes a saved/shared itinerary with its full structure |
| `ItineraryAccountController` | `Http/Controllers/Shop/ItineraryAccountController.php` | Web controller for the customer's saved itineraries account page |

### Key Classes — Webkul\TopThings

| Class | Path | Role |
|---|---|---|
| `TopThingsServiceProvider` | `Providers/TopThingsServiceProvider.php` | Registers the `top-things` theme, binds Toolbar, loads translations/views |
| `Toolbar` | `Helpers/Toolbar.php` | Adds custom sort orders: "Must-Do First", "Optional First" |
| `TimeHelper` | `Helpers/TimeHelper.php` | Formats minutes into human-readable durations (days/hours/minutes) used across views and resources |

### API Routes (`SightListing/src/Routes/Shop/api.php`)

All routes are prefixed `/api`.

| Method | Path | Action |
|---|---|---|
| GET | `/cart` | `CartController@index` — cart with itinerary structure |
| POST | `/cart/add/{id}` | `CartController@store` — add product |
| DELETE | `/cart/remove/{id}` | `CartController@destroy` — remove item |
| PATCH | `/cart/move-item` | `CartController@moveItem` — move item between days |
| PATCH | `/cart/update-day-order` | `CartController@updateDayOrder` — reorder days in area |
| PATCH | `/cart/update-area-order` | `CartController@updateAreaOrder` — reorder areas |
| POST | `/cart/add-empty-day` | `CartController@addEmptyDay` |
| DELETE | `/cart/delete-empty-day` | `CartController@deleteEmptyDay` |
| POST | `/checkout/order` | `OnepageController@storeOrder` — place order |
| GET | `/api/itineraries` | `ItineraryController@index` — list customer's saved itineraries (auth) |
| POST | `/api/itineraries` | `ItineraryController@store` — clone active cart → saved itinerary (auth) |
| GET | `/api/itineraries/{id}` | `ItineraryController@show` — load one saved itinerary (auth) |
| PATCH | `/api/itineraries/{id}` | `ItineraryController@update` — rename itinerary (auth) |
| DELETE | `/api/itineraries/{id}` | `ItineraryController@destroy` — delete saved itinerary (auth) |
| POST | `/api/itineraries/{id}/share` | `SharedItineraryController@share` — generate share token, return public URL (auth) |
| DELETE | `/api/itineraries/{id}/share` | `SharedItineraryController@revoke` — revoke share token (auth) |
| GET | `/api/shared/{token}` | `SharedItineraryController@show` — public read-only view (no auth) |
| POST | `/api/shared/{token}/copy` | `SharedItineraryController@copy` — clone shared itinerary into caller's saved list (auth) |

### Add-to-Cart / Itinerary Build Flow

1. Customer adds a `sight_listing` product → `CartController@store` → `Cart::addProduct()`.
2. `SightListing::prepareForCart()` resolves the chosen variant and its `time_required`.
3. `CartItem` is created with `area_day_id` (routed to the correct `AreaDay`) and `order_in_day`.
4. `checkout.cart.collect.totals.after` fires → `CartTotalsListener` rolls up `total_time` bottom-up: items → AreaDay → ItineraryAreaBlock → Cart.
5. `CartResource` serializes the full structure for the Vue 3 frontend.

### Save / Share Itinerary Flow

- **Save:** `ItineraryController@store` → `ItineraryCloneService` deep-clones the active cart (all area blocks, days, items) into a new `Cart` row with `is_active = 0` and the user-supplied name. The active cart is untouched.
- **Share:** `SharedItineraryController@share` sets `share_token = Str::uuid()` on a saved itinerary and returns the public URL. Revoke clears the token.
- **Copy shared:** `SharedItineraryController@copy` deep-clones the shared cart into the authenticated customer's saved list with name "Copy of {original name}". Active cart is untouched.
- **Edit saved:** Not yet implemented — deferred.

### Reorder Flow

- Items within a day: `moveItem` — calls `CartItemRepository::decrementOrderAfter()` + `incrementOrderFrom()` around the target slot.
- Days within an area: `updateDayOrder` — calls `AreaDayRepository::moveDayNumber()` in a transaction using temp values.
- Areas in itinerary: `updateAreaOrder` — calls `ItineraryAreaBlockRepository::moveArea()` in a transaction using temp values.

### Frontend

Vue 3 app entry: `packages/Webkul/TopThings/src/Resources/assets/js/app.js`. Plugins: Axios, mitt emitter, VeeValidate, Flatpickr. Built with Vite. All cart interactions are API-driven (no full-page reloads).

### Service Provider Bindings (overrides from upstream Bagisto)

```php
Cart::class           → SightListing\Cart
Product::class        → SightListing\Models\Product
CartItemRepository    → SightListing\Repositories\CartItemRepository
ProductResource       → SightListing\Http\Resources\ProductResource
```
