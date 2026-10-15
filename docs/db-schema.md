# Rofliv Ordering System — Database schema

Status: Phase 0 deliverable (15 Oct 2026), produced by three independent schema proposals merged by a judge and repaired after an adversarial review (50 findings, 38 applied). Phase 1 writes the migrations exactly as listed here; any later change is a new migration plus an update to this file.

Owner decisions applied: production database **MySQL 8.4 LTS** (SQLite for local dev and tests; the schema stays portable to PostgreSQL); stock tracked **per product per location**; ASAP kiosk/POS orders **do not** consume slot capacity unless `locations.asap_counts_toward_capacity` is on; QR codes generated with `bacon/bacon-qr-code` 3.x.

Table headings say "(Phase 1)" because every migration is created in Phase 1; "used from Phase N" marks tables that only start receiving rows in a later phase.

Conventions: string columns + PHP backed enums (no native ENUM); money is integer baisa (1 OMR = 1000); bilingual `*_ar`/`*_en` columns; all date-times UTC (`dateTime`, hot tables use the `utcTimestamps()` macro; `config/database.php` sets the connection timezone to UTC); local `date`/`time` only where noted. Counters are written as absolute values under `lockForUpdate`. Duplicate-sensitive inserts use `insertOrIgnore`.

### users (existing + Phase 1 alter)
Staff/admin accounts (Fortify login, 2FA, passkeys). Never deleted; `is_active` enforced at login and by middleware.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| name | string | no | | |
| email | string | no | | |
| email_verified_at | timestamp | yes | | |
| password | string | no | | |
| two_factor_secret | text | yes | | |
| two_factor_recovery_codes | text | yes | | |
| two_factor_confirmed_at | timestamp | yes | | |
| remember_token | rememberToken | yes | | |
| phone | string(16) | yes | | P1; E.164 |
| locale | string(2) | no | en | P1 |
| is_active | boolean | no | true | P1 |
| last_login_at | dateTime | yes | | P1 |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(email), unique(phone), index(is_active)
Foreign keys: none

### password_reset_tokens, sessions, cache, cache_locks, jobs, job_batches, failed_jobs, passkeys (existing)
Laravel and Fortify defaults, unchanged. Foreign keys: passkeys.user_id -> users.id cascadeOnDelete.

### permissions, roles, model_has_permissions, model_has_roles, role_has_permissions (Phase 1)
spatie/laravel-permission v8 published migration, teams disabled. Seeded roles: owner, admin, manager, staff, kitchen. Package indexes and cascading FKs.

### media (Phase 1)
spatie/laravel-medialibrary v11 published migration. Collections: products `images`, categories `image`, locations `hero`, `qr`. Indexes: index(model_type, model_id), unique(uuid), index(order_column).

### settings (Phase 1)
Global admin-editable key/json configuration (payments.enabled_methods, payments.gateway, payments.apple_pay_domain_association, pending_payment_ttl_minutes=10, expiry_grace_seconds=120, min_chargeable_total=100, printing.*, whatsapp.driver, kiosk.idle_reset_seconds_default, receipt.*). Cached forever, invalidated on write.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| key | string(100) | no | | |
| value | json | yes | | |
| updated_by_user_id | foreignId | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(key)
Foreign keys: updated_by_user_id -> users.id nullOnDelete

### locations (Phase 1)
A booth/event at a venue with its own schedule, slots, menu availability, inventory, staff and settings. Never deleted; `status = inactive` after the event. `venue_name_*` groups several events of one university for reports.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| slug | string(80) | no | | /l/{slug}; event QR target |
| name_ar / name_en | string(150) | no | | |
| venue_name_ar / venue_name_en | string(150) | yes | | venue grouping |
| description_ar / description_en | text | yes | | |
| address_ar / address_en | string(255) | yes | | |
| status | string(16) | no | active | LocationStatus |
| event_starts_on | date | yes | | local; NULL = open-ended |
| event_ends_on | date | yes | | local |
| ordering_opens_on | date | yes | | NULL = event_starts_on |
| booking_horizon_days | unsignedSmallInteger | no | 14 | open-ended slot horizon |
| timezone | string(40) | no | Asia/Muscat | |
| slot_length_minutes | unsignedSmallInteger | no | 15 | |
| min_lead_minutes | unsignedSmallInteger | no | 15 | |
| default_slot_capacity | unsignedSmallInteger | no | 15 | used when slot capacity IS NULL |
| asap_counts_toward_capacity | boolean | no | false | |
| kds_lookahead_minutes | unsignedSmallInteger | no | 0 | |
| payment_expiry_minutes | unsignedSmallInteger | yes | | NULL = settings TTL |
| auto_print_receipt | boolean | no | false | used from Phase 3 |
| auto_print_stickers | boolean | no | false | used from Phase 3 |
| requires_staff_confirmation | boolean | no | false | |
| website_ordering_enabled | boolean | no | true | |
| contact_phone | string(16) | yes | | |
| menu_version | unsignedInteger | no | 1 | cache key |
| sort_order | unsignedSmallInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(slug), index(status, ordering_opens_on, event_ends_on), index(venue_name_en)
Foreign keys: none

