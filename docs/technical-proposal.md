# Rofliv Ordering System — Technical proposal (§19 deliverable)

Date: 15 October 2026 · Status: Phase 0 deliverable for owner review.

Every vendor fact in this document was taken from an official vendor or documentation page on 15 Oct 2026 and re-checked by a second, independent fact-check (sources in the appendix). Anything that could not be verified from an official page — including every fee, price or capability a vendor did not publish — is written as **TO CONFIRM WITH VENDOR**. Nothing below is an invented price, cost or capability.

## 1. Summary
Rofliv gets one Laravel 13 application serving five channels (customer website, iPad kiosk, staff POS, kitchen display, admin) over one MySQL database, with Redis/Horizon queues, Reverb WebSockets, S3-compatible media storage and daily off-site backups. Payments are prepaid only through an Oman-licensed gateway behind a `PaymentGateway` contract (a fake driver is used until the vendor is chosen); WhatsApp and printers sit behind the same kind of contracts. Everything lives in Rofliv's own repository and hosting accounts and uses open-source, MIT-compatible packages only (§11).

## 2. Proposed technologies
| Layer | Choice | Version verified 15 Oct 2026 | Licence |
|---|---|---|---|
| Backend | Laravel 13 on PHP 8.5 | laravel/framework 13.36 (PHP 8.3–8.5 supported; bug fixes to Q3 2027, security fixes to Mar 2028) | MIT |
| Frontend | Vue 3 + Inertia 3 + TypeScript + Tailwind 4, Vite 8 (vite-plus), reka-ui components, Wayfinder typed routes | @inertiajs/vue3 3.9, vue 3.5, tailwindcss 4.3, vite 8.3 | MIT |
| Auth (staff) | Laravel Fortify (login, 2FA, passkeys; no public registration) | 1.41 | MIT |
| Realtime | Laravel Reverb + Laravel Echo (`pusher-js` transport) | reverb 1.12.0, laravel-echo 2.6.1, pusher-js 8.6.0 | MIT |
| Queues | Redis + Laravel Horizon | horizon 5.50.1 (requires Redis; phpredis extension present locally) | MIT |
| Roles | spatie/laravel-permission | 8.3.0 | MIT |
| Media | spatie/laravel-medialibrary (S3 disk in production) | 11.23.9 | MIT |
| Backups | spatie/laravel-backup (needs `mysqldump`) | 10.3.3 | MIT |
| Exports | maatwebsite/excel (PhpSpreadsheet 5.x, XLSX/CSV) | 4.0.4 | MIT |
| QR codes | bacon/bacon-qr-code 3.x used directly (installed by Fortify; SVG + PNG via Imagick). `simplesoftwareio/simple-qrcode` was dropped: its last release (4.2.0, Feb 2021) pins bacon ^2.0 and cannot coexist with Fortify's ^3.0 | 3.1.1 | BSD-2-Clause |
| Kiosk PWA | vite-plugin-pwa (Vite 8 peer dependency) | 2.0.0 (1.3.x fallback) | MIT |
| POS QR scanning | @zxing/browser (iPad Safari still ships the native BarcodeDetector API disabled by default) | latest | MIT |
| Tests / quality | Pest 5, PHPStan 7, Pint, vite-plus check | pest 5.3 | MIT |
| Databases | SQLite for development and tests; **MySQL 8.4 LTS** in production (owner decision); schema portable to PostgreSQL | — | GPL/Public |

## 3. Development duration per phase (estimate)
| Phase | Scope | Estimate |
|---|---|---|
| 0 | Planning documents (this package) | done |
| 1 | Foundation: enums, all migrations, models, factories, seeders, roles and location scoping, admin CRUD (locations, categories, products, variants, add-ons, per-location menu/stock/price, users, devices), tests | ≈ 2 weeks |
| 2 | Online ordering and payments: slots and capacity, pricing and coupons, order creation with reservations, payment contract + fake gateway, webhooks, reconciliation, expiry, customer website, confirmation and status pages | ≈ 3 weeks |
| 3 | Booth operations: kiosk PWA with QR payment, POS with scan-to-collect, KDS with Reverb, printing (receipt and sticker views, print jobs, auto-print), `docs/printing.md` | ≈ 3 weeks |
| 4 | WhatsApp (contract + log driver + templates + logs), dashboard and Excel/CSV exports, backups, rate limiting and performance pass | ≈ 2 weeks |
| After 4 | Live integrations once vendors are onboarded: Oman gateway driver, Meta WhatsApp Cloud API driver, Star CloudPRNT / Epson Server Direct Print drivers | depends on vendor onboarding lead times — TO CONFIRM WITH VENDOR |

