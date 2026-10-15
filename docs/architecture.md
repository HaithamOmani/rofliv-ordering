# Rofliv Ordering System — Architecture

Companion to `CLAUDE.md` (rules) and `docs/db-schema.md` (tables, enums, concurrency plan). Status: Phase 0 deliverable, 15 Oct 2026.

## 1. Channels, routes, layouts, authentication

| Channel | Routes (named, Wayfinder-typed) | Layout (`resources/js/layouts`) | Authentication |
|---|---|---|---|
| Customer website | `GET /` (landing + active locations within event dates), `GET /l/{location:slug}` (menu), `POST /l/{slug}/cart/price` (server pricing), `GET/POST /l/{slug}/checkout`, `GET /pay/{order:uuid}` (payment hand-off / wait), `GET /o/{order:uuid}` (status page, QR target), `GET /o/{uuid}/status` (JSON fallback) | `customer/CustomerLayout.vue` — mobile-first, default `ar`, RTL | none (customers have no accounts) |
| Kiosk | `GET /kiosk/pair`, `POST /kiosk/pair`, `GET /kiosk` (start → menu → cart → details → pay-by-QR → done), `POST /kiosk/orders`, `POST /kiosk/orders/{uuid}/abandon` | `kiosk/KioskLayout.vue` — large tap targets, idle auto-reset, AR/EN toggle, PWA | `device` guard: httpOnly cookie holding a 48-char token, `devices.token_hash`, `DeviceType::Kiosk` |
| POS | `GET /pos` (location picker when several), `GET /pos/orders/new`, `POST /pos/orders`, `GET /pos/orders/{order}`, `POST /pos/orders/{order}/collect`, `POST /pos/orders/{order}/confirm-payment`, `GET /pos/scan` | `pos/PosLayout.vue` | Fortify session (`web`) with role staff/manager/owner/admin + `EnsureLocationAccess`; a paired POS device may also hold a `device` session |
| KDS | `GET /kds/{location}`, `POST /kds/orders/{order}/status`, `POST /kds/orders/{order}/priority` | `kds/KdsLayout.vue` — dark, large cards | `device` guard (paired KDS iPad) or `web` session with role kitchen/manager/owner/admin |
| Admin | `/admin/*` resources (dashboard, orders, locations, categories, products, addon-groups, inventory, coupons, users, devices, printers, message-templates, settings, exports), `/horizon` | `admin/AdminLayout.vue` (extends the starter sidebar layout) | `web` session + `verified` + `EnsureUserIsActive` + role owner/admin/manager (staff/kitchen get 403) |
| Webhooks | `POST /webhooks/payments/{gateway}` | — | gateway signature verification, no CSRF, `throttle:webhook` |
| Well-known | `GET /.well-known/apple-developer-merchantid-domain-association` | — | public; content from `settings`, fallback `public/.well-known` |
| Print | `GET /print/orders/{order}/receipt`, `GET /print/print-jobs/{job}` (HTML views for browser printing), CloudPRNT/ePOS pull endpoints later | Blade | `web` or `device` session scoped to the location |

`resources/js/app.ts` resolves the layout from the page-path prefix (`customer/`, `kiosk/`, `pos/`, `kds/`, `admin/`, `auth/`, `settings/`). SSR stays enabled for customer pages and is disabled by middleware for kiosk, POS and KDS.

### Middleware
`SetLocale` (customer/kiosk default `ar`, device locale, or `users.locale`; sets `app()->setLocale` and `<html dir>`), `EnsureLocationIsActive`, `EnsureLocationAccess` (owner/admin bypass; others need `location_user` membership; failure = 404), `EnsureUserIsActive`, `AuthenticateDevice` (resolves the device cookie through the `device` guard), `EnsureDeviceType`, `HandleInertiaRequests` (shared props: auth user with roles/permissions/locations, locale, translations, enabled payment methods, flash toast).

### Roles and scoping
spatie/laravel-permission global roles `owner`, `admin`, `manager`, `staff`, `kitchen`; `location_user.role` (nullable) overrides the global role at one location. Policies resolve the effective role from the pivot first. `Gate::before` grants owner/admin everything. `users.is_active` is enforced in `Fortify::authenticateUsing` (covers password, passkey and remember-me logins) and by `EnsureUserIsActive`; deactivation revokes sessions.