### location_user (Phase 1)
Staff assignment with optional per-location role override; scopes non-admin roles.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| user_id | foreignId | no | | |
| role | string(16) | yes | | Role; NULL = global role |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(location_id, user_id), index(user_id)
Foreign keys: location_id -> locations.id cascadeOnDelete; user_id -> users.id cascadeOnDelete

### location_schedules (Phase 1)
Weekly working windows (several per weekday; no row = closed). Local wall-clock times.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| day_of_week | unsignedTinyInteger | no | | 0=Sunday..6 |
| opens_at | time | no | | local |
| closes_at | time | no | | local, > opens_at |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(location_id, day_of_week, opens_at)
Foreign keys: location_id -> locations.id cascadeOnDelete

### location_schedule_exceptions (Phase 1)
Per-date override (closed day or different hours).

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| date | date | no | | local |
| is_closed | boolean | no | false | |
| opens_at / closes_at | time | yes | | |
| reason | string(100) | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(location_id, date)
Foreign keys: location_id -> locations.id cascadeOnDelete

### pickup_slots (Phase 1)
Materialised slots. Never deleted: regeneration inserts with `insertOrIgnore` and toggles `is_active`. `capacity` NULL inherits the location default.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| slot_date | date | no | | local |
| starts_at | dateTime | no | | UTC |
| ends_at | dateTime | no | | UTC |
| capacity | unsignedSmallInteger | yes | | NULL = location default |
| reserved_count | unsignedSmallInteger | no | 0 | absolute writes under lock |
| is_closed | boolean | no | false | manual close |
| is_active | boolean | no | true | false = retired by schedule change |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(location_id, starts_at), index(location_id, slot_date, is_active, starts_at)
Foreign keys: location_id -> locations.id cascadeOnDelete

### categories (Phase 1)
Global bilingual categories, soft-deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| name_ar / name_en | string(150) | no | | |
| is_active | boolean | no | true | |
| sort_order | unsignedSmallInteger | no | 0 | |
| sticker_fields | json | yes | | StickerField[]; NULL = settings default; used from Phase 3 |
| created_at / updated_at | timestamps | yes | | |
| deleted_at | softDeletes | yes | | |

Indexes: index(is_active, sort_order)
Foreign keys: none

### products (Phase 1)
Global catalogue, soft-deleted; images via media.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| category_id | foreignId | no | | |
| name_ar / name_en | string(150) | no | | |
| description_ar / description_en | text | yes | | |
| sku | string(64) | yes | | unique withoutTrashed in validation |
| base_price | unsignedInteger | no | | baisa |
| is_active | boolean | no | true | |
| allows_custom_text | boolean | no | false | |
| custom_text_max_length | unsignedSmallInteger | no | 30 | <= 255 |
| custom_text_label_ar / _en | string(100) | yes | | |
| sort_order | unsignedSmallInteger | no | 0 | |
| sticker_fields | json | yes | | StickerField[]; overrides category; used from Phase 3 |
| created_at / updated_at | timestamps | yes | | |
| deleted_at | softDeletes | yes | | |

Indexes: index(category_id, sort_order), index(sku), index(is_active)
Foreign keys: category_id -> categories.id restrictOnDelete

### product_variants (Phase 1)
Single-select variant with signed price delta; soft-deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| product_id | foreignId | no | | |
| name_ar / name_en | string(100) | no | | |
| price_delta | integer | no | 0 | signed baisa; unit price clamped >= 0 |
| sku | string(64) | yes | | |
| is_default | boolean | no | false | |
| is_active | boolean | no | true | |
| sort_order | unsignedSmallInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |
| deleted_at | softDeletes | yes | | |

Indexes: index(product_id, sort_order)
Foreign keys: product_id -> products.id cascadeOnDelete

### addon_groups (Phase 1)
Reusable add-on groups; min/max count distinct add-ons; soft-deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| name_ar / name_en | string(100) | no | | |
| min_select | unsignedTinyInteger | no | 0 | |
| max_select | unsignedTinyInteger | yes | | NULL = unlimited |
| is_active | boolean | no | true | |
| sort_order | unsignedSmallInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |
| deleted_at | softDeletes | yes | | |

Indexes: index(is_active, sort_order)
Foreign keys: none