Estimates assume one developer working with the AI assistant, owner feedback within two working days at each phase gate, and sandbox access from the chosen gateway and WhatsApp provider before Phase 2 ends.

## 4. Project cost, annual hosting cost, maintenance cost (§19)
These figures are not invented here.
- **Project cost**: TO CONFIRM — developer quotation based on the phase scope above (the traceability matrix lists every requirement so the quote can be itemised).
- **Annual hosting cost**: the drivers and published unit prices are in §5; the yearly figure depends on the chosen track, region and sizes and is TO CONFIRM WITH VENDOR for the unpublished items.
- **Maintenance cost**: scope in §6; figure TO CONFIRM — developer quotation.

## 5. Hosting architecture and what drives the annual cost
Same application code on either track; components: stateless app server(s) → MySQL 8.4 → Redis (cache, sessions, queues) → Reverb WebSocket process → Horizon workers → scheduler (every minute) → S3-compatible bucket for product images and QR files → second-provider bucket for backups.

**Track A — Laravel Cloud (fully managed, least operations work).** Published prices (laravel.com/cloud/pricing, 15 Oct 2026): Growth plan $20/month (needed for always-on Pro compute, worker clusters and more than one queue; Starter is $5/month but Flex-only, 1 replica, 2 GiB max); App compute Pro 4 GiB $32/month (Flex 1 GiB $12/month scales to zero); a worker Flex 1 GiB $12/month running `php artisan horizon` as a custom background process (Horizon is not supported on Cloud's managed SQS queues, so queues stay on Redis-compatible Valkey); Laravel Valkey Flex 250 MB $6/month; managed Reverb cluster 100 concurrent connections $5/month (200,000 messages/day cap; 200 connections $10, 500 $25); Laravel MySQL (price per size and region TO CONFIRM WITH VENDOR; Cloud's pricing data lists UAE `me-central-1` rates such as mysql-flex-1gb $15.40/month, but the public region list shows nine regions with Frankfurt as the nearest, so **availability of the UAE region is TO CONFIRM WITH VENDOR**; the region is fixed per application); object storage (Cloudflare R2 under the hood) $0.02/GB-month with free egress; bandwidth overage $0.10/GB. The Cloud image ships imagick and the redis extension; `mysqldump` availability for backups is TO CONFIRM WITH VENDOR.

**Track B — Laravel Forge + a VPS (lowest monthly cost, Middle-East placement possible).** Forge Hobby $12/month or Growth $19/month (flat rate); one Ubuntu server with Nginx, PHP 8.5, MySQL 8.4 and Redis, where Forge's one-click toggles create the scheduler cron, the Horizon daemon and the Reverb daemon with its own `ws.` hostname and TLS. Server options: AWS `me-south-1` (Bahrain) or `me-central-1` (UAE) EC2 for proximity to Oman (opt-in regions; instance prices TO CONFIRM WITH VENDOR), DigitalOcean 2 GiB droplet $12/month (Frankfurt; no Middle-East region), Hetzner (no Middle-East region; prices TO CONFIRM WITH VENDOR). Optional managed database: DigitalOcean Managed MySQL from $15.15/month. Oman-local hosting providers were not evaluated: TO CONFIRM WITH VENDOR.

**Storage and backups (both tracks).** Cloudflare R2 $0.015/GB-month, free egress (free tier 10 GB) for media; Backblaze B2 $6.95/TB-month (EU Central) as the off-site backup destination for spatie/laravel-backup (daily DB dump + local storage, retention 7 days all / 16 daily / 8 weekly / 4 monthly / 2 yearly by default, failure notifications by mail/webhook). Media stored on S3 is protected by bucket versioning or replication, which is configured at the storage provider.