## 2. Folder structure
```
app/
  Actions/                      # one class per use case; the only writers of counters and statuses
    Orders/                     # PlaceOrder, ConfirmOrderPayment, ConfirmOrderManually, TransitionOrderStatus, ExpireUnpaidOrder, CancelOrder, AbandonPendingOrder, ReviveExpiredOrder, ReleaseOrderReservations, MarkOrderCollected, SetOrderPriority
    Payments/                   # StartPaymentAttempt, ProcessPaymentEvent, ReconcilePendingPayment, RecordUnexpectedPayment, RefundPayment
    Coupons/ Slots/ Inventory/  # ReserveCouponUse, ConfirmCouponUse, ReleaseCouponUse; GeneratePickupSlots, UpdateSlotCapacity, CloseSlot; ReserveStock, ReleaseStock, AdjustStock, RestockProduct
    Locations/ Menu/ Devices/   # CRUD actions, AssignStaff, GenerateLocationQr, SetLocationProductListing, PairDevice, RevokeDevice
    Printing/ Notifications/ Reports/   # EnqueueAutoPrintJobs, ReprintOrder; QueueOrderNotification; QueueDataExport
  Auth/                         # DeviceUserProvider for the 'device' guard
  Console/Commands/             # ExpireUnpaidOrders, ReconcilePendingPayments, RedrivePaymentEvents, ReleaseScheduledOrderSideEffects, GeneratePickupSlots, ReconcileSlotCounters, ReconcileStockCounters, DeactivateEndedLocations, PurgeExpiredExports, TestPlaceOrder
  Contracts/                    # PaymentGateway, PrinterDriver, WhatsAppNotifier
  Data/                         # readonly DTOs: CartData, PricedCart, SlotOption, CheckoutSession, WebhookResult, PaymentStatusResult, RefundResult, PrintPayload, OutboundMessage
  Enums/                        # string-backed enums (docs/db-schema.md)
  Events/ Listeners/ Observers/ # OrderPlaced, OrderPaid, OrderStatusChanged, OrderExpired, StockChanged, PrintJobCreated; print + WhatsApp listeners; MenuCacheInvalidator
  Exceptions/                   # InvalidOrderTransition, SlotUnavailableException (carries the suggested slot), OutOfStockException (lists products), InvalidCouponException, InvalidPriceException, PaymentVerificationException
  Exports/                      # maatwebsite/excel FromQuery exports (orders, sales by location/event, best sellers, payment methods, sources)
  Http/
    Controllers/                # Customer/, Kiosk/, Pos/, Kds/, Admin/, Webhooks/, Payments/, Print/, WellKnown/, FakeGatewayController
    Middleware/                 # SetLocale, EnsureLocationIsActive, EnsureLocationAccess, EnsureUserIsActive, AuthenticateDevice, EnsureDeviceType
    Requests/                   # FormRequests per channel and admin resource
  Jobs/                         # ExpireUnpaidOrder, ReconcilePendingPayment, RefundPaymentJob, SendWhatsAppMessage, ProcessPrintJob, GenerateDataExport
  Models/                       # Eloquent models; Concerns/ (HasBilingualName, BelongsToLocation)
  Payments/ Printing/ WhatsApp/ # PaymentGatewayManager + Drivers/FakeGateway; PrinterManager + Drivers/BrowserPrintDriver (+ StarCloudPrnt, EpsonEpos later); WhatsAppManager + Drivers/LogWhatsAppNotifier (+ MetaCloudApi later)
  Policies/                     # auto-discovered policies
  Presenters/                   # Inertia prop and broadcast payload shapes (OrderPresenter, MenuPresenter, KdsCardPresenter)
  Providers/                    # AppServiceProvider (rate limiters, utcTimestamps macro, strictness), FortifyServiceProvider, IntegrationServiceProvider (manager bindings), HorizonServiceProvider
  Rules/ Services/ Support/     # OmaniPhone; CartPricer, CouponValidator, SlotAvailability, MenuBuilder, OrderNumberGenerator, DashboardMetrics, QrCodeService, TemplateRenderer; Money, PhoneNumber, BusinessDate
config/                         # rofliv.php (channels, TTL defaults), payments.php (drivers + methods), printing.php, whatsapp.php, database.php (UTC connection timezone)
database/factories, migrations, seeders
docs/
lang/ar.json, lang/en.json
resources/js/
  app.ts                        # layout resolver by page prefix; Echo bootstrap
  components/                   # menu/, cart/, checkout/, order/, kds/, pos/, admin/, ui/ (reka-ui)
  composables/                  # useCart, useLocale, useTrans, useMoney, useEcho, useOrderChannel, useIdleReset, useQrScanner, useDeviceSession
  layouts/                      # customer/, kiosk/, pos/, kds/, admin/ (+ existing AppLayout, AuthLayout, settings)
  lib/ types/ pages/            # echo.ts, money.ts, phone.ts, i18n.ts; pages/customer|kiosk|pos|kds|admin|auth|settings
resources/views/                # app.blade.php, print/receipt-80mm.blade.php, print/sticker.blade.php, fake-gateway/checkout.blade.php
routes/                         # web.php, kiosk.php, pos.php, kds.php, admin.php, webhooks.php, print.php, channels.php, console.php, settings.php
tests/                          # Pest.php helpers; Unit/; Feature/{Customer,Checkout,Payments,Orders,Inventory,Slots,Coupons,Admin,Kiosk,Pos,Kds,Printing,WhatsApp,Reports}; Feature/Concurrency (MySQL in CI)
```