### addons (Phase 1)
Add-ons inside a group with per-unit price; soft-deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| addon_group_id | foreignId | no | | |
| name_ar / name_en | string(100) | no | | |
| price | unsignedInteger | no | 0 | baisa |
| is_default | boolean | no | false | |
| is_active | boolean | no | true | |
| sort_order | unsignedSmallInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |
| deleted_at | softDeletes | yes | | |

Indexes: index(addon_group_id, sort_order)
Foreign keys: addon_group_id -> addon_groups.id cascadeOnDelete

### addon_group_product (Phase 1)
Pivot product <-> addon group with ordering.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| product_id | foreignId | no | | |
| addon_group_id | foreignId | no | | |
| sort_order | unsignedSmallInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(product_id, addon_group_id), index(addon_group_id)
Foreign keys: product_id -> products.id cascadeOnDelete; addon_group_id -> addon_groups.id cascadeOnDelete

### location_product (Phase 1)
Per-location listing, availability, price override and stock. Never deleted (`is_listed`). Model `LocationProduct extends Pivot` with `$incrementing = true`. `stock_quantity` NULL = untracked, 0 = Sold Out (shown). Written only via the query builder under `lockForUpdate`.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| product_id | foreignId | no | | |
| is_listed | boolean | no | true | offered here |
| is_available | boolean | no | true | manual disable |
| price_override | unsignedInteger | yes | | baisa; NULL = base_price |
| stock_quantity | unsignedInteger | yes | | |
| low_stock_threshold | unsignedSmallInteger | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(location_id, product_id), index(product_id), index(location_id, is_listed, is_available)
Foreign keys: location_id -> locations.id cascadeOnDelete; product_id -> products.id restrictOnDelete

### stock_movements (Phase 1)
Append-only stock ledger. `attempt` = 1 + prior releases for the order, so revival inserts attempt 2. Inserted with `insertOrIgnore`.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| location_id | foreignId | no | | |
| location_product_id | foreignId | no | | |
| order_id | foreignId | yes | | NULL for restock/adjust |
| user_id | foreignId | yes | | |
| reason | string(32) | no | | StockMovementReason |
| attempt | unsignedTinyInteger | no | 1 | |
| quantity_delta | integer | no | | signed |
| quantity_after | integer | no | | replayable |
| note | string(255) | yes | | |
| created_at | dateTime | no | | |

Indexes: unique(order_id, location_product_id, reason, attempt), index(location_product_id, created_at), index(location_id, created_at)
Foreign keys: location_id -> locations.id cascadeOnDelete; location_product_id -> location_product.id restrictOnDelete; order_id -> orders.id cascadeOnDelete; user_id -> users.id nullOnDelete

### coupons (Phase 1)
Discount codes. `uses_count` counts paid orders; `reserved_count` pending orders. Never deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| code | string(50) | no | | uppercase |
| name | string(100) | yes | | |
| type | string(16) | no | | CouponType |
| value | unsignedInteger | no | | percentage 1-100 or baisa |
| max_discount | unsignedInteger | yes | | baisa cap |
| min_order_total | unsignedInteger | no | 0 | baisa |
| max_uses | unsignedInteger | yes | | |
| max_uses_per_phone | unsignedSmallInteger | yes | | |
| uses_count | unsignedInteger | no | 0 | |
| reserved_count | unsignedInteger | no | 0 | |
| location_id | foreignId | yes | | restrict to one event |
| starts_at / ends_at | dateTime | yes | | entered in location time, stored UTC |
| is_active | boolean | no | true | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(code), index(location_id), index(is_active, starts_at, ends_at)
Foreign keys: location_id -> locations.id nullOnDelete

### coupon_redemptions (Phase 1)
One row per order using a coupon; reserved -> confirmed / released; revival UPDATEs released -> reserved.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| coupon_id | foreignId | no | | |
| order_id | foreignId | no | | |
| customer_phone | string(16) | no | | |
| status | string(16) | no | reserved | CouponRedemptionStatus |
| discount_amount | unsignedInteger | no | | baisa |
| confirmed_at | dateTime | yes | | |
| released_at | dateTime | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(order_id), index(coupon_id, status), index(coupon_id, customer_phone)
Foreign keys: coupon_id -> coupons.id restrictOnDelete; order_id -> orders.id cascadeOnDelete

### order_number_sequences (Phase 1)
Counter row locked FOR UPDATE at payment confirmation. Seeded key='orders', prefix 'R', next_value 1001.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| key | string(32) | no | | |
| prefix | string(8) | no | R | |
| next_value | unsignedBigInteger | no | 1001 | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(key)
Foreign keys: none

