# Rofliv Ordering System — project rules for Claude Code

Rofliv (نظام Rofliv للطلبات الإلكترونية والبوثات) is ONE cloud platform: website pre-orders with pickup, iPad self-service kiosks and a staff POS at booths in universities and events, a kitchen display, printing, WhatsApp notifications and an owner dashboard across many simultaneous locations. It is a full ordering platform, not a brochure site.

Source of truth for requirements: `docs/requirements.md` (Arabic original, §1–§19) and the owner's brief that restates it. If they ever disagree, stop and ask the owner. No requirement may be dropped, simplified or postponed without the owner's approval. `docs/requirements-traceability.md` maps every requirement to its phase, files and tests; a phase is complete only when every row of that phase is `done` with a passing test named in the row. Update the matrix at the end of every phase.

## Stack (fixed; do not add packages without the owner's approval)
- Laravel 13 (PHP 8.5), Vue 3 + Inertia 3 + TypeScript + Tailwind 4 (reka-ui components in `resources/js/components/ui`), Vite 8 (vite-plus), Wayfinder for typed routes, Fortify for staff auth (login, 2FA, passkeys; no public registration).
- Laravel Reverb + Echo (`laravel-echo`, `pusher-js`) for realtime; Redis cache/queues/sessions + Horizon; Pest 5 for tests; PHPStan level 7; Pint (laravel preset).
- Approved packages: `laravel/reverb`, `laravel/horizon`, `spatie/laravel-permission`, `spatie/laravel-medialibrary`, `spatie/laravel-backup`, `maatwebsite/excel`; npm `@zxing/browser` (POS camera QR scan) and `vite-plugin-pwa` (kiosk PWA). QR generation uses `bacon/bacon-qr-code` 3.x directly (already installed by Fortify; `simplesoftwareio/simple-qrcode` cannot be installed because it pins bacon ^2.0). Anything else: ask first.
- Databases: SQLite for local dev and tests; **MySQL 8.4 LTS** in production; schema must stay portable (string columns + PHP enums, no native ENUM, no DB-specific SQL).

## Commands
- `composer run dev` (app + queue + logs + vite), `php artisan test --compact` (narrow with a path or `--filter`), `vendor/bin/pint --dirty --format agent` before finishing PHP changes, `phpstan analyse`, `npm run check`, `npm run types:check`, `npm run build`, `php artisan wayfinder:generate` after route changes.
- Use `php artisan make:*` generators (`--no-interaction`), Laravel Boost tools (`search-docs`, `database-schema`, `database-query`, `get-absolute-url`, `browser-logs`) before guessing framework APIs.
- Concurrency tests (`tests/Feature/Concurrency`) are skipped on SQLite and run in CI against MySQL 8.4.

## Phases (implement ONE phase at a time, then STOP and wait for the owner)
0 Planning docs (done) · 1 Foundation (enums, all migrations, models, factories, seeders, roles/scoping, admin CRUD, tests) · 2 Online ordering & payments · 3 Booth operations (kiosk PWA, POS, KDS, printing) · 4 WhatsApp, dashboard/exports, backups, hardening. Details: `docs/phases.md`. After each phase: run migrations, seeders and the full suite and fix every failure; update the traceability matrix; summarise what was built, how to run it and the decisions needed; commit; stop. Commit after each completed task with a clear message. Never mark a requirement done without a passing test.

## Architecture rules (details in `docs/architecture.md`, schema in `docs/db-schema.md`)
- Five Inertia channels, each with its own layout chosen by page-path prefix in `resources/js/app.ts`: customer (`/`, `/l/{location:slug}`, `/o/{order:uuid}`), kiosk (`/kiosk`, device token), POS (`/pos`), KDS (`/kds`), admin (`/admin`). SSR on for customer pages only.
- Thin controllers. Business logic in `app/Actions` (one use case per class, `handle()`) and `app/Services`; validation in `app/Http/Requests`; authorization in `app/Policies` (auto-discovered); DTOs in `app/Data`; drivers behind `app/Contracts/{PaymentGateway, PrinterDriver, WhatsAppNotifier}` resolved by `Illuminate\Support\Manager` subclasses.
- Enums (`app/Enums`, string-backed): `OrderStatus`, `PaymentStatus`, `PaymentMethod`, `OrderSource`, `LocationStatus`, `CouponType`, `PrintJobType` plus the supporting enums in `docs/db-schema.md`. `OrderStatus::transitions()` is the only transition map; `TransitionOrderStatus` is the only place a status changes and always writes `order_status_histories`.
- Money is integer **baisa** (1 OMR = 1000) in `*_baisa` columns, shown as OMR with 3 decimals through `App\Support\Money`; every price is computed server-side (`CartPricer`) from DB rows; never trust client prices.
- Bilingual: `*_ar` / `*_en` columns, `lang/ar.json` + `lang/en.json` shared to Inertia, `useTrans()`; customer and kiosk default to Arabic with RTL (`<html dir>` from locale); staff screens use `users.locale`.
- Time: app timezone `Asia/Muscat`, storage UTC, DB connection timezone `+00:00`, never SQL `NOW()`; local `date`/`time` only in schedules, `pickup_slots.slot_date`, `orders.pickup_date`.
- Phones: validate Omani numbers (`App\Rules\OmaniPhone`), store E.164 `+968XXXXXXXX` (`App\Support\PhoneNumber`).
- Multi-location from day one: every non-admin role is scoped through `location_user` (+ optional per-location `role`); cross-location access answers 404; adding a location is data, never code. Locations, slots and `location_product` rows are never deleted (status/`is_active`/`is_listed` flags).
- Concurrency: reservations happen inside `DB::transaction(attempts: 3)` with `lockForUpdate` in the global lock order orders → payments → pickup_slots → tracked `location_product` rows (`whereIn(id)->orderBy(id)`) → coupons → `order_number_sequences`; counters are rewritten as absolute PHP-computed values; duplicate-sensitive inserts use `insertOrIgnore` (never catch a unique violation as control flow); no outbound HTTP inside a transaction; side effects run `afterCommit`.
- Payments are prepaid only, on every channel; no cash. Orders start `pending_payment`; only a signature-verified webhook (or a server-side `fetchStatus`) confirms payment, checks amount and currency, and is idempotent via `payment_events`; the order number (`R` + gapless sequence) is assigned at confirmation. Never store card data. Kiosk/POS pay through a QR/link the customer opens on their own phone and wait on Reverb.
- Realtime: `order.{uuid}` is a public channel (status, number, payment state only); `location.{id}.orders` is private with guards `web` + `device`.
- Performance: cached menu per location/version/locale with TTL, `Model::preventLazyLoading` outside production, eager loading everywhere, indexes from `docs/db-schema.md`, no N+1.
- Security: HTTPS forced in production, named rate limiters (`checkout`, `coupon`, `webhook`, keyed by device uuid / user id / IP+phone), webhooks excluded from CSRF only, encrypted gateway payloads and printer secrets, daily off-site backups with failure notifications.