## 3. Integration contracts
```php
interface PaymentGateway {
    public function key(): string;                                   // 'fake', 'thawani', ...
    /** @return list<PaymentMethod> */
    public function supportedMethods(): array;
    public function createCheckout(Payment $payment, CheckoutOptions $options): CheckoutSession; // hosted URL / QR link, expiry
    public function verifyWebhook(Request $request): WebhookResult;   // signature + sanitisedPayload(); throws PaymentVerificationException
    public function fetchStatus(Payment $payment): PaymentStatusResult;
    public function refund(Payment $payment, int $amountBaisa, string $idempotencyKey): RefundResult;
}
interface PrinterDriver {
    public function key(): string;                                   // 'browser', 'star_cloudprnt', 'epson_epos'
    public function supports(PrinterKind $kind): bool;
    public function dispatch(PrintJob $job, Printer $printer): PrintDispatchResult; // browser: marks ready for pull; cloud: queues for the printer poll
}
interface WhatsAppNotifier {
    public function key(): string;                                   // 'log', 'meta_cloud'
    public function send(OutboundMessage $message): SendResult;      // returns provider message id / status
}
```
Managers (`PaymentGatewayManager`, `PrinterManager`, `WhatsAppManager`) extend `Illuminate\Support\Manager` and read the active driver from config; customers see only methods the active gateway supports AND the admin enabled in `settings.payments.enabled_methods`. `FakeGateway` ships a local page `/fake-gateway/{session}` with Succeed / Fail / Timeout buttons that POST a signed webhook to our own endpoint, so development and tests exercise the real webhook, expiry and reconcile paths.

## 4. Ordering and payment flow (summary; full transactions T1–T8 below)
1. Client sends cart lines (product, variant, add-ons, quantity, custom text), slot, name, phone, coupon. `CartPricer` rebuilds every price from the database.
2. `PlaceOrder` (T1) reserves slot capacity, tracked stock and the coupon under row locks, inserts the `pending_payment` order with `expires_at`, then `StartPaymentAttempt` (T1b) creates the gateway checkout (website: redirect; kiosk/POS: QR + link shown on screen).
3. The gateway calls `POST /webhooks/payments/{gateway}`; `ConfirmOrderPayment` (T2) verifies the signature, deduplicates through `payment_events`, checks amount and currency, marks the payment and order paid, counts the coupon, takes the next `R` number, transitions to `new`, and after commit broadcasts `OrderPaid`, queues the WhatsApp "confirmed" message and auto-print jobs.
4. The customer page, kiosk and POS wait on the public `order.{uuid}` channel (with a 5 s polling fallback) and then show the order number and QR.
5. Missed webhooks are covered by `payments:reconcile` (server-side `fetchStatus`) and by the manual Confirm action; unpaid orders expire after the TTL (T3) and release everything.