### orders (Phase 1; `paid_payment_id` added by a follow-up migration after `payments`)
Central order for all channels. Never deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | uuid7; /o/{uuid}, QR, public channel |
| number | unsignedBigInteger | yes | | NULL until paid; shown R{number} |
| location_id | foreignId | no | | |
| pickup_slot_id | foreignId | yes | | NULL for ASAP not counted |
| source | string(16) | no | | OrderSource |
| status | string(24) | no | pending_payment | OrderStatus |
| payment_status | string(24) | no | pending | PaymentStatus |
| payment_method | string(24) | yes | | PaymentMethod |
| paid_payment_id | foreignId | yes | | unique; the one paid attempt |
| customer_name | string(100) | no | | |
| customer_phone | string(16) | no | | E.164 |
| customer_locale | string(2) | no | ar | |
| is_asap | boolean | no | false | |
| pickup_at | dateTime | no | | UTC; slot start or created_at |
| pickup_date | date | no | | local; KDS board date |
| items_count | unsignedSmallInteger | no | 0 | |
| subtotal | unsignedInteger | no | | baisa |
| discount_total | unsignedInteger | no | 0 | <= subtotal |
| total | unsignedInteger | no | | >= min_chargeable_total |
| currency | string(3) | no | OMR | |
| coupon_id | foreignId | yes | | |
| coupon_code | string(50) | yes | | snapshot |
| priority | unsignedTinyInteger | no | 0 | |
| created_by_user_id | foreignId | yes | | POS staff |
| device_id | foreignId | yes | | kiosk/POS/KDS device; used from Phase 3 |
| confirmed_at | dateTime | yes | | |
| confirmed_by_user_id | foreignId | yes | | NULL = automatic |
| expires_at | dateTime | yes | | cleared when paid |
| paid_at | dateTime | yes | | |
| ready_at | dateTime | yes | | |
| collected_at | dateTime | yes | | |
| cancelled_at | dateTime | yes | | |
| cancel_reason | string(100) | yes | | |
| attention_reason | string(64) | yes | | amount_mismatch, paid_after_cancel, duplicate_payment, late_payment_refunded, paid_after_collect |
| notes | text | yes | | |
| metadata | json | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(uuid), unique(number), unique(paid_payment_id), index(location_id, pickup_date, status, priority, pickup_at), index(location_id, paid_at), index(paid_at, location_id), index(status, expires_at), index(customer_phone, created_at), index(pickup_slot_id), index(coupon_id), index(created_by_user_id), index(device_id)
Foreign keys: location_id -> locations.id restrictOnDelete; pickup_slot_id -> pickup_slots.id restrictOnDelete; coupon_id -> coupons.id nullOnDelete; paid_payment_id -> payments.id restrictOnDelete; created_by_user_id -> users.id nullOnDelete; confirmed_by_user_id -> users.id nullOnDelete; device_id -> devices.id nullOnDelete

### order_items (Phase 1)
Line items with name/price snapshots and the exact stock row reserved.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| order_id | foreignId | no | | |
| product_id | foreignId | no | | |
| location_product_id | foreignId | yes | | stock row locked at placement |
| product_variant_id | foreignId | yes | | |
| category_id | foreignId | yes | | |
| product_name_ar / _en | string(150) | no | | snapshot |
| variant_name_ar / _en | string(100) | yes | | snapshot |
| category_name_ar / _en | string(150) | yes | | snapshot |
| quantity | unsignedSmallInteger | no | | |
| unit_price | unsignedInteger | no | | max(0, price + delta) |
| addons_unit_total | unsignedInteger | no | 0 | |
| line_total | unsignedInteger | no | | quantity * (unit_price + addons_unit_total) |
| custom_text | string(255) | yes | | |
| sort_order | unsignedTinyInteger | no | 0 | |
| created_at / updated_at | timestamps | yes | | |

Indexes: index(order_id), index(product_id), index(location_product_id), index(product_variant_id), index(category_id)
Foreign keys: order_id -> orders.id cascadeOnDelete; product_id -> products.id restrictOnDelete; location_product_id -> location_product.id restrictOnDelete; product_variant_id -> product_variants.id restrictOnDelete; category_id -> categories.id nullOnDelete

### order_item_addons (Phase 1)
One row per selected add-on per line item (no quantity) with snapshots.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| order_item_id | foreignId | no | | |
| addon_id | foreignId | no | | |
| addon_group_id | foreignId | no | | |
| group_name_ar / _en | string(100) | no | | snapshot |
| addon_name_ar / _en | string(100) | no | | snapshot |
| unit_price | unsignedInteger | no | | baisa per item unit |
| created_at | dateTime | no | | |

Indexes: unique(order_item_id, addon_id), index(addon_id)
Foreign keys: order_item_id -> order_items.id cascadeOnDelete; addon_id -> addons.id restrictOnDelete; addon_group_id -> addon_groups.id restrictOnDelete

