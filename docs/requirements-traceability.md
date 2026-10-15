# Rofliv Ordering System — Requirements traceability matrix

Status: Phase 0 (15 Oct 2026). One `R-` row per requirement in `docs/requirements.md` (every bullet and every sentence-level requirement of §1–§19 plus the final goal), and one `B-` row per elaboration the owner's brief adds on top of the Arabic document. IDs are `R-§§-nn` / `B-§§-nn`. Status values: `planned`, `in progress`, `done` (with the passing test named in the row), `done (document)` for Phase 0 deliverables that are documents reviewed by the owner. A phase is complete only when every row assigned to it is `done`. This file is updated at the end of every phase.

Counts: 157 `R-` rows (covering all 101 bullets of the Arabic document plus 56 sentence-level requirements) and 98 `B-` rows.

## Part A — Requirements from docs/requirements.md

| ID | Requirement (AR) | Requirement (EN) | Phase | Implementation (files / routes) | Test(s) | Status |
|---|---|---|---|---|---|---|
| R-01-01 | نحتاج تطوير نظام طلبات إلكتروني متكامل لعلامة Rofliv، وليس مجرد موقع تعريفي. | A complete ordering system for Rofliv, not a brochure website. | 1–4 | whole platform (customer site, kiosk, POS, KDS, admin) | full suite | planned |
| R-01-02 | النظام يجب أن يخدم مسارين من خلال نظام مركزي واحد (Backend + Dashboard) | One central backend + dashboard serves both ordering paths. | 1 | single Laravel app; `orders` table shared by all sources; `app/Http/Controllers/{Customer,Kiosk,Pos,Kds,Admin}` | tests/Feature/Orders/OrderSourcesTest (orders from every source land in the same table) | planned |
| R-01-03 | الطلب المسبق عبر الموقع الإلكتروني والاستلام في الوقت والموقع المحدد. | Online pre-order on the website with pickup at a chosen time and location. | 2 | customer website flow (`routes/web.php`, `pages/customer/*`, `PlaceOrder`) | tests/Feature/Customer/CheckoutFlowTest | planned |
| R-01-04 | الطلب المباشر من بوث Rofliv في الجامعات والفعاليات من خلال أجهزة iPad بنظام Self-Service Kiosk. | Direct ordering at Rofliv booths in universities and events via iPad self-service kiosk. | 3 | kiosk PWA (`routes/kiosk.php`, `pages/kiosk/*`) | tests/Feature/Kiosk/KioskOrderTest | planned |
| R-01-05 | يجب تصميم النظام من البداية بحيث يكون Cloud-Based وقابلًا للتوسع مستقبلًا إلى عدة بوثات ومواقع تعمل في الوقت نفسه. | Cloud-based from day one and scalable to several booths/locations running simultaneously. | 1 | `locations` table + `location_*` scoping; stateless app servers; Redis/Horizon; `docs/architecture.md` §1, `docs/technical-proposal.md` §5 | tests/Feature/Admin/LocationCrudTest (several active locations), tests/Feature/Orders/MultiLocationIsolationTest | planned |
| R-02-01 | رحلة العميل: الدخول للموقع → اختيار موقع الاستلام → اختيار المنتجات → تخصيص الطلب → تحديد وقت الاستلام → إدخال البيانات → الدفع → تأكيد الطلب | Customer journey: landing → pickup location → products → customise → pickup time → details → pay → order confirmation. | 2 | `pages/customer/{Landing,Menu,Checkout,Pay,OrderStatus}.vue`, `CustomerCheckoutController` | tests/Feature/Customer/CheckoutFlowTest (end-to-end with FakeGateway) | planned |
| R-02-02 | اختيار موقع/فعالية الاستلام. | Choose the pickup location/event. | 2 | `GET /` lists active locations within event dates; `LocationPicker.vue` | tests/Feature/Customer/LocationListingTest | planned |
| R-02-03 | مشاهدة المنيو المتاح في ذلك الموقع فقط. | See only the menu available at that location. | 2 | `MenuBuilder` filters `location_product.is_listed`; cached per location | tests/Feature/Customer/MenuVisibilityTest | planned |
| R-02-04 | اختيار المنتج والكمية. | Choose product and quantity. | 2 | `CartData`, `useCart`, `CartPricer` | tests/Unit/Pricing/CartPricerTest, tests/Feature/Customer/CartPricingTest | planned |
| R-02-05 | اختيار الإضافات والخيارات الخاصة بكل منتج. | Choose the add-ons and options (variants) of each product. | 2 | `product_variants`, `addon_groups`/`addons`, `CartPricer` min/max validation | tests/Unit/Pricing/AddonGroupRulesTest | planned |
| R-02-06 | كتابة اسم أو رسالة مخصصة للمنتجات التي تدعم ذلك. | Write a custom name or message on products that support it. | 2 | `products.allows_custom_text` + `custom_text_max_length`; `order_items.custom_text` | tests/Feature/Customer/CustomTextTest | planned |
| R-02-07 | اختيار تاريخ ووقت الاستلام من الفترات المتاحة. | Choose pickup date and time from the available slots. | 2 | `SlotAvailability`, `pickup_slots`, `SlotPicker.vue` | tests/Feature/Slots/SlotAvailabilityTest | planned |
| R-02-08 | إدخال الاسم ورقم الهاتف. | Enter name and phone number. | 2 | `StoreCustomerOrderRequest` (required name, `OmaniPhone`) | tests/Feature/Customer/CheckoutValidationTest | planned |
| R-02-09 | مراجعة الطلب قبل الدفع. | Review the order before paying. | 2 | `pages/customer/Review.vue` (items, variants, add-ons, custom text, location, date/time, discount, total) | tests/Feature/Customer/ReviewStepTest | planned |
| R-02-10 | الدفع إلكترونيًا. | Pay electronically. | 2 | `StartPaymentAttempt` → gateway checkout (`PaymentGateway::createCheckout`) | tests/Feature/Payments/StartPaymentAttemptTest | planned |
| R-02-11 | الحصول على رقم Order واضح بعد نجاح الدفع. | Receive a clear order number after successful payment. | 2 | `ConfirmOrderPayment` assigns `R` number; confirmation page | tests/Feature/Payments/ConfirmOrderPaymentTest (number only after paid) | planned |
| R-02-12 | يفضل إصدار QR Code مرتبط بالطلب لتسهيل الاستلام. | Preferably issue a QR code linked to the order to ease pickup (mandatory per brief). | 2 | `QrCodeService`; QR → `/o/{uuid}`; shown on confirmation and status page | tests/Feature/Customer/OrderQrTest | planned |
| R-03-01 | جميع الطلبات Prepaid فقط. | All orders are prepaid only. | 2 | orders start `pending_payment`; no cash paths exist in any channel | tests/Feature/Payments/PrepaidOnlyTest | planned |
| R-03-02 | لا نريد: Cash on Delivery / Pay on Pickup / الدفع عند الاستلام. | No cash on delivery, pay on pickup or payment at collection. | 2 | no such `PaymentMethod` case; POS/kiosk have no cash option | tests/Unit/Enums/PaymentMethodTest, tests/Feature/Pos/PosNoCashTest (Phase 3) | planned |
| R-03-03 | ولا يتم تأكيد الطلب أو إرساله للتحضير إلا بعد نجاح عملية الدفع. | The order is neither confirmed nor sent for preparation before payment succeeds. | 2 | only `ConfirmOrderPayment` moves `pending_payment → new`; KDS query filters `payment_status = paid` | tests/Feature/Payments/ConfirmOrderPaymentTest, tests/Feature/Kds/KdsBoardTest (Phase 3) | planned |
| R-03-04 | Apple Pay. | Apple Pay. | 2 | `PaymentMethod::ApplePay`; driver capability; `.well-known` domain-association route | tests/Feature/Payments/EnabledMethodsTest, tests/Feature/Payments/ApplePayDomainFileTest | planned |
| R-03-05 | Samsung Wallet / Samsung Pay إذا كانت بوابة الدفع المستخدمة تدعمه. | Samsung Wallet / Samsung Pay if the gateway supports it. | 2 | `PaymentMethod::SamsungPay` as optional driver capability | tests/Feature/Payments/EnabledMethodsTest | planned |
| R-03-06 | الدفع الإلكتروني بالبطاقات عند الحاجة. | Electronic card payment when needed. | 2 | `PaymentMethod::Card` on the gateway-hosted page | tests/Feature/Payments/EnabledMethodsTest | planned |
| R-03-07 | الدفع/التحويل باستخدام رقم الهاتف إذا كانت الخدمة البنكية أو بوابة الدفع المستخدمة في عُمان توفر API مناسبًا لذلك. | Payment/transfer by phone number if an Omani bank service or gateway offers a suitable API. | 2 | `PaymentMethod::PhonePayment` optional capability; off until a vendor API exists (`docs/technical-proposal.md` §7) | tests/Feature/Payments/EnabledMethodsTest (hidden when unsupported) | planned |
| R-03-08 | النظام يجب أن يحصل على تأكيد الدفع آليًا من مزود الدفع، وليس اعتمادًا على تأكيد يدوي من العميل. | Payment confirmation comes automatically from the provider, never from the customer. | 2 | `PaymentWebhookController` + `ProcessPaymentEvent`; `payments:reconcile` | tests/Feature/Payments/WebhookTest, tests/Feature/Payments/ReconcileTest | planned |
| R-04-01 | في الجامعات والفعاليات سيكون هناك iPad يستطيع الزبون الطلب منه بنفسه. | An iPad at the booth lets the customer order by themselves. | 3 | kiosk PWA + device pairing | tests/Feature/Kiosk/KioskOrderTest | planned |
| R-04-02 | بسيطة جدًا. | Kiosk UI: very simple. | 3 | `pages/kiosk/*` (start → menu → cart → details → pay → done) | tests/Feature/Kiosk/KioskPagesTest (Inertia component assertions) | planned |
| R-04-03 | سريعة. | Kiosk UI: fast. | 3 | cached menu, prefetching, minimal steps | tests/Feature/Kiosk/KioskMenuCacheTest | planned |
| R-04-04 | مناسبة للمس. | Kiosk UI: touch-friendly. | 3 | `KioskLayout.vue` large tap targets (layout review) | tests/Feature/Kiosk/KioskPagesTest | planned |
| R-04-05 | تعرض الصور والأسعار بوضوح. | Kiosk UI shows images and prices clearly. | 3 | `KioskProductCard.vue` (medialibrary conversions, `Money`) | tests/Feature/Kiosk/KioskMenuTest | planned |
| R-04-06 | تسمح باختيار الإضافات. | Kiosk allows choosing add-ons. | 3 | product customisation sheet (variants, add-ons, custom text) | tests/Feature/Kiosk/KioskOrderTest | planned |
| R-04-07 | تعرض السلة والإجمالي. | Kiosk shows the cart and the total. | 3 | `useCart` running total; `CartDrawer.vue` | tests/Feature/Kiosk/KioskOrderTest | planned |
| R-04-08 | تنفذ الدفع الإلكتروني. | Kiosk performs electronic payment. | 3 | pay-by-QR/link screen + Reverb wait (`useOrderChannel`) | tests/Feature/Kiosk/KioskPaymentTest | planned |
| R-04-09 | تصدر رقم Order بعد الدفع. | Kiosk issues an order number after payment. | 3 | done screen after `OrderPaid` broadcast | tests/Feature/Kiosk/KioskPaymentTest | planned |
| R-04-10 | طلبات الـKiosk وطلبات الموقع يجب أن تدخل نفس النظام المركزي. | Kiosk and website orders enter the same central system. | 3 | `orders.source` kiosk/website; same `PlaceOrder` | tests/Feature/Orders/OrderSourcesTest | planned |
| R-05-01 | نحتاج أيضًا واجهة منفصلة للموظف. | A separate staff interface. | 3 | `routes/pos.php`, `pages/pos/*`, `PosLayout.vue` | tests/Feature/Pos/PosAccessTest | planned |
| R-05-02 | إذا كان العميل لا يريد استخدام جهاز الطلب الذاتي، يستطيع الموظف إدخال الطلب له. | Staff can enter an order for a customer who does not want the kiosk. | 3 | `PosOrderController@store` → `PlaceOrder` (source staff, `created_by_user_id`) | tests/Feature/Pos/PosOrderTest | planned |
| R-05-03 | الدفع المسبق إلزامي قبل اعتماد الطلب. | Prepayment is mandatory before the order is approved. | 3 | POS pay-by-QR; order stays `pending_payment` until the webhook | tests/Feature/Pos/PosPaymentTest | planned |
| R-06-01 | Order Number. | Every order stores: Order number. | 1 | `orders.number` (`R` + sequence) | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-02 | اسم العميل. | Every order stores: Customer name. | 1 | `orders.customer_name` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-03 | رقم الهاتف. | Every order stores: Phone number. | 1 | `orders.customer_phone` (E.164) | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-04 | المنتجات. | Every order stores: Products. | 1 | `order_items` with product snapshot | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-05 | الكمية. | Every order stores: Quantity. | 1 | `order_items.quantity` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-06 | الإضافات. | Every order stores: Add-ons. | 1 | `order_item_addons` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-07 | الرسالة/الاسم المخصص إن وجد. | Every order stores: Custom message/name if any. | 1 | `order_items.custom_text` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-08 | المبلغ الإجمالي. | Every order stores: Total amount. | 1 | `orders.total_baisa` (+ subtotal/discount) | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-09 | طريقة الدفع. | Every order stores: Payment method. | 1 | `orders.payment_method` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-10 | حالة الدفع. | Every order stores: Payment status. | 1 | `orders.payment_status` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-11 | موقع الاستلام. | Every order stores: Pickup location. | 1 | `orders.location_id` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-12 | تاريخ ووقت الاستلام. | Every order stores: Pickup date and time. | 1 | `orders.pickup_at` / `pickup_date` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-13 | مصدر الطلب: Website / Kiosk / Staff. | Every order stores: Order source: Website / Kiosk / Staff. | 1 | `orders.source` (`OrderSource`) | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-14 | وقت إنشاء الطلب. | Every order stores: Order creation time. | 1 | `orders.created_at` | tests/Feature/Orders/OrderSchemaTest (factory round-trip of every field) | planned |
| R-06-15 | حالات الطلب: New → Preparing → Ready → Collected | Order statuses New → Preparing → Ready → Collected. | 1 | `OrderStatus` enum + `transitions()` | tests/Unit/Enums/OrderStatusTest | planned |
| R-06-16 | بالعربية: جديد → قيد التحضير → جاهز → تم الاستلام | Arabic labels for the statuses. | 1 | `OrderStatus::label('ar')`; `lang/ar.json` | tests/Unit/Enums/OrderStatusTest (Arabic labels) | planned |
| R-07-01 | نحتاج شاشة خاصة بالموظفين لعرض الطلبات القادمة. | A staff screen that shows incoming orders (KDS). | 3 | `routes/kds.php`, `pages/kds/Board.vue` | tests/Feature/Kds/KdsBoardTest | planned |
| R-07-02 | أولوية التحضير. | KDS sorted by preparation priority. | 3 | `orders.priority` desc first in `KdsBoardQuery` | tests/Feature/Kds/KdsSortingTest | planned |
| R-07-03 | وقت الاستلام. | …then by pickup time. | 3 | `orders.pickup_at` asc | tests/Feature/Kds/KdsSortingTest | planned |
| R-07-04 | رقم الطلب. | …then by order number. | 3 | `orders.number` asc | tests/Feature/Kds/KdsSortingTest | planned |
| R-07-05 | ويستطيع الموظف تغيير حالة الطلب مباشرة من الشاشة. | Staff change the order status directly from the screen. | 3 | `KdsOrderStatusController` → `TransitionOrderStatus` | tests/Feature/Kds/KdsTransitionTest | planned |
| R-07-06 | مثال: #R1045 – استلام 12:30 PM – Preparing ثم Ready | Card line format “#R1045 – pickup 12:30 PM – Preparing” then “Ready”. | 3 | `KdsCardPresenter` (number, pickup time in location timezone, status label) | tests/Unit/Presenters/KdsCardPresenterTest | planned |
| R-08-01 | النظام يجب أن يدعم الطباعة المباشرة. | The system supports direct printing. | 3 | `print_jobs`, `PrinterDriver`, `BrowserPrintDriver`, auto-print flags | tests/Feature/Printing/AutoPrintTest | planned |
| R-08-02 | أ. فاتورة الطلب: تحتوي على معلومات الطلب الكاملة والسعر وطريقة الدفع. | Order invoice/receipt with full order information, price and payment method. | 3 | `print/receipt-80mm.blade.php` (`ReceiptRenderer`) | tests/Feature/Printing/ReceiptViewTest | planned |
| R-08-03 | ب. Product Sticker: ستيكر صغير يمكن لصقه على الكوب أو العلبة، ويحتوي بحسب نوع المنتج على: | Product sticker for cup/box with content depending on product type. | 3 | `print/sticker.blade.php`, `sticker_fields` per category/product, one job per unit | tests/Feature/Printing/StickerViewTest | planned |
| R-08-04 | رقم الطلب. | Sticker shows: Order number. | 3 | `StickerField` enum case; `StickerRenderer` | tests/Feature/Printing/StickerFieldsTest | planned |
| R-08-05 | اسم العميل. | Sticker shows: Customer name. | 3 | `StickerField` enum case; `StickerRenderer` | tests/Feature/Printing/StickerFieldsTest | planned |
| R-08-06 | اسم المنتج. | Sticker shows: Product name. | 3 | `StickerField` enum case; `StickerRenderer` | tests/Feature/Printing/StickerFieldsTest | planned |
| R-08-07 | الإضافات. | Sticker shows: Add-ons. | 3 | `StickerField` enum case; `StickerRenderer` | tests/Feature/Printing/StickerFieldsTest | planned |
| R-08-08 | الاسم أو الرسالة الخاصة. | Sticker shows: Custom name or message. | 3 | `StickerField` enum case; `StickerRenderer` | tests/Feature/Printing/StickerFieldsTest | planned |
| R-08-09 | ويجب مناقشة توافق النظام مع Thermal Receipt Printers وLabel Printers قبل اختيار أجهزة الطباعة. | Printer compatibility (thermal receipt vs label printers) is discussed before hardware is chosen. | 3 | `docs/printing.md` (Phase 3) and `docs/technical-proposal.md` §9 now | owner review of `docs/printing.md` | planned |
| R-09-01 | نريد إمكانية ربط النظام بخدمة WhatsApp Business المناسبة لإرسال إشعارات تلقائية. | Integrate a suitable WhatsApp Business service for automatic notifications. | 4 | `WhatsAppNotifier` contract, `LogWhatsAppNotifier`, `SendWhatsAppMessage` job | tests/Feature/WhatsApp/NotificationDispatchTest | planned |
| R-09-02 | تم تأكيد الطلب والدفع. | Message when the order is confirmed and paid. | 4 | `MessageTemplateKey::OrderConfirmed` queued from `ConfirmOrderPayment` | tests/Feature/WhatsApp/OrderConfirmedMessageTest | planned |
| R-09-03 | طلبك #R1045 جاهز للاستلام من Rofliv. | Message when the order is ready: “Your order #R1045 is ready for pickup from Rofliv”. | 4 | `MessageTemplateKey::OrderReady` queued on transition to ready | tests/Feature/WhatsApp/OrderReadyMessageTest | planned |
| R-09-04 | ويفضل أن تكون الرسائل والإشعارات قابلة للإدارة من لوحة التحكم. | Messages manageable from the dashboard (mandatory per brief). | 4 | `message_templates` admin (AR/EN, enable/disable, placeholders) | tests/Feature/Admin/MessageTemplateCrudTest | planned |
| R-10-01 | يجب أن أستطيع إنشاء موقع/فعالية جديدة، مثل: Rofliv – Sohar University | Create a new location/event, e.g. “Rofliv – Sohar University”. | 1 | `Admin/LocationController`, `StoreLocationRequest`, `CreateLocation` action | tests/Feature/Admin/LocationCrudTest | planned |
| R-10-02 | اسم الموقع. | Location setting: Location name. | 1 | `name_ar`/`name_en` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-03 | عنوان/وصف الموقع. | Location setting: Address/description. | 1 | `address_*`, `description_*` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-04 | تاريخ بداية ونهاية الفعالية. | Location setting: Event start and end dates. | 1 | `event_starts_on`, `event_ends_on` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-05 | أيام العمل. | Location setting: Working days. | 1 | `location_schedules.day_of_week` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-06 | ساعات العمل. | Location setting: Working hours. | 1 | `location_schedules.opens_at/closes_at` + exceptions | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-07 | أوقات الاستلام. | Location setting: Pickup times. | 1 | `slot_length_minutes`, `min_lead_minutes`, generated `pickup_slots` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-08 | المنيو المتاح. | Location setting: Available menu. | 1 | `location_product.is_listed` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-09 | المخزون. | Location setting: Inventory. | 1 | `location_product.stock_quantity` | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-10 | الطاقة الاستيعابية. | Location setting: Capacity. | 1 | `default_slot_capacity` (+ per-slot override) | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-11 | حالة الموقع: Active / Inactive. | Location setting: Location status: Active / Inactive. | 1 | `locations.status` (`LocationStatus`) | tests/Feature/Admin/LocationCrudTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-10-12 | بعد انتهاء الفعالية أستطيع إغلاق الموقع بدون حذفه من النظام. | After the event the location can be closed without deleting it. | 1 | status → inactive; no destroy route; history kept | tests/Feature/Admin/LocationDeactivationTest | planned |
| R-11-01 | لا نريد أن يستطيع عدد غير محدود من الأشخاص اختيار نفس الوقت. | Unlimited customers must not pick the same time. | 2 | `pickup_slots.reserved_count` vs capacity under lock | tests/Feature/Slots/CapacityTest, tests/Feature/Concurrency/SlotCapacityTest | planned |
| R-11-02 | يجب أن يكون هناك Order Capacity per Time Slot. مثال: 12:00–12:15 → Maximum 15 Orders | Order capacity per time slot (e.g. 12:00–12:15 → maximum 15 orders). | 2 | `pickup_slots.capacity` (NULL = location default) | tests/Feature/Slots/CapacityTest | planned |
| R-11-03 | إذا وصل العدد إلى 15، يغلق النظام الفترة تلقائيًا ويطلب من العميل اختيار الفترة التالية. | When full, the slot closes automatically and the customer is asked to choose the next one. | 2 | `SlotUnavailableException` with suggested next slot; slot hidden when full | tests/Feature/Slots/FullSlotSuggestionTest | planned |
| R-11-04 | ويجب أن يستطيع المدير تعديل الطاقة الاستيعابية لكل موقع وفعالية. | Managers can edit capacity per location/event. | 2 | `Admin/SlotController` (`UpdateSlotCapacity`), location default capacity | tests/Feature/Admin/SlotCapacityAdminTest | planned |
| R-12-01 | إضافة/تعديل/حذف المنتجات. | Add/edit/delete products. | 1 | `Admin/ProductController` (soft delete) | tests/Feature/Admin/ProductCrudTest | planned |
| R-12-02 | إضافة الصور والوصف والأسعار. | Add images, description and prices. | 1 | medialibrary images, `description_*`, `price_baisa` | tests/Feature/Admin/ProductCrudTest | planned |
| R-12-03 | إنشاء Categories. | Create categories. | 1 | `Admin/CategoryController` | tests/Feature/Admin/CategoryCrudTest | planned |
| R-12-04 | إضافة Variants وAdd-ons. | Add variants and add-ons. | 1 | `product_variants`, `addon_groups`/`addons` admin | tests/Feature/Admin/VariantAddonCrudTest | planned |
| R-12-05 | تحديد المنتجات المتوفرة في كل فعالية. | Set the products available in each event. | 1 | `location_product.is_listed` page | tests/Feature/Admin/LocationMenuStockTest | planned |
| R-12-06 | تحديد كمية المخزون لكل بوث بشكل مستقل. | Set stock quantity per booth independently. | 1 | `location_product.stock_quantity` | tests/Feature/Admin/LocationMenuStockTest | planned |
| R-12-07 | إيقاف المنتج يدويًا. | Stop a product manually. | 1 | `location_product.is_available` / `products.is_active` | tests/Feature/Admin/LocationMenuStockTest | planned |
| R-12-08 | جعل المنتج يظهر Sold Out عند نفاده. | Show the product as Sold Out when out of stock. | 1 | `stock_quantity = 0` → Sold Out badge (shown, not hidden) | tests/Feature/Customer/MenuVisibilityTest (Phase 2) | planned |
| R-12-09 | إعادة تفعيله بسهولة. | Re-enable it easily. | 1 | one-click restock / enable (`RestockProduct`) | tests/Feature/Admin/LocationMenuStockTest | planned |
| R-13-01 | كل فعالية/جامعة يمكن أن يكون لها QR Code خاص. | Each event/university can have its own QR code. | 1 | `GenerateLocationQr` → `/l/{slug}`; PNG/SVG download in admin | tests/Feature/Admin/LocationQrTest | planned |
| R-13-02 | عند مسحه يفتح مباشرة: Rofliv → Sohar University Menu بدون أن يضطر العميل للبحث عن موقع الاستلام. | Scanning opens that location's menu directly without searching for the pickup location. | 2 | `GET /l/{location:slug}` preselects the location | tests/Feature/Customer/LocationSlugMenuTest | planned |
| R-14-01 | إجمالي مبيعات اليوم. | Dashboard shows: Today's total sales. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-02 | إجمالي عدد الطلبات. | Dashboard shows: Total number of orders. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-03 | Average Order Value. | Dashboard shows: Average order value. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-04 | المبيعات حسب الموقع. | Dashboard shows: Sales by location. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-05 | المبيعات حسب الفعالية. | Dashboard shows: Sales by event. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-06 | المنتجات الأكثر مبيعًا. | Dashboard shows: Best-selling products. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-07 | الطلبات الحالية. | Dashboard shows: Current orders. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-08 | الطلبات القادمة. | Dashboard shows: Upcoming orders. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-09 | الطلبات الجاهزة. | Dashboard shows: Ready orders. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-10 | طرق الدفع. | Dashboard shows: Payment methods. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-11 | الطلبات حسب Website / Kiosk. | Dashboard shows: Orders by Website / Kiosk (and Staff per brief). | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-12 | مقارنة الفعاليات والمواقع. | Dashboard shows: Comparison of events and locations. | 4 | `DashboardMetrics` service + `pages/admin/Dashboard.vue` | tests/Feature/Reports/DashboardMetricsTest | planned |
| R-14-13 | ويفضل إمكانية تصدير التقارير إلى Excel/CSV. | Export reports to Excel/CSV (mandatory per brief). | 4 | `app/Exports/*`, `data_exports`, `GenerateDataExport` job | tests/Feature/Reports/ExportsTest | planned |
| R-15-01 | Discount Codes. | Discount codes. | 2 | `coupons.code`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-02 | نسبة خصم. | Percentage discount. | 2 | `CouponType::Percentage`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-03 | مبلغ خصم ثابت. | Fixed-amount discount. | 2 | `CouponType::Fixed`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-04 | تاريخ بداية وانتهاء. | Start and end dates. | 2 | `starts_at`/`ends_at`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-05 | حد أقصى للاستخدام. | Maximum number of uses. | 2 | `max_uses` + `uses_count`/`reserved_count`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-06 | كوبون خاص بجامعة أو فعالية. | Coupon restricted to a university/event. | 2 | `coupons.location_id`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-07 | Minimum Order Value. | Minimum order value. | 2 | `min_order_total_baisa`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-15-08 | تفعيل وإيقاف الكوبون. | Activate/deactivate the coupon. | 2 | `is_active`; `CouponValidator`; `Admin/CouponController` | tests/Feature/Coupons/CouponValidationTest, tests/Feature/Admin/CouponCrudTest | planned |
| R-16-01 | نحتاج Accounts وصلاحيات مختلفة | Accounts with different permissions. | 1 | spatie roles + `location_user`; `Admin/UserController` | tests/Feature/Admin/UserCrudTest | planned |
| R-16-02 | Owner / Admin: وصول كامل للنظام والتقارير والمبيعات. | Owner/Admin: full access to the system, reports and sales. | 1 | roles `owner`, `admin`; `Gate::before` | tests/Feature/Auth/RoleMatrixTest | planned |
| R-16-03 | Manager: إدارة الموقع والطلبات والمخزون. | Manager: manages the location, orders and inventory. | 1 | role `manager` scoped by `location_user`; policies | tests/Feature/Auth/RoleMatrixTest, tests/Feature/Auth/LocationScopeTest | planned |
| R-16-04 | Staff: مشاهدة وتنفيذ الطلبات فقط. | Staff: view and process orders only. | 1 | role `staff`; POS + mark collected only | tests/Feature/Auth/RoleMatrixTest | planned |
| R-16-05 | Kitchen/Preparation: مشاهدة شاشة التحضير وتغيير حالة الطلب. | Kitchen: view the KDS and change order status. | 1 | role `kitchen`; KDS routes only | tests/Feature/Auth/RoleMatrixTest | planned |
| R-17-01 | النظام يجب ألا يكون مربوطًا بموقع واحد. | The system must not be tied to one location. | 1 | every query scoped by `location_id`; no single-location assumptions | tests/Feature/Orders/MultiLocationIsolationTest | planned |
| R-17-02 | الثلاثة يعملون في الوقت نفسه (Booth 1 – Sohar University, Booth 2 – Muscat Event, Booth 3 – Nizwa University). | Several booths run at the same time. | 1 | several `locations` rows active simultaneously | tests/Feature/Admin/LocationCrudTest (three active locations) | planned |
| R-17-03 | Orders مستقلة. | Per booth: Independent orders. | 1 | `orders.location_id` | tests/Feature/Orders/MultiLocationIsolationTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-17-04 | Inventory مستقل. | Per booth: Independent inventory. | 1 | `location_product.stock_quantity` per location | tests/Feature/Orders/MultiLocationIsolationTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-17-05 | Menu يمكن تخصيصه. | Per booth: Customisable menu. | 1 | `location_product.is_listed` / price override | tests/Feature/Orders/MultiLocationIsolationTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-17-06 | ساعات عمل مستقلة. | Per booth: Independent working hours. | 1 | `location_schedules` per location | tests/Feature/Orders/MultiLocationIsolationTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-17-07 | موظفون مستقلون. | Per booth: Independent staff. | 1 | `location_user` | tests/Feature/Orders/MultiLocationIsolationTest, tests/Feature/Admin/LocationMenuStockTest | planned |
| R-17-08 | لكن المالك يستطيع مشاهدة كل شيء من Dashboard واحدة. | The owner sees everything from one dashboard. | 4 | admin dashboard with location filter “all” | tests/Feature/Reports/DashboardMetricsTest (owner sees all locations) | planned |
| R-18-01 | Responsive للموبايل والتابلت والكمبيوتر. | Responsive for mobile, tablet and desktop. | 2–3 | Tailwind responsive layouts per channel | manual device check + Inertia page tests | planned |
| R-18-02 | سريع جدًا في عملية الطلب. | Very fast ordering. | 2 | cached menu, minimal steps, eager loading, indexes | tests/Feature/Customer/MenuQueryCountTest | planned |
| R-18-03 | Cloud-Based. | Cloud-based. | 1 | stateless app, Redis, S3 media (`docs/technical-proposal.md` §5) | deployment checklist | planned |
| R-18-04 | آمن في عمليات الدفع. | Secure payments. | 2 | signature-verified webhooks, amount/currency checks, HTTPS, no card data | tests/Feature/Payments/WebhookSignatureTest | planned |
| R-18-05 | قابلًا للتوسع. | Scalable. | 1 | queues/Horizon, Reverb scaling, stateless servers | tests/Feature/Infrastructure/QueueConfigTest | planned |
| R-18-06 | يحتوي Backup مناسب. | Proper backup. | 4 | spatie/laravel-backup daily DB + storage off-site, retention, failure notifications | tests/Feature/Infrastructure/BackupConfigTest | planned |
| R-18-07 | قابلًا لإضافة مواقع جديدة مستقبلًا. | New locations can be added in future. | 1 | locations are data; no code per location | tests/Feature/Admin/LocationCrudTest | planned |
| R-18-08 | سهل الاستخدام للموظفين. | Easy for staff to use. | 3 | POS/KDS one-tap flows | tests/Feature/Kds/KdsTransitionTest, tests/Feature/Pos/PosOrderTest | planned |
| R-18-09 | قابلًا للتكامل مع Payment Gateway وWhatsApp والطابعات. | Integrable with payment gateway, WhatsApp and printers. | 2–4 | `PaymentGateway`, `WhatsAppNotifier`, `PrinterDriver` contracts + managers | tests/Unit/Integrations/ManagerResolutionTest | planned |
| R-18-10 | لا يخزن بيانات البطاقات البنكية الحساسة داخل نظام Rofliv نفسه؛ معالجة الدفع تتم عبر مزود الدفع المعتمد. | No sensitive card data stored in Rofliv; payment processed by the approved provider. | 2 | hosted gateway pages only; sanitised encrypted webhook payloads | tests/Feature/Payments/NoCardDataTest | planned |
| R-19-01 | التقنيات المقترحة | Proposed technologies. | 0 | `docs/technical-proposal.md` §2 | n/a — document; owner review | done (document) |
| R-19-02 | مدة التطوير | Development duration. | 0 | `docs/technical-proposal.md` §3 | n/a — document; owner review | done (document) |
| R-19-03 | تكلفة المشروع | Project cost. | 0 | `docs/technical-proposal.md` §4 (drivers listed; figure TO CONFIRM — developer quote) | n/a — document; owner review | done (document) |
| R-19-04 | تكلفة الاستضافة السنوية | Annual hosting cost. | 0 | `docs/technical-proposal.md` §4–§5 (published prices cited; rest TO CONFIRM WITH VENDOR) | n/a — document; owner review | done (document) |
| R-19-05 | تكلفة الصيانة | Maintenance cost. | 0 | `docs/technical-proposal.md` §6 (scope; figure TO CONFIRM) | n/a — document; owner review | done (document) |
| R-19-06 | بوابة الدفع المقترحة في عُمان | Proposed payment gateway in Oman. | 0 | `docs/technical-proposal.md` §7 | n/a — document; owner review | done (document) |
| R-19-07 | رسومها | Its fees. | 0 | `docs/technical-proposal.md` §7 (only published fees; otherwise TO CONFIRM WITH VENDOR) | n/a — document; owner review | done (document) |
| R-19-08 | طريقة ربط Apple Pay ووسائل الدفع الأخرى | How Apple Pay and the other payment methods are linked. | 0 | `docs/technical-proposal.md` §7 | n/a — document; owner review | done (document) |
| R-19-09 | حل WhatsApp | WhatsApp solution. | 0 | `docs/technical-proposal.md` §8 | n/a — document; owner review | done (document) |
| R-19-10 | أنواع الطابعات المتوافقة | Compatible printer types. | 0 | `docs/technical-proposal.md` §9 | n/a — document; owner review | done (document) |
| R-19-11 | أي رسوم API أو خدمات خارجية شهرية/أسنوية | Any monthly/annual API or external service fees. | 0 | `docs/technical-proposal.md` §10 | n/a — document; owner review | done (document) |
| R-19-12 | هل النظام والكود وقاعدة البيانات ستكون مملوكة بالكامل لـRofliv بعد التسليم | Whether the system, code and database are fully owned by Rofliv after delivery. | 0 | `docs/technical-proposal.md` §11 | n/a — document; owner review | done (document) |
| R-20-01 | الهدف النهائي: منصة واحدة قابلة للتوسع، وليس موقع طلبات مؤقتًا؛ طلب ودفع مسبق من الموقع مع استلام في الموعد، أو طلب مباشر من iPad في أي بوث، وإدارة كل الطلبات والمدفوعات والطباعة والمخزون والفعاليات والتقارير من نظام مركزي واحد. | Final goal: one scalable platform (not a temporary ordering site) covering prepaid website orders with scheduled pickup, direct iPad ordering at any booth, and central management of orders, payments, printing, inventory, events and reports. | 1–4 | the platform as a whole | full suite green at the end of Phase 4 | planned |