## 5. Realtime (Reverb + Echo)
- `order.{uuid}`: public channel; payload limited to status, number, payment state and pickup time.
- `location.{id}.orders`: private; authorised for `web` users by role/membership and for `device` sessions by `devices.location_id` (`routes/channels.php` declares guards `['web', 'device']`).
- `admin.dashboard`: private, owner/admin/manager.
- Events: `OrderPlaced`, `OrderPaid`, `OrderStatusChanged`, `OrderExpired`, `StockChanged`, `PrintJobCreated`. Echo auto-reconnects and the KDS/POS refetch on reconnect.

## 6. Caching, queues, scheduler
- Menu: `Cache::remember("menu:{location_id}:{menu_version}:{locale}", 1 day)`; `MenuCacheInvalidator` (observers on Category, Product, ProductVariant, AddonGroup, Addon, LocationProduct listing/availability/price fields, and spatie Media for products/categories) forgets the outgoing keys then bumps `locations.menu_version`. Stock, availability and slot availability are never cached. Settings cached forever, invalidated on write.
- Horizon queues: `default`, `payments`, `notifications`, `printing`, `exports`.
- Scheduler: `orders:expire-unpaid`, `payments:reconcile`, `payments:redrive-events`, `orders:release-scheduled` every minute (`withoutOverlapping()->onOneServer()`); `slots:generate` hourly; `slots:reconcile`, `stock:reconcile`, `locations:deactivate-ended`, `backup:run`, `backup:clean`, `backup:monitor`, `exports:purge` daily; `horizon:snapshot` every five minutes.

## 7. Internationalisation and RTL
`lang/ar.json` and `lang/en.json` are shared to Inertia by `HandleInertiaRequests` (per request locale) and consumed by `useTrans()`; model text uses `*_ar`/`*_en` columns through `HasBilingualName`. `SetLocale` picks `ar` by default on customer and kiosk screens (toggle persists per device or session), and `users.locale` on staff screens. `<html lang dir>` follows the locale; layouts use Tailwind logical properties. Arabic webfont loaded through the Vite fonts helper. Printed receipts and stickers are rendered server-side (HTML for browser printing now; PNG for cloud printers later) because printers do not shape Arabic.

## 8. Kiosk PWA and devices
`vite-plugin-pwa` generates `manifest.webmanifest` (display `standalone`, start URL `/kiosk`) and a service worker that precaches the shell and treats API calls network-first; Apple meta tags for Home Screen apps. The iPad runs the Home Screen app under Guided Access. Devices are paired from the admin (`devices` row with a hashed one-time pairing code), receive an httpOnly cookie with a 48-char token, are locked to one location and type, and can be revoked. Idle timeout resets the kiosk to the start screen and abandons any pending order (`AbandonPendingOrder`). Kiosk orders collect customer name and phone, source `kiosk`, pickup ASAP (`pickup_at = created_at`).

## 9. Printing
`print_jobs` rows (location, printer/device, type receipt/sticker, minimal payload, status, `dedupe_key`) are created after payment confirmation when `locations.auto_print_receipt` / `auto_print_stickers` are on (scheduled orders are released by `orders:release-scheduled` at `pickup_at - min_lead_minutes`), and on manual reprint. One sticker job per item unit. `BrowserPrintDriver` renders `print/receipt-80mm.blade.php` and `print/sticker.blade.php` (configurable size, e.g. 50×30 mm) for the browser print dialog or AirPrint; Star CloudPRNT and Epson Server Direct Print drivers (pull endpoints that serve PNG/XML) come later. Sticker fields are configurable per category and product (`sticker_fields`). Hardware comparison: `docs/printing.md` (Phase 3).

## 10. WhatsApp
`QueueOrderNotification` renders `message_templates` (AR/EN body, enabled flag, placeholders `{order_number}`, `{customer_name}`, `{location}`, `{pickup_time}`) into `notification_logs` rows with a `dedupe_key`, and `SendWhatsAppMessage` (queue `notifications`, retries with backoff) calls the active `WhatsAppNotifier`. The log driver writes to the log; the Meta Cloud API driver uses `provider_template_name_{ar,en}` and updates delivery status from Meta's webhooks. Messages: order confirmed and paid; order ready; optionally order cancelled. Customers opt in at checkout/kiosk (`orders.whatsapp_opt_in`).