### order_status_histories (Phase 1)
Append-only audit of every status change.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| order_id | foreignId | no | | |
| from_status | string(24) | yes | | NULL on creation |
| to_status | string(24) | no | | |
| actor_type | string(16) | no | | StatusActor |
| user_id | foreignId | yes | | |
| device_id | foreignId | yes | | kiosk/POS/KDS device; used from Phase 3 |
| reason | string(255) | yes | | payment_confirmed, expired_unpaid, manual_confirmation, kds, pos_scan, admin_cancelled, kiosk_abandoned, customer_abandoned, late_payment_revived, late_payment_cancelled |
| metadata | json | yes | | payment_id, event_id |
| created_at | dateTime | no | | |

Indexes: index(order_id, created_at), index(to_status, created_at)
Foreign keys: order_id -> orders.id cascadeOnDelete; user_id -> users.id nullOnDelete; device_id -> devices.id nullOnDelete

### payments (Phase 1)
Payment attempts (many per order; one paid, enforced by orders.paid_payment_id). Never stores card data.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | /pay/{uuid}; provider client_reference |
| order_id | foreignId | no | | |
| gateway | string(32) | no | | driver key |
| method | string(24) | yes | | PaymentMethod |
| status | string(24) | no | pending | PaymentStatus |
| amount | unsignedInteger | no | | == orders.total |
| currency | string(3) | no | OMR | |
| idempotency_key | string(80) | no | | payment:{uuid} |
| gateway_session_id | string(191) | yes | | |
| gateway_transaction_id | string(191) | yes | | |
| checkout_url | text | yes | | |
| verified_via | string(16) | yes | | webhook / status_check |
| failure_code | string(64) | yes | | |
| failure_message | string(255) | yes | | |
| refund_idempotency_key | string(80) | yes | | refund:{uuid}:{n}; set under lock |
| refund_requested_at | dateTime | yes | | |
| refunded_amount | unsignedInteger | no | 0 | |
| refund_reference | string(191) | yes | | |
| gateway_payload | json | yes | | sanitised; encrypted:array |
| expires_at | dateTime | yes | | |
| last_checked_at | dateTime | yes | | reconcile throttle |
| succeeded_at / failed_at / refunded_at | dateTime | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(uuid), unique(idempotency_key), unique(refund_idempotency_key), unique(gateway, gateway_session_id), index(gateway, gateway_transaction_id), index(order_id, status), index(status, last_checked_at), index(status, created_at)
Foreign keys: order_id -> orders.id restrictOnDelete

### payment_events (Phase 1)
Verified webhooks and status-check results only (invalid signatures are never stored). `insertOrIgnore` on unique(gateway, event_id); unprocessed duplicates are re-driven; `payments:redrive-events` replays received/failed rows.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| gateway | string(32) | no | | |
| event_id | string(191) | no | | provider id or sha256(txn_id|status|amount) |
| event_type | string(64) | no | | |
| payment_id | foreignId | yes | | |
| order_id | foreignId | yes | | |
| source | string(16) | no | webhook | webhook / status_check / fake |
| status | string(16) | no | received | PaymentEventStatus |
| payload | json | no | | sanitised; encrypted:array |
| headers | json | yes | | encrypted:array |
| attempts | unsignedTinyInteger | no | 0 | |
| error | text | yes | | |
| processed_at | dateTime | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(gateway, event_id), index(payment_id), index(order_id), index(status, created_at)
Foreign keys: payment_id -> payments.id nullOnDelete; order_id -> orders.id nullOnDelete

### devices (Phase 1 migration; used from Phase 3)
Kiosk/POS/KDS devices locked to one location; `Device` implements Authenticatable for the `device` guard. Revoked, never deleted.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | |
| location_id | foreignId | no | | |
| type | string(16) | no | | DeviceType |
| name | string(100) | no | | |
| token_hash | string(64) | yes | | sha256; NULL until paired |
| pairing_code_hash | string(64) | yes | | unique; code >= 8 chars |
| pairing_expires_at | dateTime | yes | | |
| locale | string(2) | no | ar | |
| idle_reset_seconds | unsignedSmallInteger | no | 90 | |
| settings | json | yes | | |
| is_active | boolean | no | true | |
| last_seen_at | dateTime | yes | | |
| last_ip | string(45) | yes | | |
| registered_by_user_id | foreignId | yes | | |
| revoked_at | dateTime | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(uuid), unique(token_hash), unique(pairing_code_hash), index(location_id, type, is_active)
Foreign keys: location_id -> locations.id restrictOnDelete; registered_by_user_id -> users.id nullOnDelete