## Business rules (short form; `docs/requirements.md` wins)
- Customer flow New → Preparing → Ready → Collected (جديد → قيد التحضير → جاهز → تم الاستلام); internal `pending_payment`, `expired`, `cancelled`. Ready triggers the WhatsApp "ready" message; confirmation triggers the "confirmed and paid" message.
- Unpaid orders expire after 10 minutes (per-location override, 120 s grace) and release slot capacity, stock and coupon reservations; a kiosk idle reset abandons its pending order immediately.
- Slots come from each location's schedule within event dates, respect `min_lead_minutes`, have a capacity (`NULL` = location default) and close when full with the next slot suggested; ASAP kiosk/POS orders never consume capacity unless the location toggle is on.
- Stock is per product per location: decremented on reservation, restored on expiry/cancel, `0` shows "Sold Out" (shown, not hidden), one-click restock/re-enable; products are soft-deleted.
- Coupons: percentage or fixed, window, max uses, per-phone max, optional location, minimum order; validated server-side under lock, counted only when payment succeeds; a coupon may not push the total below 100 baisa.
- Roles: owner/admin (everything), manager (assigned locations: settings, slots, menu availability, inventory, orders), staff (POS and mark collected at assigned locations), kitchen (KDS status changes only).
- "Confirm order": the customer reviews and confirms before paying; staff/admin have a Confirm action that re-checks the gateway (`fetchStatus`) and confirms a paid order; per-location `requires_staff_confirmation` optionally holds paid orders in `new` until staff tap Confirm.

## Conventions
- PHP: curly braces always, constructor promotion, explicit types, PHPDoc with array shapes, `#[Fillable]` / `#[Hidden]` attributes and `casts()` as in `app/Models/User.php`, `CarbonImmutable`, descriptive names (`isRegisteredForDiscounts`). Models carry `@property` docblocks. Prefer `search-docs` over memory for framework APIs.
- Vue/TS: `<script setup lang="ts">`, single root element, reka-ui components first, Wayfinder route functions (never hard-coded URLs), Inertia `Form`/`useForm`, deferred props get skeletons. Run `npm run check` and `types:check`.
- Tests: Pest feature tests first (auth, role matrix via datasets, cross-location 404, validation, happy path asserting persisted state), unit tests only for framework-free logic, factories with named states, framework fakes (`Queue::fake([...])`, `Event::fake([...])`, `Http::preventStrayRequests()`, `freezeTime()`), `LazilyRefreshDatabase`. Read the `testing-best-practices` skill before writing tests. Never delete tests without approval.
- Docs: only the files listed in `docs/` and this file; create new documentation only when asked.

## Owner decisions on record (15 Oct 2026)
1. Requirements come from the owner's Arabic document, stored verbatim in `docs/requirements.md`; five "preferred" items are mandatory per the brief (order QR, dashboard-managed messages, Excel/CSV export, Staff as an order source, subtotal/discount on orders).
2. ASAP kiosk/POS orders do not count toward slot capacity (per-location toggle, default off).
3. Production database MySQL 8.4 LTS.
4. `@zxing/browser` and `vite-plugin-pwa` approved.
5. QR codes via `bacon/bacon-qr-code` 3.x directly (simple-qrcode dropped).
6. Stock per product per location.
Design defaults that stand unless the owner says otherwise: single global `R` sequence starting at 1001; late or duplicate verified payments are auto-refunded and listed under "Attention"; paid orders can be cancelled (with refund) from `new`/`preparing` only; the public status page never expires; pending-payment TTL 10 min.

## Docs map
`docs/requirements.md` (Arabic original) · `docs/requirements-traceability.md` · `docs/technical-proposal.md` · `docs/db-schema.md` · `docs/architecture.md` · `docs/phases.md` · `docs/printing.md` (written in Phase 3).