## 11. Security and performance
HTTPS forced in production; named rate limiters `checkout` and `coupon` keyed by device uuid (60/min), staff user id, or IP + phone (10/min); `webhook` 120/min per IP; webhooks are the only CSRF-exempt routes; gateway payloads and printer secrets cast `encrypted:array`; no card data anywhere; signed cookies, HttpOnly device token; `Model::preventLazyLoading(!production)` and `preventSilentlyDiscardingAttributes(local)`; eager loading in every list; indexes per `docs/db-schema.md`; dashboard and export queries always filter paid, non-cancelled orders and bind Muscat day boundaries as UTC ranges.

## 12. Testing strategy
Pest feature tests per channel and resource (auth, role matrix, cross-location 404, validation, persisted state), unit tests for pricing, slots, phone, money, template rendering, an `ArchitectureTest` (string-backed enums, no `DB::raw` in actions, controllers thin), `FakeGateway` end-to-end payment tests through the real webhook route, `Queue::fake`/`Event::fake` with explicit classes, `freezeTime()`, `Http::preventStrayRequests()`. `tests/Feature/Concurrency` uses `DatabaseTruncation`, is skipped on SQLite and runs in CI against MySQL 8.4 with parallel workers (`app:test:place-order`) to prove capacity 1 / stock 1 / coupon max_uses 1 admit exactly one order and `R` numbers are unique and gapless.

## 13. Concurrency plan (verbatim from the verified design)
RULES. Global lock order: (1) orders row, (2) payments row, (3) pickup_slots row, (4) location_product rows with stock_quantity NOT NULL only, whereIn(id)->orderBy(id)->lockForUpdate() in one query, (5) coupons row, (6) order_number_sequences row. locations rows are never locked in T1-T5. All locks inside DB::transaction(fn, attempts: 3). Counters are read from the locked row and written as absolute values computed in PHP (max(0, n-1)); never decrement()/GREATEST/NOW()/DB::raw. Stock and counter writes use the query builder so no observer or menu_version bump fires. Duplicate-sensitive inserts (stock_movements, payment_events, print_jobs, notification_logs) use insertOrIgnore and treat 0 affected rows as duplicate, never a caught UniqueConstraintViolationException (Postgres aborts the transaction). No outbound HTTP inside a transaction; side effects via DB::afterCommit / dispatch()->afterCommit(). Booleans bound as PHP values.

T1 PLACE ORDER (PlaceOrder; website, kiosk, POS). Pre-transaction: normalise phone; load location (active, ordering_opens_on <= today <= event_ends_on or open-ended, website_ordering_enabled for website); CartPricer rebuilds prices from DB rows, clamps unit_price = max(0, ...) (negative -> InvalidPriceException), validates variant, addon group min/max distinct count, custom text, is_active/is_listed/is_available; CouponValidator advisory check; determine tracked location_product ids. Transaction: (3) if scheduled, or ASAP with asap_counts_toward_capacity, lock the slot; reject if !is_active, is_closed, starts_at < now()+min_lead, outside event dates, or reserved_count >= COALESCE(capacity, default_slot_capacity) -> SlotUnavailableException with SlotAvailability::nextAvailable (computed outside the lock); write reserved_count+1. (4) lock tracked rows ordered by id; require stock_quantity >= qty else OutOfStockException (lists products); write absolute new quantity; insertOrIgnore stock_movements(reserve, attempt 1, quantity_after). (5) coupon: lock; re-validate active, window, location, min_order_total, max_uses (uses_count + reserved_count < max_uses), max_uses_per_phone via coupon_redemptions count; discount = min(discount, subtotal); reject if subtotal - discount < settings.min_chargeable_total (coupon_exceeds_total); write reserved_count+1. Insert orders (uuid, number NULL, pending_payment, pickup_at, pickup_date, expires_at = now() + effective TTL), order_items (with location_product_id), order_item_addons, coupon_redemptions(reserved), history NULL->pending_payment. Commit. Post-commit: StartPaymentAttempt (T1b), broadcast OrderPlaced on order.{uuid}.