### printers (Phase 1 migration; used from Phase 3)
Printer registry per location; driver browser now, star_cloudprnt / epson_epos later.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | /cloudprnt/{uuid} |
| location_id | foreignId | no | | |
| device_id | foreignId | yes | | browser driver target |
| name | string(100) | no | | |
| driver | string(32) | no | browser | PrinterDriverType |
| kind | string(16) | no | | PrinterKind |
| paper_width_mm | unsignedSmallInteger | no | 80 | |
| label_width_mm / label_height_mm | unsignedSmallInteger | yes | | e.g. 50x30 |
| connection | json | yes | | encrypted:array |
| is_active | boolean | no | true | |
| is_default | boolean | no | false | |
| last_polled_at | dateTime | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(uuid), index(location_id, kind, is_active), index(device_id)
Foreign keys: location_id -> locations.id cascadeOnDelete; device_id -> devices.id nullOnDelete

### print_jobs (Phase 1 migration; used from Phase 3)
Receipt/sticker jobs (one sticker row per item unit) with a minimal payload; renderers read snapshotted order data. `dedupe_key` via `insertOrIgnore` makes auto-print exactly-once.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | print URLs |
| location_id | foreignId | no | | |
| printer_id | foreignId | yes | | |
| device_id | foreignId | yes | | |
| order_id | foreignId | yes | | |
| order_item_id | foreignId | yes | | stickers |
| type | string(16) | no | | PrintJobType |
| status | string(16) | no | queued | PrintJobStatus |
| copies | unsignedTinyInteger | no | 1 | |
| payload | json | no | | unit_index, copies, size, fields |
| dedupe_key | string(120) | yes | | auto:receipt:{order}, auto:sticker:{item}:{unit} |
| is_reprint | boolean | no | false | |
| attempts | unsignedTinyInteger | no | 0 | |
| error | text | yes | | |
| requested_by_user_id | foreignId | yes | | |
| claimed_at | dateTime | yes | | CloudPRNT pull |
| printed_at | dateTime | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(uuid), unique(dedupe_key), index(printer_id, status, created_at), index(location_id, status, created_at), index(order_id, type)
Foreign keys: location_id -> locations.id cascadeOnDelete; printer_id -> printers.id nullOnDelete; device_id -> devices.id nullOnDelete; order_id -> orders.id nullOnDelete; order_item_id -> order_items.id nullOnDelete; requested_by_user_id -> users.id nullOnDelete

### message_templates (Phase 1 migration; used from Phase 4)
Dashboard-managed WhatsApp texts per key with Meta provider template names. Seeded: order_confirmed, order_ready, order_cancelled (disabled).

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| key | string(50) | no | | MessageTemplateKey |
| name | string(100) | no | | |
| channel | string(16) | no | whatsapp | |
| body_ar / body_en | text | no | | {order_number} {customer_name} {location} {pickup_time} |
| provider_template_name_ar / _en | string(100) | yes | | Meta approved template |
| is_enabled | boolean | no | true | |
| updated_by_user_id | foreignId | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(channel, key)
Foreign keys: updated_by_user_id -> users.id nullOnDelete

### notification_logs (Phase 1 migration; used from Phase 4)
Every outbound message with rendered body, driver and delivery status. `dedupe_key` via `insertOrIgnore`; resends use NULL.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | |
| order_id | foreignId | yes | | |
| location_id | foreignId | yes | | |
| message_template_id | foreignId | yes | | |
| template_key | string(50) | no | | |
| channel | string(16) | no | | |
| driver | string(32) | no | | log / meta_cloud |
| recipient_phone | string(16) | no | | |
| locale | string(2) | no | | |
| body | text | no | | |
| status | string(16) | no | queued | NotificationStatus |
| provider_message_id | string(191) | yes | | |
| dedupe_key | string(120) | yes | | order:{id}:order_confirmed / order_ready |
| attempts | unsignedTinyInteger | no | 0 | |
| error | text | yes | | |
| sent_at / delivered_at / read_at / failed_at | dateTime | yes | | |
| created_at / updated_at | dateTime | yes | | utcTimestamps() |

Indexes: unique(uuid), unique(dedupe_key), index(order_id), index(location_id, created_at), index(status, created_at), index(provider_message_id)
Foreign keys: order_id -> orders.id nullOnDelete; location_id -> locations.id nullOnDelete; message_template_id -> message_templates.id nullOnDelete

### data_exports (Phase 1 migration; used from Phase 4)
Queued Excel/CSV exports with filters, status, storage path and expiry.

| column | type | null | default | notes |
|---|---|---|---|---|
| id | id | no | | |
| uuid | uuid | no | | |
| user_id | foreignId | no | | |
| type | string(32) | no | | orders / sales_summary / best_sellers |
| format | string(8) | no | xlsx | xlsx / csv |
| filters | json | yes | | |
| status | string(16) | no | queued | ExportStatus |
| disk | string(32) | yes | | |
| path | string(255) | yes | | |
| row_count | unsignedInteger | yes | | |
| error | text | yes | | |
| completed_at | dateTime | yes | | |
| expires_at | dateTime | yes | | |
| created_at / updated_at | timestamps | yes | | |