## Part B — Elaborations from the owner's brief (beyond the Arabic document)

| ID | Requirement (EN) | Phase | Implementation (files / routes) | Test(s) | Status |
|---|---|---|---|---|---|
| B-01-01 | Five channels, each with its own Inertia layout: `/` and `/l/{location:slug}`, `/kiosk`, `/pos`, `/kds`, `/admin`. | 1 (admin) · 2 (customer) · 3 (kiosk, POS, KDS) | `resources/js/app.ts` layout resolver; `layouts/{customer,kiosk,pos,kds,admin}` | tests/Feature/Layouts/ChannelLayoutTest | planned |
| B-01-02 | Adding a new location requires no code changes. | 1 | locations are data; seeders/admin only | tests/Feature/Admin/LocationCrudTest | planned |
| B-01-03 | Ordering flow very fast: minimal steps, cached menus, eager loading, proper indexes, no N+1 queries. | 2 (+4 performance pass) | `MenuBuilder` cache, `preventLazyLoading`, indexes in `docs/db-schema.md` | tests/Feature/Customer/MenuQueryCountTest | planned |
| B-01-04 | Staff-facing screens simple and easy to use. | 3 | POS/KDS layouts | tests/Feature/Kds/KdsTransitionTest | planned |
| B-01-05 | Thin controllers; business logic in `app/Actions` and `app/Services`; validation in FormRequests; authorization in Policies. | 1 | folder structure in `docs/architecture.md` §2 | tests/Unit/ArchitectureTest | planned |
| B-01-06 | PHP backed enums `OrderStatus`, `PaymentStatus`, `PaymentMethod`, `OrderSource`, `LocationStatus`, `CouponType`, `PrintJobType`. | 1 | `app/Enums/*` | tests/Unit/ArchitectureTest (string-backed), tests/Unit/Enums/* | planned |
| B-01-07 | Money stored as integer baisa, displayed as OMR with 3 decimals; all prices calculated server-side. | 1 (Money) · 2 (pricing) | `App\Support\Money`, `*_baisa` columns, `CartPricer` | tests/Unit/Support/MoneyTest, tests/Feature/Customer/CartPricingTest | planned |
| B-01-08 | Bilingual Arabic/English with full RTL; customer screens default to Arabic; product, category, add-on and location names stored in both languages. | 1 | `*_ar`/`*_en` columns, `lang/*.json`, `SetLocale`, `useTrans`, `<html dir>` | tests/Feature/I18n/LocaleTest | planned |
| B-01-09 | App timezone Asia/Muscat (UTC storage). | 1 | `config/app.php`, `config/database.php` connection timezone, `BusinessDate` | tests/Unit/Support/BusinessDateTest | planned |
| B-01-10 | Validate Omani phone numbers and store E.164 `+968XXXXXXXX`. | 1 | `App\Rules\OmaniPhone`, `App\Support\PhoneNumber` | tests/Unit/Rules/OmaniPhoneTest (dataset) | planned |
| B-02-01 | Show only active locations within their event dates. | 2 | `Location::scopeOrderable()` (status, ordering_opens_on, event_ends_on) | tests/Feature/Customer/LocationListingTest | planned |
| B-02-02 | Menu shows clear images, prices and “Sold Out” badges. | 2 | `MenuPresenter`, medialibrary conversions | tests/Feature/Customer/MenuVisibilityTest | planned |
| B-02-03 | Add-on groups enforce min/max selection. | 2 | `addon_groups.min_select/max_select`, `CartPricer` | tests/Unit/Pricing/AddonGroupRulesTest | planned |
| B-02-04 | Custom text is a per-product flag with a maximum length. | 1 (fields) · 2 (enforcement) | `products.allows_custom_text`, `custom_text_max_length` | tests/Feature/Customer/CustomTextTest | planned |
| B-02-05 | Slots fall within event dates, working days and hours, respect the per-location minimum lead time, and exclude full or past slots. | 2 | `GeneratePickupSlots`, `SlotAvailability` | tests/Feature/Slots/SlotAvailabilityTest, tests/Feature/Slots/SlotGenerationTest | planned |
| B-02-06 | Name and phone are both required. | 2 | `StoreCustomerOrderRequest` | tests/Feature/Customer/CheckoutValidationTest | planned |
| B-02-07 | Apply a coupon code at checkout. | 2 | `CouponValidator`, checkout coupon field | tests/Feature/Coupons/ApplyCouponTest | planned |
| B-02-08 | Review shows items, variants, add-ons, custom text, pickup location, date, time, discount and total. | 2 | `pages/customer/Review.vue` | tests/Feature/Customer/ReviewStepTest | planned |
| B-02-09 | The order QR opens the status page and is scanned by staff at pickup. | 2 (status page) · 3 (scan) | `/o/{uuid}`, `useQrScanner` on POS | tests/Feature/Customer/OrderStatusPageTest, tests/Feature/Pos/PosScanTest | planned |
| B-02-10 | Order number and QR are issued only after successful payment. | 2 | `ConfirmOrderPayment` | tests/Feature/Payments/ConfirmOrderPaymentTest | planned |
| B-03-01 | Orders are created as `pending_payment`; not confirmed and not sent to KDS, printers or WhatsApp until the provider confirms payment. | 2 | `PlaceOrder`, `ConfirmOrderPayment`, afterCommit side effects | tests/Feature/Payments/SideEffectsOnlyAfterPaymentTest | planned |
| B-03-02 | Webhook verification: signature check, amount and currency check, idempotent handling. | 2 | `PaymentWebhookController`, `payment_events` unique(gateway, event_id) | tests/Feature/Payments/WebhookTest (signature, amount mismatch, duplicate) | planned |
| B-03-03 | Server-side status check fallback for missed webhooks. | 2 | `payments:reconcile`, `ReconcilePendingPayment` | tests/Feature/Payments/ReconcileTest | planned |
| B-03-04 | Never rely on the browser redirect or on the customer saying they paid. | 2 | `/pay/{uuid}` return page only reads state; no confirmation path from the client | tests/Feature/Payments/ReturnUrlDoesNotConfirmTest | planned |
| B-03-05 | `PaymentMethod` enum: apple_pay, samsung_pay, card, phone_payment. | 1 | `app/Enums/PaymentMethod.php` | tests/Unit/Enums/PaymentMethodTest | planned |
| B-03-06 | Each gateway driver declares supported methods; customers see only methods the active gateway supports AND the admin enabled. | 2 | `PaymentGateway::supportedMethods()`, `settings.payments.enabled_methods` | tests/Feature/Payments/EnabledMethodsTest | planned |
| B-03-07 | Apple Pay domain verification file served at `/.well-known/apple-developer-merchantid-domain-association`. | 2 | `WellKnown/AppleMerchantDomainController` (content from settings) | tests/Feature/Payments/ApplePayDomainFileTest | planned |
| B-03-08 | Samsung Pay and phone-number payment are optional capabilities enabled only if the gateway supports them. | 2 | driver capability flags | tests/Feature/Payments/EnabledMethodsTest | planned |
| B-03-09 | Card entry only on the gateway hosted page or SDK fields; no card data in Rofliv. | 2 | hosted checkout only; sanitised payloads | tests/Feature/Payments/NoCardDataTest | planned |
| B-03-10 | `PaymentGateway` interface (createCheckout, verifyWebhook, fetchStatus, refund) with a `FakeGateway` simulating success, failure and timeout. | 2 | `app/Contracts/PaymentGateway.php`, `app/Payments/Drivers/FakeGateway.php`, `/fake-gateway/{session}` | tests/Feature/Payments/FakeGatewayTest | planned |
| B-03-11 | Kiosk and POS show a payment QR/link the customer pays on their own phone; the screen listens via Reverb; card-terminal/SoftPOS driver can be added later. | 3 | pay-by-QR screen, `useOrderChannel`, driver abstraction | tests/Feature/Kiosk/KioskPaymentTest, tests/Feature/Pos/PosPaymentTest | planned |
| B-04-01 | Kiosk offers variants, add-ons and custom text. | 3 | kiosk customisation sheet | tests/Feature/Kiosk/KioskOrderTest | planned |
| B-04-02 | Arabic/English toggle on the kiosk. | 3 | `useLocale` toggle persisted per device | tests/Feature/Kiosk/KioskLocaleTest | planned |
| B-04-03 | Auto-reset to the start screen after inactivity. | 3 | `useIdleReset` + `AbandonPendingOrder` | tests/Feature/Kiosk/KioskAbandonTest | planned |
| B-04-04 | Kiosk locked to one location via a device token (`devices` table). | 1 (table, admin pairing) · 3 (guard) | `devices`, `PairDevice`, `device` guard, `AuthenticateDevice` | tests/Feature/Admin/DeviceCrudTest, tests/Feature/Kiosk/DevicePairingTest | planned |
| B-04-05 | Installable as a PWA for iPad Guided Access. | 3 | `vite-plugin-pwa` manifest + service worker | tests/Feature/Kiosk/PwaManifestTest | planned |
| B-04-06 | Kiosk collects customer name and phone. | 3 | kiosk details step | tests/Feature/Kiosk/KioskOrderTest | planned |
| B-04-07 | Kiosk orders: source = kiosk, pickup = as soon as possible. | 3 | `OrderSource::Kiosk`, `is_asap`, `pickup_at = created_at` | tests/Feature/Kiosk/KioskOrderTest | planned |
| B-05-01 | POS uses the same menu, stock and pricing rules as the kiosk. | 3 | shared `PlaceOrder`/`CartPricer` | tests/Feature/Pos/PosOrderTest | planned |
| B-05-02 | POS orders: source = staff; the creating staff user is recorded. | 3 | `OrderSource::Staff`, `orders.created_by_user_id` | tests/Feature/Pos/PosOrderTest | planned |
| B-05-03 | Staff scan an order QR with the device camera (or search by order number) to open the order and mark it Collected. | 3 | `pages/pos/Scan.vue` (`@zxing/browser`), order search, `MarkOrderCollected` | tests/Feature/Pos/PosScanTest, tests/Feature/Pos/MarkCollectedTest | planned |
| B-06-01 | Orders store variants in addition to add-ons. | 1 | `order_items.product_variant_id` + snapshot | tests/Feature/Orders/OrderSchemaTest | planned |
| B-06-02 | Orders store subtotal, discount and total. | 1 | `subtotal_baisa`, `discount_baisa`, `total_baisa` | tests/Feature/Orders/OrderSchemaTest | planned |
| B-06-03 | Internal statuses `pending_payment`, `expired`, `cancelled`. | 1 | `OrderStatus` | tests/Unit/Enums/OrderStatusTest | planned |
| B-06-04 | Valid transitions enforced in one place; every change recorded (who and when) in `order_status_histories`. | 2 | `TransitionOrderStatus`, `OrderStatus::transitions()` | tests/Feature/Orders/StatusTransitionTest | planned |
| B-06-05 | Order numbers “R” + sequential number, unique and safe under concurrency. | 2 | `order_number_sequences` locked in `ConfirmOrderPayment` | tests/Feature/Orders/OrderNumberTest, tests/Feature/Concurrency/OrderNumberSequenceTest | planned |
| B-06-06 | Slot capacity and stock reserved in a DB transaction with `lockForUpdate` at order creation. | 2 | `PlaceOrder` (T1) | tests/Feature/Concurrency/PlaceOrderConcurrencyTest | planned |
| B-06-07 | Scheduled job expires orders unpaid after 10 minutes and releases their slot and stock. | 2 | `orders:expire-unpaid`, `ExpireUnpaidOrder` | tests/Feature/Orders/ExpiryTest | planned |
| B-07-01 | KDS is real-time via Reverb with auto-reconnect. | 3 | `location.{id}.orders` channel, `useEcho` | tests/Feature/Kds/KdsBroadcastTest | planned |
| B-07-02 | Orders have a priority field managers can raise. | 3 | `orders.priority`, `SetOrderPriority` | tests/Feature/Kds/PriorityTest | planned |
| B-07-03 | Kiosk and staff (ASAP) orders use their creation time as pickup time. | 2 | `PlaceOrder` sets `pickup_at = created_at` when `is_asap` | tests/Feature/Orders/AsapPickupTimeTest | planned |
| B-07-04 | Each KDS card shows items, variants, add-ons and custom text. | 3 | `KdsCardPresenter` | tests/Feature/Kds/KdsBoardTest | planned |
| B-07-05 | One-tap status changes from the screen; moving to Ready triggers the WhatsApp “ready” message. | 3 (+4 message) | `KdsOrderStatusController`, listener on `OrderStatusChanged` | tests/Feature/Kds/KdsTransitionTest, tests/Feature/WhatsApp/OrderReadyMessageTest | planned |
| B-08-01 | Auto-print when an order is confirmed paid (per-location setting) plus a manual reprint button. | 3 | `locations.auto_print_receipt/auto_print_stickers`, `EnqueueAutoPrintJobs`, `ReprintOrder` | tests/Feature/Printing/AutoPrintTest, tests/Feature/Printing/ReprintTest | planned |
| B-08-02 | Receipt includes items, add-ons, prices, discount, total, payment method and status, pickup location, date and time, and the order QR. | 3 | `ReceiptRenderer` | tests/Feature/Printing/ReceiptViewTest | planned |
| B-08-03 | One sticker per item unit. | 3 | one `print_jobs` row per unit (`dedupe_key auto:sticker:{item}:{unit}`) | tests/Feature/Printing/StickerPerUnitTest | planned |
| B-08-04 | Sticker fields configurable per product or category. | 3 | `sticker_fields` json (`StickerField` enum) | tests/Feature/Printing/StickerFieldsTest | planned |
| B-08-05 | `print_jobs` table (location, device, type, payload, status) and a `PrinterDriver` interface. | 1 (table) · 3 (driver) | `docs/db-schema.md`, `app/Contracts/PrinterDriver.php`, `PrinterManager` | tests/Feature/Printing/PrintJobLifecycleTest | planned |
| B-08-06 | HTML print views now: 80 mm receipt and a label with configurable size (e.g. 50×30 mm); browser-print fallback. | 3 | `print/receipt-80mm.blade.php`, `print/sticker.blade.php`, `BrowserPrintDriver` | tests/Feature/Printing/ReceiptViewTest, tests/Feature/Printing/StickerViewTest | planned |
| B-08-07 | Designed for ESC/POS receipt printers and label printers, with Star CloudPRNT and Epson ePOS as later drivers. | 3 | `PrinterDriverType` enum, `printers.connection` | tests/Unit/Integrations/ManagerResolutionTest | planned |
| B-08-08 | `docs/printing.md` compares thermal receipt and label printer options and connection methods before hardware is chosen. | 3 | `docs/printing.md` | owner review | planned |
| B-09-01 | `WhatsAppNotifier` interface: log driver now, Meta WhatsApp Business Cloud API driver later. | 4 | `app/Contracts/WhatsAppNotifier.php`, `LogWhatsAppNotifier`, `WhatsAppManager` | tests/Feature/WhatsApp/LogDriverTest | planned |
| B-09-02 | Messages managed from the dashboard: AR/EN text, enable/disable per type, placeholders {order_number}, {customer_name}, {location}, {pickup_time}. | 4 | `message_templates`, `TemplateRenderer` | tests/Unit/Services/TemplateRendererTest, tests/Feature/Admin/MessageTemplateCrudTest | planned |
| B-09-03 | Sent via queued jobs with retries and logged in `notification_logs` with delivery status. | 4 | `SendWhatsAppMessage` (tries/backoff), `notification_logs` | tests/Feature/WhatsApp/NotificationDispatchTest | planned |
| B-10-01 | Location has a slug. | 1 | `locations.slug` | tests/Feature/Admin/LocationCrudTest | planned |
| B-10-02 | Pickup slot length and minimum lead time per location. | 1 | `slot_length_minutes`, `min_lead_minutes` | tests/Feature/Admin/LocationCrudTest | planned |
| B-10-03 | Default slot capacity per location. | 1 | `default_slot_capacity` | tests/Feature/Admin/LocationCrudTest | planned |
| B-10-04 | Assigned staff per location. | 1 | `location_user` (+ role override), `AssignStaff` | tests/Feature/Admin/LocationStaffTest | planned |
| B-10-05 | Auto-print setting per location. | 1 | `auto_print_receipt`, `auto_print_stickers` | tests/Feature/Admin/LocationCrudTest | planned |
| B-10-06 | Inactive locations are hidden from customers and kiosks; history and reports remain. | 2 (customer) · 3 (kiosk) | `Location::scopeOrderable()`, `EnsureLocationIsActive` | tests/Feature/Customer/LocationListingTest, tests/Feature/Kiosk/InactiveLocationTest | planned |
| B-10-07 | Event QR downloadable as PNG/SVG for printing on the booth. | 1 | `Admin/LocationQrController` (bacon-qr-code) | tests/Feature/Admin/LocationQrTest | planned |
| B-11-01 | Slot capacity defaults to the location capacity and is overridable per slot. | 2 | `pickup_slots.capacity` NULL = inherit | tests/Feature/Slots/CapacityTest | planned |
| B-11-02 | When a slot is full the next available slot is suggested. | 2 | `SlotUnavailableException::suggestedSlot` | tests/Feature/Slots/FullSlotSuggestionTest | planned |
| B-11-03 | Managers can edit capacity per slot. | 2 | `Admin/SlotController` | tests/Feature/Admin/SlotCapacityAdminTest | planned |
| B-11-04 | No overbooking under simultaneous checkouts. | 2 | slot row `lockForUpdate` in `PlaceOrder` | tests/Feature/Concurrency/SlotCapacityTest (MySQL) | planned |
| B-11-05 | ASAP kiosk/staff orders do not count toward slot capacity by default (per-location toggle). Owner decision 2. | 2 | `locations.asap_counts_toward_capacity` | tests/Feature/Slots/AsapCapacityToggleTest | planned |
| B-12-01 | Products are soft-deleted. | 1 | `SoftDeletes` on catalogue models | tests/Feature/Admin/ProductCrudTest | planned |
| B-12-02 | Optional per-location price override. | 1 | `location_product.price_override_baisa` | tests/Feature/Admin/LocationMenuStockTest | planned |
| B-12-03 | Re-enable or restock in one click. | 1 | `RestockProduct` | tests/Feature/Admin/LocationMenuStockTest | planned |
| B-12-04 | Stock decremented on reservation and restored on expiry or cancellation. | 2 | `ReserveStock`/`ReleaseStock`, `stock_movements` | tests/Feature/Inventory/StockReservationTest, tests/Feature/Concurrency/StockTest | planned |
| B-14-01 | Orders by source include Staff. | 4 | `DashboardMetrics::ordersBySource()` | tests/Feature/Reports/DashboardMetricsTest | planned |
| B-14-02 | Dashboard has date-range and location filters. | 4 | `DashboardFiltersRequest` | tests/Feature/Reports/DashboardFiltersTest | planned |
| B-14-03 | Dashboard updates live. | 4 | `admin.dashboard` channel + `useEcho` | tests/Feature/Reports/DashboardBroadcastTest | planned |
| B-15-01 | Coupon validation is server-side and concurrency-safe; usage counted only when payment succeeds. | 2 | coupon row lock in T1/T2/T3; `coupon_redemptions` | tests/Feature/Coupons/CouponUsageCountingTest, tests/Feature/Concurrency/CouponMaxUsesTest | planned |
| B-15-02 | Coupon admin (create, edit, activate/deactivate). | 2 | `Admin/CouponController` | tests/Feature/Admin/CouponCrudTest | planned |
| B-16-01 | All non-admin roles are scoped to their assigned locations. | 1 | `location_user`, `EnsureLocationAccess`, policies | tests/Feature/Auth/LocationScopeTest (cross-location 404) | planned |
| B-16-02 | Manager scope covers location settings, slots, menu availability, inventory and orders. | 1 | policies for Location, Slot, LocationProduct, Order | tests/Feature/Auth/RoleMatrixTest | planned |
| B-18-01 | HTTPS only; webhook signature verification. | 2 | `URL::forceScheme` in production, `PaymentGateway::verifyWebhook` | tests/Feature/Payments/WebhookSignatureTest | planned |
| B-18-02 | Rate limiting on checkout, coupon and webhook endpoints. | 2 (+4 review) | named limiters in `AppServiceProvider` | tests/Feature/Security/RateLimitTest | planned |
| B-18-03 | Queues, caching, stateless app servers, media on S3-compatible storage. | 1 (config) · 4 (pass) | Redis/Horizon, medialibrary `s3` disk, `config/filesystems.php` | tests/Feature/Infrastructure/QueueConfigTest, tests/Feature/Admin/ProductImageUploadTest | planned |
| B-18-04 | spatie/laravel-backup: daily DB + media backup to off-site storage, retention policy, failure notifications. | 4 | `config/backup.php`, scheduled `backup:run/clean/monitor` | tests/Feature/Infrastructure/BackupConfigTest | planned |
| B-18-05 | Integration-ready: payment gateway, WhatsApp and printers each behind an interface. | 2 · 3 · 4 | `app/Contracts/*`, managers | tests/Unit/Integrations/ManagerResolutionTest | planned |
| B-19-01 | Technical proposal never invents prices, fees, project cost or vendor capabilities; unknowns marked “TO CONFIRM WITH VENDOR”. | 0 | `docs/technical-proposal.md` | owner review | done (document) |
| B-19-02 | Code-ownership section: Rofliv owns code, database and assets; only MIT-compatible open-source packages; no proprietary lock-in. | 0 | `docs/technical-proposal.md` §11 | owner review | done (document) |
| B-19-03 | Requirements traceability matrix with one row per requirement, updated at the end of every phase. | 0 | `docs/requirements-traceability.md` (this file) | owner review | done (document) |
| B-19-04 | `CLAUDE.md` capturing stack, architecture rules and business rules. | 0 | `CLAUDE.md` | owner review | done (document) |
| B-19-05 | Phase protocol: one phase at a time; migrations, seeders and full suite green; matrix updated; summary; commit; stop and wait. | 0 | `docs/phases.md`, `CLAUDE.md` | owner review | done (document) |
| B-19-06 | Owner note “ability to confirm the order”: customer review-and-confirm step; staff/admin Confirm action re-checking the gateway; optional per-location staff-confirmation hold. | 2 (review, confirm action) · 3 (POS button, KDS hold) | `pages/customer/Review.vue`, `ConfirmOrderManually`, `locations.requires_staff_confirmation` | tests/Feature/Customer/ReviewStepTest, tests/Feature/Orders/ManualConfirmationTest | planned |

## Phase summary

| Phase (first delivery) | Rows |
|---|---|
| 0 | 17 |
| 1 | 82 |
| 1–4 | 2 |
| 2 | 74 |
| 2–3 | 1 |
| 2–4 | 1 |
| 3 | 52 |
| 4 | 26 |