T1b START PAYMENT ATTEMPT. Transaction: (1) lock orders row; require pending_payment and not expired; if a pending unexpired payments row exists return it; else set older pending attempts superseded and insert payments(uuid, pending, idempotency_key payment:{uuid}, amount = orders.total, expires_at = orders.expires_at). Commit; then createCheckout(payment) outside any transaction with payments.uuid as client_reference and the expiry; store checkout_url/gateway_session_id; gateway failure -> attempt failed; the customer may retry (new attempt) or the order expires (T3). Double taps and QR regeneration return the same checkout_url.

T2 CONFIRM PAYMENT (ConfirmOrderPayment via ProcessPaymentEvent). Entry: PaymentWebhookController (verifyWebhook signature BEFORE any DB write; invalid -> log + 400, nothing stored), ReconcilePendingPayment (fetchStatus, source status_check), FakeGateway simulate, RedrivePaymentEvents. Step A, own transaction: insertOrIgnore payment_events(gateway, event_id, sanitised payload); if 0 rows: load existing; processed/ignored -> 200; received/failed with attempts < 5 -> attempts+1 and continue with that row as a re-drive. Step B transaction: resolve payment by payments.uuid (client_reference) then gateway_session_id, no lock; (1) lock orders; (2) lock payments. Verify event amount == orders.total AND currency OMR AND payments.amount == orders.total (Money::fromDecimalString); mismatch -> payments.failure_code amount_mismatch, event failed, orders.attention_reason amount_mismatch, alert, stop. If payments.status already paid/refund_pending -> event processed (no-op). Case pending_payment: payments paid (succeeded_at, method, gateway_transaction_id, verified_via); orders payment_status paid, payment_method, paid_at, paid_payment_id, expires_at NULL; (5) coupon lock: reserved_count-1, uses_count+1, redemption reserved->confirmed (status guard); (6) sequence lock: number = next_value, next_value+1; TransitionOrderStatus pending_payment->new (actor gateway, metadata payment_id/event_id); confirmed_at = now() unless requires_staff_confirmation; event processed. Case expired (revivable): ReviveExpiredOrder re-runs (3)(4)(5) in canonical order with attempt = 1 + prior releases (stock_movements reserve attempt n; coupon_redemptions UPDATE released->reserved); success -> expired->new (reason late_payment_revived) then continue as pending_payment case; failure -> falls to the unexpected-payment case. Case anything else (cancelled, collected, already paid via another attempt, non-revivable expired): RecordUnexpectedPayment marks the payments row refund_pending (succeeded_at, gateway_transaction_id, verified_via), sets orders.attention_reason (paid_after_cancel | duplicate_payment | late_payment_refunded | paid_after_collect), history reason late_payment_cancelled where applicable, event processed, RefundPaymentJob afterCommit, admin alert. Never 'ignored' for a verified success. Post-commit: OrderPaid broadcast (order.{uuid} public; location.{id}.orders private); if confirmed_at set: QueueWhatsAppMessage order_confirmed (dedupe order:{id}:order_confirmed) and, for ASAP orders, EnqueueAutoPrintJobs per auto_print_receipt/auto_print_stickers (dedupe auto:receipt:{id}, auto:sticker:{item}:{unit}); scheduled orders get print jobs from orders:release-scheduled when pickup_at - min_lead_minutes <= now(). When staff confirmation is required those fire from ConfirmOrderManually.

T3 EXPIRE UNPAID. orders:expire-unpaid (every minute, withoutOverlapping) selects WHERE status = pending_payment AND expires_at <= now() - expiry_grace_seconds and dispatches ExpireUnpaidOrder jobs (queue payments). Job: fetchStatus for the latest pending attempt OUTSIDE any transaction; paid -> ProcessPaymentEvent (source status_check) -> T2. Else transaction: (1) lock orders; re-check pending_payment and expires_at (another worker/webhook may have won) else skip; ReleaseOrderReservations: (3) slot lock, reserved_count = max(0, n-1) (log drift); (4) lock tracked rows by order_items.location_product_id, stock_quantity + qty, insertOrIgnore stock_movements(release_expired, attempt = current attempt); (5) coupon lock, reserved_count = max(0, n-1), redemption released; payments pending -> expired; orders.payment_status expired; TransitionOrderStatus -> expired (actor system, expired_unpaid). Commit; broadcast OrderExpired on order.{uuid}. payments:reconcile (every minute) dispatches ReconcilePendingPayment jobs for payments pending older than 60s AND payments expired created within 30 minutes (last_checked_at throttle, bounded batch); a late success goes through T2 revive/refund.