**What drives the annual cost**: app compute size and replica count, managed database size, Reverb connection tier and daily message volume (every KDS/POS/admin screen holds a connection), bandwidth, storage volume, number of environments (a staging environment doubles most lines), SSR (an extra process), and backup retention.

## 6. Maintenance scope
Laravel and package security updates (monthly) and PHP minor upgrades; dependency audit (`composer audit`); backup monitoring and a quarterly restore drill; gateway, WhatsApp template and printer firmware changes; onboarding each new event (location, menu, staff, devices, QR print); uptime monitoring and incident response during events; small feature requests. Figure: TO CONFIRM (developer quotation).

## 7. Payment gateways in Oman and how each payment method connects
Integration pattern for every gateway (already designed): the server creates a hosted checkout session → the customer pays on the gateway's page (or scans the QR shown on the kiosk/POS and pays on their own phone) → the gateway calls our signed webhook → the server verifies the signature, amount (baisa) and currency (OMR), records the event idempotently and confirms the order → a scheduled status query covers missed webhooks → refunds go through the gateway's refund API with an idempotency key. Card data never touches Rofliv.

| Gateway | Status in Oman | Apple Pay | Samsung Pay | Cards / OmanNet | Pay by phone number | Webhook / status / refund | PHP / Laravel | Published fees |
|---|---|---|---|---|---|---|---|---|
| **Thawani Pay** | Licensed PSP (Central Bank of Oman) | Configurable in the merchant Business Portal together with Google Pay | Not mentioned → TO CONFIRM WITH VENDOR | Visa, Mastercard; OmanNet TO CONFIRM WITH VENDOR | Thawani wallet, Tajer QR and payment links; no e-commerce request-to-pay API found | Hosted checkout, webhooks (success/failure/refund), session retrieve, refunds; signature spec not readable (docs are script-rendered) → TO CONFIRM WITH VENDOR | Community packages only (no official SDK) | TO CONFIRM WITH VENDOR |
| **Paymob Oman** | Licensed PSP (CBO, Jan 2024) | Documented for the OMN region (domain verification + certificates for Pixel/API; Unified Checkout steps TO CONFIRM) | TO CONFIRM WITH VENDOR | Visa, Mastercard, Amex, OmanNet; full and partial refunds | — | HMAC-signed callbacks ("source of truth"), transaction inquiry, refund API | Official `paymob/php-library` | Oman pricing page: 1.5 % OmanNet / 2.5 % Visa-Mastercard per transaction, no monthly fee, weekly settlement; its own FAQ says 2.9 % + 1 OMR → **TO CONFIRM WITH VENDOR** |
| **PayTabs Oman** | Oman endpoint `secure-oman.paytabs.com` | Available for Oman/OMR (activation by email) | UAE only | OmanNet enabled by default (sale mode only), Visa, Mastercard | — | IPN + callback with HMAC-SHA256 `Signature` header, query API, refund API | Official `paytabs/laravel-sdk` v3 (Laravel 10–13; proprietary licence — used only as an optional driver dependency) | TO CONFIRM WITH VENDOR; payouts weekly on Mondays above 120 OMR |
| AmwalPay | Licensed PSP (CBO) | Express Apple Pay in SmartBox (embedded JS) | TO CONFIRM WITH VENDOR | Visa, Mastercard; OmanNet TO CONFIRM WITH VENDOR | — | Server-to-server cloud notification with secure hash, status by merchant reference, refund API | `amwalpay/laravel-package` | TO CONFIRM WITH VENDOR (plan tiers shown; transaction fee "in the contract") |
| Tap Payments | Regulatory approval incl. Oman; OMR payouts Mon/Wed, min 25 OMR | On hosted checkout (Tap hosts the Apple certificate; merchant hosts the domain file); OMR not in the published Apple Pay list → TO CONFIRM WITH VENDOR | Documented (Samsung developer account required); Oman TO CONFIRM WITH VENDOR | OmanNet (`src_omannet`), cards | — | `post.url` webhook with `hashstring`, retrieve charge, idempotent refunds | None official | TO CONFIRM WITH VENDOR (pricing page not reachable) |
| Bank Muscat (SmartRoute hosted page) | Local acquirer | TO CONFIRM WITH VENDOR | TO CONFIRM WITH VENDOR | Hosted page; methods not published | Bank Muscat MPCSS QR for merchants (in person) | No public API documentation → TO CONFIRM WITH VENDOR | — | TO CONFIRM WITH VENDOR |
| Checkout.com / Amazon Payment Services / HyperPay | OmanNet "gateway only" (needs a local acquirer) / OmanNet via APS / lists Oman | Checkout.com and APS document Apple Pay (APS requires Rofliv's own Apple Developer membership, $99/year) | Not documented | OmanNet (OMR, direct capture) | — | Documented webhooks | Checkout.com official PHP SDK (MIT) | TO CONFIRM WITH VENDOR |
| Telr, Stripe | Telr has no Oman footprint; Stripe does not serve Oman-registered merchants | — | — | — | — | — | — | not applicable |

- **Apple Pay on the web**: Rofliv serves `/.well-known/apple-developer-merchantid-domain-association` over HTTPS (TLS 1.2+); which party hosts the Apple merchant identity certificate and runs merchant validation differs per gateway (Tap does it for the merchant; Paymob and APS require certificates on the merchant side; Thawani, PayTabs and AmwalPay: TO CONFIRM WITH VENDOR).
- **Samsung Pay**: no vendor documents it for Oman today; it stays an optional capability switched on only when the chosen gateway confirms it.
- **Pay by phone number**: in Oman this is the Central Bank's Mobile Payment Clearing and Switching System (MPCSS: mobile number, alias or QR, OMR, person-to-business). No gateway fetched exposes an e-commerce "request to pay" API for it, so `phone_payment` is implemented as a driver capability that stays off until a vendor provides an API; the booth fallback is a merchant MPCSS QR or a payment link sent by SMS/WhatsApp (Thawani Tajer, AmwalPay Pay-by-Link). Regulatory context: CBO caps merchant fees (credit cards ≤ 1.5 %; QR scan-and-pay ≤ 0.50 %, max RO 2, from 1 July 2026).
- **In-person card acceptance later**: Thawani Tajer "Lamsa" tap-to-pay, Bank Muscat Tap 'N' Go, Sohar International Soft POS, BankDhofar Tap-to-Phone — all Android apps today; iPad availability and API linkage TO CONFIRM WITH VENDOR.
- **Recommendation**: shortlist Thawani Pay, Paymob Oman and PayTabs Oman. Before Phase 2 ends, obtain from each: the binding rate card and settlement terms, sandbox credentials, the webhook signature specification, Apple Pay merchant-validation ownership, Samsung Pay and OmanNet refund support, and the onboarding document list.

## 8. WhatsApp Business solution options
All paths end at Meta's WhatsApp Business Platform; the choice is direct Cloud API integration versus a Business Solution Provider.
- **Meta Cloud API direct (recommended for Rofliv's volume)**: no platform fee; Meta bills per delivered template message by category (utility for order updates) and recipient country. Oman is in the "Rest of Middle East" region and moves to a standalone rate card with higher utility rates on 1 October 2026; from the same date Meta also charges for free-form service messages and for utility messages inside the 24-hour window. Third-party rate cards quote Oman utility at ≈ USD 0.025–0.028 per message (≈ OMR 0.010–0.011; two messages per order ≈ OMR 0.02) — **official Meta rate TO CONFIRM WITH VENDOR**. OMR is not a billing currency (card billing in USD). Requirements: Meta business portfolio + WhatsApp Business Account, a dedicated number (coexistence with the WhatsApp Business app is possible through a partner), display name, approved **utility** templates (review up to 24 h; promotional content gets reclassified as marketing), Arabic language code `ar` or `ar_AE` (there is no `ar_OM`), customer opt-in recorded at checkout/kiosk (Oman Personal Data Protection Law, Royal Decree 6/2022), HTTPS webhook with `X-Hub-Signature-256` verification for delivery status, free test number for development.
- **BSP alternatives**: Twilio ($0.005 per message plus Meta fees; a Laravel-13-ready MIT notification channel exists), 360dialog (49 EUR per channel per month), Gupshup ($0.001 per message), Wati (subscription from ≈ $119/month). Useful if Rofliv prefers partner-led onboarding and billing; each adds its fee on top of Meta's.
- **Design**: `WhatsAppNotifier` contract with a log driver now; templates managed in the dashboard with provider template names per language; every send logged with provider id and delivery status; queued with retries.

## 9. Compatible printer types and connection methods
Rofliv's server runs in the cloud while printers sit on booth Wi-Fi behind NAT, so the only robust direct path is a printer that **pulls** jobs from our HTTPS endpoint:
- **Star CloudPRNT** (built into Star mC-Print3, mC-Print2, TSP100IV receipt printers and mC-Label3 / TSP143IV SK linerless label printers): the printer polls our URL (HTTP or MQTT), fetches jobs as PNG, StarPRNT or text, and confirms printing; no VPN, port forwarding or certificate on the printer. Our driver renders receipts and stickers to PNG (which also solves Arabic shaping). Self-hosted CloudPRNT has no documented fee (TO CONFIRM WITH VENDOR); Star's hosted StarIO.Online is subscription-based (price and Oman availability TO CONFIRM WITH VENDOR). QR payloads must be ASCII (URL/token).
- **Epson Server Direct Print** (TM-m30III, TM-T88VII receipts; TM-L100 linerless labels): the printer polls our URL at a configurable interval and prints ePOS-Print XML; Digest auth + HTTPS. Epson ePOS-Print on the LAN is an alternative only when a device on the same network drives it.
- **Die-cut label printers** (Brother QL-820NWB, Zebra ZD421/ZD621): good 50×30 mm stickers, but no vendor pull protocol — they need iPad AirPrint from the browser-print fallback or an on-site bridge (PrintNode or a small PC), or Zebra's Cloud Print API subscription.
- **Fallbacks and limits**: raw ESC/POS over TCP 9100 is LAN-only and unreachable from the cloud; iPad Safari AirPrint works only with AirPrint-certified printers on the same Wi-Fi with limited paper control; consumer label printers (Niimbot, Phomemo) have no supported server path.
- **Shortlist**: Star mC-Print3 (80 mm receipt + QR) and Star mC-Label3 (linerless stickers) on one protocol; or Epson TM-m30III + TM-L100. Hardware prices and the Oman distributor: TO CONFIRM WITH VENDOR. The booth needs an internet connection for the printer (Ethernet or Wi-Fi module). Detailed comparison before purchase: `docs/printing.md` (Phase 3).

## 10. Third-party services with recurring monthly or annual fees
| Service | Needed for | Published fee (15 Oct 2026) | Else |
|---|---|---|---|
| Hosting (Track A Laravel Cloud) | app, workers, scheduler | Growth $20/mo + compute from $12–$32/mo + Valkey $6/mo + Reverb $5/mo + MySQL + storage $0.02/GB-mo | MySQL size/region TO CONFIRM WITH VENDOR |
| Hosting (Track B Forge + VPS) | same | Forge $12–$19/mo + VPS (DigitalOcean $12/mo; AWS Bahrain/UAE EC2 TO CONFIRM WITH VENDOR) | managed DB optional $15.15/mo |
| Object storage | product images, QR files, exports | Cloudflare R2 $0.015/GB-mo, free egress | — |
| Backup storage | off-site daily backups | Backblaze B2 $6.95/TB-mo | — |
| Domain + TLS | website, API, WebSockets | domain registration (registrar) TO CONFIRM; TLS free (Let's Encrypt) | — |
| Payment gateway | all payments | Paymob published 1.5 % / 2.5 % (conflicting FAQ) | all others TO CONFIRM WITH VENDOR |
| WhatsApp (Meta) | notifications | per delivered template message; Oman standalone rate from 1 Oct 2026 | official rate TO CONFIRM WITH VENDOR (≈ USD 0.025–0.028 per third parties) |
| WhatsApp BSP (optional) | partner onboarding | Twilio $0.005/msg, 360dialog 49 EUR/mo, Gupshup $0.001/msg, Wati from ≈ $119/mo | — |
| Printer cloud (optional) | hosted CloudPRNT / Zebra Cloud Print | subscription-based | TO CONFIRM WITH VENDOR |
| Apple Developer Program | only if Rofliv must host its own Apple Pay merchant certificate (e.g. Amazon Payment Services) | $99/year | not needed with gateway-hosted validation |
| Transactional email | backup failure alerts, password resets | provider dependent | TO CONFIRM |
| Uptime monitoring (optional) | event-day alerting | provider dependent | TO CONFIRM |

## 11. Code ownership
Yes: the system, its source code, database and all assets are fully owned by Rofliv after delivery. The code lives in Rofliv's own git repository; hosting, storage, gateway, WhatsApp and domain accounts are opened in Rofliv's name; secrets are held by Rofliv. The platform uses only open-source packages with MIT-compatible licences (Laravel, Inertia, Vue, Tailwind, reka-ui, spatie packages, maatwebsite/excel, Horizon, Reverb — all MIT; bacon-qr-code BSD-2-Clause). No proprietary framework, SaaS back-end or vendor runtime is required to run the application; the only proprietary-licensed component that could ever enter the project is PayTabs' optional SDK, and even then only as a replaceable driver behind Rofliv's own `PaymentGateway` contract. Rofliv can move hosting providers, gateways, WhatsApp providers and printers without rewriting the application.

## 12. Open items to settle before Phase 2 ends
Gateway shortlist answers (§7), WhatsApp onboarding path and rate card (§8), hosting track and region (§5), printer order after `docs/printing.md` (§9).

## Appendix — sources consulted (official pages, fetched 15 Oct 2026)
Laravel: laravel.com/docs/13.x (reverb, horizon, broadcasting, filesystem, database, queries), laravel.com/cloud/pricing, laravel.com/cloud/docs (compute, workers, queues, websockets, resources/caches, object-storage, applications, environments), laravel.com/forge/pricing and /forge/docs. Packagist/GitHub: laravel/reverb, laravel/horizon, spatie/laravel-permission, spatie/laravel-medialibrary, spatie/laravel-backup, SpartnerNL/Laravel-Excel, SimpleSoftwareIO/simple-qrcode, Bacon/BaconQrCode, endroid/qr-code; npm registry: vite-plugin-pwa, laravel-echo, pusher-js. Apple: developer.apple.com (Apple Pay on the web, merchant domains), webkit.org (web push, storage policy), support.apple.com (Guided Access). Payments: thawani.om (payment-gateway, business-portal, tajer-app, developers), amwalpay.om/developers, om.paymob.com/en/pricing.html and developers.paymob.com, support.paytabs.com and docs.paytabs.com, developers.tap.company and support.tap.company, bankmuscat.om, checkout.com/payment-methods/omannet, paymentservices.amazon.com (OmanNet, Apple Pay), hyperpay.com, telr.com, stripe.com/global, cbo.gov.om (MPCSS operating rules, National Payment Systems report 2022), omanobserver.om (CBO fee changes 2026, Bank Muscat QR, AmwalPay). WhatsApp: developers.facebook.com (pricing, non-template-messages pricing update, templates, categorization, messaging limits, throughput, webhooks, supported languages, solution providers), twilio.com/whatsapp/pricing, docs.360dialog.com and 360dialog.com/pricing, gupshup.io pricing, wati.io pricing, GitHub composer.json of netflie/laravel-notification-whatsapp and laravel-notification-channels/twilio. Printers: star-m.jp (CloudPRNT protocol guide 2.6.0, Star Document Markup, mC-Print3, TSP100IV, mC-Label3), star-emea.com (CloudPRNT, mC-Print2/3, StarIO.Online), starmicronics.com (StarIO.Online), epson (Server Direct Print specification, TM-m30III, TM-L100 technical reference), brother (QL-820NWB/QL-1110NWB), zebra.com (SendFileToPrinter, Cloud Print API), printnode.com/pricing. Storage: developers.cloudflare.com/r2/pricing, digitalocean.com/pricing, backblaze.com/cloud-storage/pricing, docs.aws.amazon.com (regions), docs.hetzner.com (locations). Browser support: caniuse.com and MDN (BarcodeDetector), bugs.webkit.org.
