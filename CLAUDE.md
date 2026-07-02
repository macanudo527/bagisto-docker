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

Use `docker compose` (v2 plugin) — the legacy `docker-compose` 1.29.2 binary crashes with `KeyError: 'ContainerConfig'` on the current Docker Engine.

```sh
docker compose build          # Build the PHP-Apache image
docker compose up -d          # Start all services
docker compose down -v        # Stop and remove volumes
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

Test suites defined in `phpunit.xml`: `Admin Feature Test`, `Core Unit Test`, `DataGrid Unit Test`, `Shop Feature Test`, `SightListing Feature Test`, `TopThings Feature Test`. Tests live in `packages/Webkul/*/tests/`.

## Service Ports

Host ports are remapped to avoid conflicts with the loominate app (which owns 80 and 6379).

| Service            | Host Port(s) | Container Port |
|--------------------|--------------|----------------|
| Web (Apache)       | 8001         | 80             |
| MySQL              | 3306         | 3306           |
| Redis              | 6380         | 6379           |
| PHPMyAdmin         | 8080         | 80             |
| Mailpit SMTP       | 1025         | 1025           |
| Mailpit UI         | 8025         | 8025           |
| Cloudflare Tunnel  | —            | (internal)     |

**Note:** Docker's iptables FORWARD chain is broken on this host (nftables conflict). Host→container HTTP never works via port mappings. Always test with `docker exec <container> curl http://localhost/`. The Cloudflare Tunnel accesses Apache directly inside the Docker network and is the only external access path. Live at `https://driftwood.topthingstodo.in`.

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

### View Resolution — Published Theme Views

Theme views are resolved in this priority order:
1. `resources/themes/top-things/views/` — **published** views (highest priority, edit these)
2. `packages/Webkul/TopThings/src/Resources/views/` — package source views (fallback)

Run `php artisan vendor:publish --tag=top-things-views` to re-publish from package source. After any view change run `php artisan optimize:clear` (not just `view:clear`) to flush all six cache types.

The TopThings Tailwind config (`packages/Webkul/TopThings/tailwind.config.js`) scans **both** package source views and published views:
```js
content: [
    "./src/Resources/**/*.blade.php",
    "./src/Resources/**/*.js",
    "../../../resources/themes/top-things/views/**/*.blade.php",
]
```
Any new Tailwind utility class added to a published view requires a theme CSS rebuild to take effect. Do not use inline `<style>` blocks as a workaround — run the build instead.

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

- URL: `https://driftwood.topthingstodo.in/admin/login` (or `http://localhost:8001/admin/login` from the host, if iptables works)
- Email: `admin@example.com`
- Password: `admin123`

---

## sight-shopper Application

`workspace/sight-shopper/` is a Bagisto fork that manages sightseeing/tour itineraries. Products represent activities; a cart is an itinerary structured by geographic area and day. All routing is delegated to the custom packages — `routes/web.php` is empty.

### Domain Concepts

- **SightListing product** — configurable product type with `area_id` (derived from the leaf-node category) and variants that carry a `time_required` attribute (minutes).
- **Restaurant product** — simple product type (`type=restaurant`) for dining listings. Has its own product page layout and `getPriceHtml()` that returns the `price_tier` EAV field (¥/¥¥/¥¥¥) instead of a Bagisto currency price. Key attributes: `cuisine`, `price_tier`, `meal_type`, `hours_of_operation`, `reservation`, `location` (lat,long CSV), `dietary_options` (comma-separated: vegetarian/vegan/halal/gluten_free), `link_to_website`, `kid_rating` (numeric 1–10), `stroller_friendly`, `what_to_order`.
- **ItineraryAreaBlock** — a geographic area in the cart, with `block_order`, `days_allocated`, and `total_time`.
- **AreaDay** — an individual day inside an area block, with `day_number` and `total_time`.
- **CartItem** — linked to an `AreaDay` via `area_day_id`; ordered within the day by `order_in_day`.

### Database Schema