T4 CANCEL / ABANDON. CancelOrder (admin/manager; from pending_payment, new, preparing) and AbandonPendingOrder (device, order session, idle reset; pending_payment only). Transaction: (1) lock orders; state-machine check; (2) lock paid payment if any -> refund_pending with refund_idempotency_key set; pending attempts -> cancelled; ReleaseOrderReservations as T3 with reason release_cancelled / release_abandoned; if the coupon was confirmed: uses_count-1 and redemption released; status cancelled + history (actor user|device|customer, reason admin_cancelled|kiosk_abandoned|customer_abandoned); orders.payment_status cancelled (unpaid) or refund_pending (paid). Post-commit: RefundPaymentJob (locks payments row, skips if refund_reference set, writes refund_requested_at, calls refund(payment, amount, key) outside any transaction, on timeout calls fetchStatus before retry; tries 3 with backoff; RefundFailed alert) then payments refunded/refunded_amount/refund_reference and orders.payment_status refunded; QueueWhatsAppMessage order_cancelled if enabled.

T5 KITCHEN/STAFF TRANSITIONS (TransitionOrderStatus, ConfirmOrderManually, MarkOrderCollected, SetOrderPriority). Transaction: (1) lock orders; canTransitionTo else InvalidOrderTransition (409 with current state); if requires_staff_confirmation and confirmed_at IS NULL reject every transition out of new except via ConfirmOrderManually (awaiting_confirmation); update status + ready_at/collected_at/confirmed_at/confirmed_by_user_id; history (actor user|device). Post-commit: OrderStatusChanged broadcast; ready -> order_ready WhatsApp (dedupe order:{id}:order_ready); manual confirm -> auto-print + order_confirmed as in T2. Priority only broadcasts.

T6 SLOTS / INVENTORY ADMIN. UpdateSlotCapacity locks the slot and writes capacity (NULL = inherit; below reserved_count allowed). CloseSlot sets is_closed. GeneratePickupSlots (slots:generate hourly and on schedule change): builds UTC windows from schedules + exceptions in locations.timezone across event_starts_on..event_ends_on (or booking_horizon_days), insertOrIgnore on unique(location_id, starts_at) with capacity NULL, sets is_active=false on future slots outside the schedule and true on those back inside; never deletes, never touches capacity/reserved_count/is_closed. ReconcileSlotCounters and ReconcileStockCounters (nightly) recount from orders / replay stock_movements under the row lock and write reconcile_fix rows. AdjustStock/RestockProduct lock the location_product row, write the absolute value, insert stock_movements(restock|manual_adjust, user_id); re-enable sets is_available in the same statement. SetLocationProductListing toggles is_listed (rows never deleted).

T7 COUPONS: apply-time validation advisory; authoritative under the coupon lock in T1; counted in T2; released in T3/T4; coupon_redemptions.order_id unique + status guard make every step idempotent and revival an UPDATE.

T8 DEVICES: token lookup read-only via the device guard; last_seen_at updated at most once per minute. PairDevice: select by unique pairing_code_hash with lockForUpdate, verify pairing_expires_at > now() and revoked_at IS NULL, generate 48-char token, store sha256, null the code, commit, set httpOnly secure cookie.

REALTIME, CACHING, SCHEDULE: order.{uuid} public channel (status/number/payment_status/pickup only); location.{id}.orders and admin.dashboard private with guards web+device; Echo auto-reconnect + refetch; 5s polling fallback on /o/{uuid}/status. Menu cached per location+menu_version+locale with 1-day TTL; invalidator forgets old keys then bumps version only on menu-relevant changes (incl. Media for Product/Category); settings cached forever; stock/availability/slots never cached. Horizon queues: default, payments, notifications, printing, exports. Scheduler: orders:expire-unpaid, payments:reconcile, payments:redrive-events, orders:release-scheduled every minute; slots:generate hourly; slots:reconcile, stock:reconcile, locations:deactivate-ended, backup:run, backup:clean, backup:monitor, exports:purge daily; horizon:snapshot every 5 minutes.