Indexes: unique(uuid), index(user_id, created_at), index(status, expires_at)
Foreign keys: user_id -> users.id cascadeOnDelete

## Enums

- **OrderStatus**: pending_payment, new, preparing, ready, collected, expired, cancelled. Transitions: pending_payment -> new | expired | cancelled; new -> preparing | ready | cancelled; preparing -> ready | cancelled; ready -> collected | preparing; expired -> new (late payment only); collected and cancelled terminal. While `requires_staff_confirmation` and `confirmed_at IS NULL`, no transition out of new except via ConfirmOrderManually.
- **PaymentStatus**: pending, paid, failed, expired, superseded, cancelled, refund_pending, refunded, partially_refunded.
- **PaymentMethod**: apple_pay, samsung_pay, card, phone_payment.
- **OrderSource**: website, kiosk, staff.
- **LocationStatus**: active, inactive.
- **CouponType**: percentage, fixed.
- **PrintJobType**: receipt, sticker, test.
- **PrintJobStatus**: queued, printing, printed, failed, cancelled.
- **PrinterDriverType**: browser, star_cloudprnt, epson_epos.
- **PrinterKind**: receipt, label.
- **DeviceType**: kiosk, pos, kds.
- **CouponRedemptionStatus**: reserved, confirmed, released.
- **StockMovementReason**: reserve, release_expired, release_cancelled, release_abandoned, restock, manual_adjust, reconcile_fix.
- **PaymentEventStatus**: received, processed, ignored, failed.
- **StatusActor**: user, device, system, gateway, customer.
- **NotificationStatus**: queued, sent, delivered, read, failed, skipped.
- **MessageTemplateKey**: order_confirmed, order_ready, order_cancelled.
- **StickerField**: order_number, customer_name, product_name, variant, addons, custom_text, pickup_time, location.
- **Role**: owner, admin, manager, staff, kitchen.
- **ExportStatus**: queued, processing, completed, failed.
- **Locale**: ar, en.

## Concurrency plan

Global lock order (never reversed, skip what is not needed): orders row -> payments row -> pickup_slots row -> location_product rows with tracked stock (one `whereIn(id)->orderBy(id)->lockForUpdate()` query) -> coupons row -> order_number_sequences row. `locations` is never locked inside order transactions. Every mutation runs inside `DB::transaction(fn, attempts: 3)`. Counters are read from the locked row and written as absolute PHP-computed values (`max(0, n - 1)`), never SQL `decrement()`, `GREATEST`, `NOW()` or `DB::raw`. Stock and counter writes use the query builder so observers never fire. Duplicate-sensitive inserts (`stock_movements`, `payment_events`, `print_jobs`, `notification_logs`) use `insertOrIgnore` and treat zero affected rows as a duplicate; a caught `UniqueConstraintViolationException` is never used as control flow (Postgres aborts the transaction). No outbound HTTP inside a transaction; broadcasts, print jobs, WhatsApp and refunds run `afterCommit`.