```
Cart  (total_time, name, share_token)
 └── ItineraryAreaBlock  (block_order, days_allocated, total_time)
      └── AreaDay  (day_number, total_time)
           └── CartItem  (order_in_day, area_day_id)
                └── Product  (area_id, type: sight_listing | sight_variant | restaurant)
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
| `Restaurant` (type) | `Type/Restaurant.php` | Simple product type for dining listings; overrides `getPriceHtml()` to return `price_tier` (¥ symbols) instead of currency; `getLatitudeLongitude()` parses `location` CSV; `prepareForCart()` assigns `order_in_day` |
| `ProductCardOption` | `Helpers/ProductCardOption.php` | Builds sight-card config for the product detail sidebar; shows Cost card for `sight_listing`, Price Range card for `restaurant`; guards `getMinTime()`/`getMaxTime()` with `instanceof SightListingType` check |
| `Importer` | `Helpers/Importer.php` | Extends BaseImporter; `prepareProductsWithAreas()` extracts `area_id` from the leaf-node category during bulk import |
| `SightListingOption` | `Helpers/SightListingOption.php` | `getTimes()` maps variant IDs → formatted durations for the product configuration UI |
| `ItineraryController` (API) | `Http/Controllers/API/ItineraryController.php` | CRUD for saved itineraries (auth-gated): list, save, show, rename, delete |
| `SharedItineraryController` (API) | `Http/Controllers/API/SharedItineraryController.php` | Share management: generate/revoke token, public read, copy to own list |
| `ItineraryCloneService` | `Services/ItineraryCloneService.php` | Deep-clones a cart (cart → area_blocks → area_days → cart_items) for save and copy flows |
| `SavedItineraryResource` | `Http/Resources/SavedItineraryResource.php` | Serializes a saved/shared itinerary with its full structure |
| `ItineraryAccountController` | `Http/Controllers/Shop/ItineraryAccountController.php` | Web controller for the customer's saved itineraries account page |
| `BucketlistController` (API) | `Http/Controllers/API/BucketlistController.php` | Bucketlist (renamed wishlist) CRUD: list, add, move-to-cart, remove, remove-all |

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
| GET | `/api/checkout/cart` | `CartController@index` — cart with itinerary structure |
| POST | `/api/checkout/cart` | `CartController@store` — add product |
| PUT | `/api/checkout/cart` | `CartController@update` — update quantities |
| DELETE | `/api/checkout/cart` | `CartController@destroy` — remove item |
| DELETE | `/api/checkout/cart/selected` | `CartController@destroySelected` — remove selected items |
| POST | `/api/checkout/cart/move-to-bucketlist` | `CartController@moveToBucketlist` — move item to bucketlist |
| PATCH | `/api/checkout/cart/move-item` | `CartController@moveItem` — move item between days |
| PATCH | `/api/checkout/cart/update-day-order` | `CartController@updateDayOrder` — reorder days in area |
| PATCH | `/api/checkout/cart/update-area-order` | `CartController@updateAreaOrder` — reorder areas |
| POST | `/api/checkout/cart/add-empty-day` | `CartController@addEmptyDay` |
| DELETE | `/api/checkout/cart/delete-empty-day` | `CartController@deleteEmptyDay` |
| POST | `/api/checkout/onepage/orders` | `OnepageController@storeOrder` — place order |
| GET | `/api/customer/bucketlist` | `BucketlistController@index` — list bucketlist items (auth) |
| POST | `/api/customer/bucketlist` | `BucketlistController@store` — add to bucketlist (auth) |
| POST | `/api/customer/bucketlist/{id}/move-to-cart` | `BucketlistController@moveToCart` (auth) |
| DELETE | `/api/customer/bucketlist/all` | `BucketlistController@destroyAll` (auth) |
| DELETE | `/api/customer/bucketlist/{id}` | `BucketlistController@destroy` (auth) |
| GET | `/api/itineraries` | `ItineraryController@index` — list customer's saved itineraries (auth) |
| POST | `/api/itineraries` | `ItineraryController@store` — clone active cart → saved itinerary (auth) |
| GET | `/api/itineraries/{id}` | `ItineraryController@show` — load one saved itinerary (auth) |
| PATCH | `/api/itineraries/{id}` | `ItineraryController@update` — rename itinerary (auth) |
| DELETE | `/api/itineraries/{id}` | `ItineraryController@destroy` — delete saved itinerary (auth) |
| POST | `/api/itineraries/{id}/load` | `ItineraryController@load` — replace active cart with saved itinerary (auth) |
| POST | `/api/itineraries/{id}/share` | `SharedItineraryController@share` — generate share token (auth) |
| DELETE | `/api/itineraries/{id}/share` | `SharedItineraryController@revoke` — revoke share token (auth) |
| GET | `/api/shared/{token}` | `SharedItineraryController@show` — public read-only view (no auth) |
| POST | `/api/shared/{token}/copy` | `SharedItineraryController@copy` — clone into caller's saved list (auth) |

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

### Restaurant Product View

`resources/themes/top-things/views/products/view/restaurant.blade.php` is the restaurant-specific product page, included from `view.blade.php` when `$product->type === 'restaurant'`. It replaces the standard `<v-product>` component and skips the sightseeing score bars and travel-tips sections entirely.

Layout (desktop): left column — 280px cuisine icon anchor (coloured tinted bg, emoji + cuisine label); right column — fact grid (price, meal, hours, reservation, location, dietary), website link, family badges (kid rating · X/10, stroller), Add to Itinerary button. Below: description + what to order in a side-by-side row.

On mobile (≤525px): anchor goes full-width above the details; lower section stacks to column.

Dietary icons map: 🥗 Veg · 🌱 Vegan · ☪️ Halal · 🌾 GF. `none_documented` is filtered out and shows italic "None documented" instead.

`kid_rating` is stored as a numeric value (e.g. `5`) and rendered as "Kid rating · 5/10".

The `<v-restaurant-product>` Vue component handles only add-to-cart and bucketlist toggle; all other content is rendered by Blade.

`what_to_order` in the cart item view uses `v-html` (not `{{ }}` interpolation) because it is stored as HTML. Loominate exports it as raw markdown (bold only: `**dish**`); the importer converts `**bold**` → `<strong>` (with `htmlspecialchars` escaping) before storing.

### Frontend

Vue 3 app entry: `packages/Webkul/TopThings/src/Resources/assets/js/app.js`. Plugins: Axios, mitt emitter, VeeValidate, Flatpickr, @headlessui/vue (Dialog), vue-draggable-plus. Built with Vite. All cart interactions are API-driven (no full-page reloads).

**Two separate Vite builds** (both must be run after JS/CSS changes):
```sh
# Root app (outputs to public/build/)
cd workspace/sight-shopper && npm run build

# TopThings theme (outputs to public/themes/shop/top-things/build/)
cd workspace/sight-shopper/packages/Webkul/TopThings && npm run build
```

Print CSS (`@media print { … }`) lives in `packages/Webkul/TopThings/src/Resources/assets/css/app.css`. The `.no-print` utility class hides interactive controls during printing.

### Service Provider Bindings (overrides from upstream Bagisto)

```php
Cart::class           → SightListing\Cart
Product::class        → SightListing\Models\Product
CartItemRepository    → SightListing\Repositories\CartItemRepository
ProductResource       → SightListing\Http\Resources\ProductResource
```