- **T1 PlaceOrder**: pre-transaction pricing from DB rows (clamped `unit_price >= 0`, `discount <= subtotal`, `total >= min_chargeable_total`), add-on min/max, custom text, listing/availability; transaction: slot lock (active, open, lead time, event dates, `reserved_count < COALESCE(capacity, default)`; else SlotUnavailableException with suggested next slot), tracked stock locks (OutOfStockException), coupon lock (window, location, min total, max_uses, per-phone), then insert order (pending_payment, `expires_at`), items with `location_product_id`, add-ons, redemption, history; after commit **T1b StartPaymentAttempt** locks the orders row, reuses the pending attempt or supersedes older ones, inserts `payment:{uuid}`, then calls `createCheckout` outside the transaction with `payments.uuid` as client reference.
- **T2 ConfirmOrderPayment**: signature verified before any write (invalid = 400, nothing stored); `insertOrIgnore` payment_events (duplicates that are not processed are re-driven, attempts < 5); lock orders then payments; verify amount/currency; pending_payment -> mark paid, set `paid_payment_id`, confirm coupon, take the next R-number, transition to new, `confirmed_at` unless staff confirmation is required. Expired orders are revived by re-reserving with `attempt + 1` ledger rows and an UPDATE of the redemption. Any other state records the payment as `refund_pending` with `orders.attention_reason` and queues an idempotent refund; a verified success is never ignored. Side effects after commit: broadcasts, order_confirmed WhatsApp, auto-print (ASAP immediately, scheduled via `orders:release-scheduled`).
- **T3 Expire**: `orders:expire-unpaid` selects `expires_at <= now() - grace` and dispatches one job per order; the job calls `fetchStatus` outside any transaction (paid -> T2), then locks the order, re-checks, releases slot/stock/coupon (absolute values, ledger rows `release_expired`), marks payments and `orders.payment_status` expired, transitions to expired. `payments:reconcile` keeps polling pending attempts older than 60 s and expired attempts for 30 minutes.
- **T4 Cancel / Abandon**: lock order (and paid payment -> `refund_pending` with `refund_idempotency_key`), release reservations (`release_cancelled` / `release_abandoned`), decrement `uses_count` if the coupon was confirmed, pending attempts -> cancelled, transition to cancelled; RefundPaymentJob after commit (locks the payment, skips if `refund_reference` set, `fetchStatus` on timeout, 3 tries).
- **T5 Kitchen/staff transitions**: lock order, state machine check, awaiting-confirmation guard, timestamps, history; broadcast and order_ready WhatsApp after commit.
- **T6 Slots / inventory admin**: capacity edits under the slot lock; `GeneratePickupSlots` uses `insertOrIgnore` and `is_active` (never deletes); nightly slot and stock reconciliation under row locks with `reconcile_fix` ledger rows; restock/adjust under the location_product lock.
- **T7 Coupons**: advisory at apply time, authoritative in T1, counted in T2, released in T3/T4; the unique redemption row makes every step idempotent.
- **T8 Devices**: read-only token lookup via the `device` guard; pairing by unique `pairing_code_hash` under lock.
- **Realtime / cache / schedule**: `order.{uuid}` public (status, number, payment state only); `location.{id}.orders` and `admin.dashboard` private with guards `web` + `device`; Echo reconnect + refetch and a 5 s polling fallback. Menu cached per location + `menu_version` + locale with a 1-day TTL, old keys forgotten on bump; stock, availability and slots never cached. Scheduler: expire-unpaid, reconcile, redrive-events, release-scheduled every minute; slots:generate hourly; reconciliations, deactivate-ended, backups, exports:purge daily. Concurrency suites run against a MySQL 8.4 service in CI (skipped on SQLite); PostgreSQL stays supported by the schema but is not the chosen production engine.

## Migration order

All tables are created in Phase 1 (owner instruction: Phase 1 ships every migration; later phases only use them). One `php artisan migrate:fresh --seed` must pass on SQLite and MySQL 8.4. Compared with the design's original split, `devices` moves before `orders` so `device_id` is part of the create migrations, and `sticker_fields` is part of the `categories`/`products` create migrations.

```
[existing] 0001_01_01_000000_create_users_table
[existing] 0001_01_01_000001_create_cache_table
[existing] 0001_01_01_000002_create_jobs_table
[existing] 2024_01_01_000000_create_passkeys_table
[existing] 2025_08_14_170933_add_two_factor_columns_to_users_table
[P1] 2026_10_15_000001_add_profile_columns_to_users_table
[P1] 2026_10_15_000002_create_permission_tables            (spatie publish)
[P1] 2026_10_15_000003_create_media_table                  (spatie publish)
[P1] 2026_10_15_000004_create_settings_table
[P1] 2026_10_15_000005_create_locations_table
[P1] 2026_10_15_000006_create_location_user_table
[P1] 2026_10_15_000007_create_location_schedules_table
[P1] 2026_10_15_000008_create_location_schedule_exceptions_table
[P1] 2026_10_15_000009_create_pickup_slots_table
[P1] 2026_10_15_000010_create_categories_table             (incl. sticker_fields)
[P1] 2026_10_15_000011_create_products_table               (incl. sticker_fields)
[P1] 2026_10_15_000012_create_product_variants_table
[P1] 2026_10_15_000013_create_addon_groups_table
[P1] 2026_10_15_000014_create_addons_table
[P1] 2026_10_15_000015_create_addon_group_product_table
[P1] 2026_10_15_000016_create_location_product_table
[P1] 2026_10_15_000017_create_coupons_table
[P1] 2026_10_15_000018_create_order_number_sequences_table (seeds orders/R/1001)
[P1] 2026_10_15_000019_create_devices_table
[P1] 2026_10_15_000020_create_orders_table                 (incl. device_id; without paid_payment_id)
[P1] 2026_10_15_000021_create_order_items_table
[P1] 2026_10_15_000022_create_order_item_addons_table
[P1] 2026_10_15_000023_create_order_status_histories_table (incl. device_id)
[P1] 2026_10_15_000024_create_stock_movements_table
[P1] 2026_10_15_000025_create_coupon_redemptions_table
[P1] 2026_10_15_000026_create_payments_table
[P1] 2026_10_15_000027_create_payment_events_table
[P1] 2026_10_15_000028_add_paid_payment_id_to_orders_table
[P1] 2026_10_15_000029_create_printers_table
[P1] 2026_10_15_000030_create_print_jobs_table
[P1] 2026_10_15_000031_create_message_templates_table
[P1] 2026_10_15_000032_create_notification_logs_table
[P1] 2026_10_15_000033_create_data_exports_table
```
