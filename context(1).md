# Haramain Travels Manager — Project Context

> Project context for Codex and developers. Updated: 2026-11-01.  
> **Document purpose:** Build an internal Shopify app for managing paid transport bookings, supplier assignments, cost of service, profitability, and exports. This is a specification and historical context, **not** evidence that the manager app has been built or deployed.

## 1. Project identity and terminology

- **Internal app name:** Haramain Travels Manager (working title).
- **Business:** Private fixed-fare pilgrim transport between Makkah, Madinah, Jeddah Airport, Haramain train stations, and Ziyarah routes.
- **Current public-facing brand:** **Haramain Move** (formerly Haramain Travels). The latest known public domain is `https://move.haramain.com`; the Shopify store used during the preceding development work was `https://travel.haramain.com`. **Verify which Shopify store/domain is canonical before installation**. Keep “Haramain Travels Manager” as the internal app's working name unless the business decides to rename it.
- **Customer booking and payment platform:** Shopify.
- **Admin currency:** Saudi riyal (SAR). Storefront may support display/payment in SAR, GBP, USD through Shopify Markets; never assume all paid amounts are SAR. Record source currency and actual payment amounts and convert only using recorded, auditable exchange rates when needed.
- **Timezone for transport:** `Asia/Riyadh`; booking creation and settlement timestamps should be preserved in UTC as well.
- **Existing customer account portal:** `https://account.travel.haramain.com` (verify if domain changes with rebranding).
- **Support WhatsApp:** `+966 55 761 3244`; link `https://wa.me/966557613244`.
- **App users:** Internal authorised staff, not public customers or suppliers by default.

## 2. Goal and business problem

The business collects customer payments through Shopify, then arranges the journey with a third-party transport company, initially **Vista Travels**. Operations need to see who is travelling, assign an individual journey to a supplier, record actual supplier cost and payment status, and export filtered trip lists to XLSX/CSV. A single Shopify order can contain **multiple independent journeys**, sometimes assigned to different suppliers. Therefore the app's primary working record is the **journey/transfer**, not only the order.

Build a private app accessible from Shopify Admin → Apps. Prefer a Shopify-hosted **App Home UI extension** and Shopify-native storage if current platform limitations permit; validate capability first, especially server-side work, permission requirements, data volume, and file downloads. If necessary, provide a clearly scoped minimal-backend alternative with estimated operational costs before implementing.

Do **not** rebuild the storefront booking form, process customer payments, or replace Shopify checkout.

## 3. MVP modules

### 3.1 Overview dashboard

Provide date filters and KPI cards:
- Orders and journeys (not interchangeable); paid, unpaid, cancelled, refunded.
- Total customer revenue in a selected reporting currency, with original currencies retained.
- Estimated supplier cost; confirmed supplier cost.
- Estimated and confirmed gross profit; margin percentage.
- Unassigned journeys and pickup deadlines; journeys today/tomorrow/next 7 days (Saudi time).
- Supplier payable balance, including unpaid and partially paid costs.
- Counts by assignment status, supplier, route, vehicle, city.

Use clear labels for **estimated**, **confirmed**, **gross profit**, **net profit**, **paid**, **refunded**; never claim an estimate is actual profit.

### 3.2 Bookings / journeys

- Read Shopify orders and line items via the Shopify Admin API, subject to granted order scopes and Shopify privacy rules.
- Convert each booking line item into a journey record. One order can produce 1–N journeys.
- Columns: Shopify order number, journey sequence, created date, travel date, pickup time, route, vehicle, customer contact (permission-based), trip status, supplier, selling amount, cost, profit, supplier payment status.
- Search by order number, customer, flight/train number, route, hotel and supplier; pagination and filters.
- Drill into trip details and linked Shopify order. Show only the customer data needed to fulfil the transfer.
- Handle repeat routes and multiple identical variants in one order; identify each journey with the **Shopify order ID + line-item ID** (and journey number for human display), not a route name or variant ID alone.

### 3.3 Suppliers and trip assignments

- Create, edit and deactivate suppliers; initial supplier: **Vista Travels**.
- Assign one journey to one supplier at a time for MVP. Preserve assignment history; support reassignment with reason and audit trail.
- Fields: assigned supplier ID, quoted cost, agreed final cost, supplier booking reference, confirmation date, assigned-by user, driver name/phone where provided, notes, supplier invoice reference, supplier payment status.
- Journey operational states: `unassigned`, `assigned`, `supplier_confirmed`, `driver_assigned`, `in_progress`, `completed`, `cancelled` (validate required states in UI).
- Supplier payment states: `unpaid`, `partially_paid`, `paid`, `disputed` (independent of trip status and independent of Shopify customer payment state).
- Store payment amount(s), date(s), method, references; do not equate assignment with supplier payment.

### 3.4 Supplier pricing / cost rules

- Import the supplied `Haramain_Travels_Pricing.xlsx` as the baseline supplier rate card. **Do not guess missing prices.**
- Workbook tabs verified: `B2B Cost`, `Retail`, `Margin`, `Settings`.
- Supplier cost sheet: **Vista Travels – B2B Cost (SAR)**; source is an October 2026 rate card provided by Luqman according to workbook metadata.
- Retail prices are calculated from a per-vehicle markup, rounded up to a SAR increment; workbook Settings has a **SAR 5** rounding increment. Treat retail values as reference only; authoritative sold fare is the paid Shopify line amount.
- Rate table keys: supplier, canonical route ID, vehicle ID, effective-from/effective-to dates, currency, cost. Support revisions and historical versions.
- Suggest a cost automatically when a journey arrives; require manual confirmation or explicit auto-approval workflow. Persist the **cost snapshot** used for each assignment so future rate updates do not alter historical profit.
- Distinguish quoted, agreed and actual costs; allow overrides with audit metadata.
- Map historical generic `Hotel - Train Station` / `Train Station - Hotel` rates to city-specific Makkah and Madinah train routes only where the business has confirmed that the same prices apply; do not merge their journey identities.

### 3.5 Financial tracking

At the journey level:
- Customer selling price from the real paid line-item amount (including discounts allocation as appropriate).
- Refunds allocated to affected journey(s); taxes, processing fees and FX effects separately where data exists.
- Supplier cost estimate and agreed actual supplier cost.
- **Gross profit** = recognised journey revenue − applicable supplier cost (before other operating expenses).
- **Net contribution** = gross profit − allocated payment fees − journey-specific other expenses, when reliable values are available.
- Profit margin = gross profit / recognised revenue (guard against zero denominator).
- Overall totals must not double count order-level delivery fees, discounts, refunds, or multi-currency conversions.
- For unassigned trips, show provisional cost and profit as *estimated*, or `—` if costs are unavailable.
- Preserve original customer transaction currency, Shopify presentment/shop amounts as supplied, and FX conversion source/time.

### 3.6 Export and reporting

- Export by **Travel date** (default) or **Booking/order date** (separate filter), in `Asia/Riyadh` for trip schedules.
- Filters: start/end dates (inclusive), company, assignment status, customer payment/refund status, vehicle, route, city, trip status.
- One exported row per journey, repeating order number across multi-journey orders.
- **Supplier/operations export**: order reference, passenger/contact (authorised), pickup date/time, route, vehicle, pickup and drop-off, flight/train number, special instructions, assignment/reference/status; **do not expose customer selling price or internal gross margin** to transport companies.
- **Internal finance export**: selling price, cost snapshot, payments, refunds, estimated/confirmed margin, supplier, payable state, expense totals, currency, timestamp.
- CSV required in MVP; XLSX preferable if compatible with the chosen Shopify App Home runtime. If the runtime cannot safely download or generate XLSX, document the limitation and use an approved backend or other official export mechanism.
- Use safe CSV escaping (including prevention of spreadsheet formula injection), UTF-8, meaningful filenames, export audit logging, role checks, and no leakage of private fields.

## 4. Existing Shopify booking data

The earlier custom Shopify booking form builds itinerary items in the browser and adds route products to cart via Shopify AJAX (`cart/add.js`), then sends customers to Shopify checkout. It attaches **line-item properties**:

| Property | Meaning |
|---|---|
| `Route` | Human-readable route string |
| `Vehicle` | Selected vehicle |
| `Travel date` | Date selected by customer |
| `Pickup time (Asia/Riyadh)` | Pickup clock time in Saudi time |
| `Pickup hotel` | When needed |
| `Drop-off hotel` | When needed |
| `Flight number` | Airport routes when needed |
| `Train number` | Train routes when needed |
| `Map link` | Optional |
| `Special instructions` | Optional |
| `_Journey number` | Sequence within itinerary; private display property |

Field presence varies by route. Do not require airport/train attributes on ordinary hotel-to-hotel trips. Ignore unrelated shop items unless the app intentionally supports them. Use order/line-item IDs to maintain uniqueness; route property strings should be mapped to canonical route records, not used as immutable identifiers.

**Product/variant mapping (historical, verify against current store before deployment):**
- Earlier route product handles: `haramain-route-01` through `haramain-route-17`; each route product has 8 vehicle variants.
- Existing checkout handles and variant mappings must remain intact. Never reorder product variant IDs merely to change display order.
- Old route IDs 01 and 15 were updated to Makkah hotel/train transfers; 16 and 17 were added for Madinah hotel/train transfers. Actual current product names, statuses and route mappings must be fetched afresh.

## 5. Route master data

These are **historical canonical choices from the 17-route implementation**; verify current route IDs, names, directions, prices and whether rebranding changed any before seeding:

| Earlier product handle | Route |
|---|---|
| `haramain-route-01` | Makkah Hotel → Makkah Train Station |
| `haramain-route-02` | Jeddah Airport → Madinah |
| `haramain-route-03` | Jeddah Airport → Makkah |
| `haramain-route-04` | Madinah → Jeddah Airport |
| `haramain-route-05` | Madinah → Makkah, direct |
| `haramain-route-06` | Madinah → Makkah, via Badr |
| `haramain-route-07` | Madinah Airport → Madinah Hotel |
| `haramain-route-08` | Madinah → Madinah Airport |
| `haramain-route-09` | Madinah Ziyarah |
| `haramain-route-10` | Makkah → Jeddah Airport |
| `haramain-route-11` | Makkah → Madinah, direct |
| `haramain-route-12` | Makkah → Madinah, via Badr |
| `haramain-route-13` | Makkah → Taif Ziyarah |
| `haramain-route-14` | Makkah Ziyarah |
| `haramain-route-15` | Makkah Train Station → Makkah Hotel |
| `haramain-route-16` | Madinah Hotel → Madinah Train Station |
| `haramain-route-17` | Madinah Train Station → Madinah Hotel |

**Do not infer trips from route text alone** if there are historical spelling changes or altered labels. Create stable route codes and maintain aliases for historical orders (e.g., `Ziarat`/`Ziyarah`, `Camery`/`Camry`, `GMC`/`GMC Yukon`).

### Vehicle list (display order)

1. Camry — Sedan (3 seats)
2. Starex — Family van (7 seats)
3. Staria — Minivan (7 seats)
4. GMC Yukon — SUV (6 seats)
5. Luxury Staria — Premium van (7 seats)
6. Hiace — Van (11 seats)
7. Coaster — Minibus (20 seats)
8. Bus — Coach (45 seats)

Seats **exclude the driver**. Treat capacities beyond the explicitly verified sedan and Starex entries as *provisional* until checked against the actual fleet. Product variant order in Shopify may differ from visual display order.

## 6. Pricing workbook reference examples (SAR)

The following pairs are verified against the uploaded workbook and are included only for sanity tests. Import **all** rows/vehicles from the actual workbook for the application; do not seed just these examples.

| Route and vehicle | Vista B2B cost | Retail/reference |
|---|---:|---:|
| Hotel → Train Station, Camry | 90 | 110 |
| Train Station → Hotel, Camry | Check workbook | Check workbook |
| Jeddah Airport → Makkah, Camry | 200 | 240 |
| Jeddah Airport → Makkah, Starex | 220 | 265 |
| Madinah → Makkah direct, Camry | 350 | 420 |

The `Margin` sheet shows reference fare minus supplier cost; **do not use its values in place of actual-order profitability**. The source workbook should be retained as a versioned private artefact, not uploaded to a public repository if it contains confidential pricing.

## 7. Data model proposal

### Supplier
`id`, `name`, `contact_name`, `phone`, `email`, `active`, `created_at`, `updated_at`.

### RateCard / SupplierRate
`id`, `supplier_id`, `canonical_route_code`, `vehicle_code`, `cost_sar`, `effective_from`, `effective_to`, `source_document`, `version`, `approved_at`.

### JourneyAssignment
`id`, `shopify_order_gid`, `shopify_line_item_gid`, `journey_number`, `route_code`, `vehicle_code`, `travel_date`, `pickup_time_local`, `supplier_id`, `estimated_cost_sar`, `agreed_cost_sar`, `cost_source_version`, `operational_status`, `supplier_payment_status`, `supplier_reference`, `driver_name`, `driver_contact`, `notes`, `assigned_at`, `assigned_by`, `updated_at`.

### SupplierPayment
`id`, `supplier_id`, `journey_assignment_id` (or invoice allocation), `amount`, `currency`, `paid_at`, `method`, `reference`, `recorded_by`.

### AssignmentAuditEvent
`id`, `journey_assignment_id`, `actor`, `action`, `previous_value`, `new_value`, `occurred_at`.

Prefer Shopify metaobjects / metafields where API access, limits, indexing, performance, and integrity permit. Otherwise evaluate a minimal secured persistent backend. **Do not write supplier costs or internal margin into publicly visible product data or storefront line-item properties.** Document how sensitive personal data is encrypted, restricted, retained and deleted.

## 8. Data synchronisation and correctness

- Authoritative order/payment/refund source: Shopify Admin; authoritative assignment source: the Manager app.
- Initial import: query historical orders within granted access scope, with pagination and progress tracking.
- Incremental sync: if a backend exists, use verified Shopify webhooks with HMAC checks and idempotent processing. If extension-only, explicitly document the supported refresh/reconciliation mechanism and its operational trade-offs.
- Keep order status and trip status separate. Customer checkout success does not imply the supplier has confirmed the journey.
- Do not assume every order is paid, fulfilled, or completed; handle refunds, cancellations, edits, duplicate webhook delivery, and missing custom properties.
- Use immutable Shopify IDs and persist changes with proper concurrency control to avoid double assignments.
- Never silently modify customer order prices, create refunds, or change live checkout behavior.

## 9. UI and UX direction

- Branded, professional Shopify Admin experience: deep green `#0B5B4A` or Haramain green `#62A438`, charcoal `#26282B`, white, muted surfaces. Match native Shopify Polaris patterns where required; don't force storefront styling into restricted admin components.
- Desktop-oriented operations table with effective mobile fallback.
- Clear high-level nav: **Overview · Bookings · Suppliers · Pricing · Finances · Exports · Settings**.
- Date filters should visibly distinguish booking date versus travel date and default to **travel date** for dispatch.
- Journey details drawer/page should show order context, passenger contact, route/hotel details, supplier and actual-versus-estimated cost.
- Explicit empty/loading/error states, permission restrictions and confirmation dialogs for destructive changes.
- No fabricated revenue, bookings or balances. Any sample values must carry a visible *Demo data* label.

## 10. Security and access

- App is for this merchant only (private/custom distribution); do not build public customer login or a new checkout.
- Authenticate through Shopify-supported app/extension mechanisms. Never commit tokens to GitHub or paste secrets in chat.
- Request least-privilege scopes. Determine whether orders older than the default window require additional access; comply with protected customer data requirements.
- Restrict supplier cost/margin/financial exports to authorised staff; maintain staff roles, audit actions, minimise personal data in logs.
- Read-only Shopify order integration initially. Explicit approval before any production writes, app installation, webhook registration, or theme changes.

## 11. Build and deployment plan for Codex

**Phase 0 — Feasibility:**
1. Confirm current Shopify app home extension framework, extension targets, runtime/file download capabilities, size/performance limits, supported Admin API access, staff access, and metadata persistence using latest official documentation.
2. Confirm actual Shopify store ID, main domain, application registration and allowed app distribution mode.
3. Assess no-backend MVP versus minimal backend. Do not promise zero hosting costs until validated.

**Phase 1 — Prototype / test store:**
1. Private repository `haramain-travels-manager`.
2. App shell, authentication, permissions and a basic Overview screen.
3. Import Shopify development-store orders, one journey per line item.
4. Implement suppliers, assignment, rate import and gross-profit calculations.
5. Date-filtered CSV export with correct escaping; evaluate XLSX.
6. Add tests for multi-journey orders, missing properties, currency, refunds, date boundaries, reassignment and rate changes.

**Phase 2 — Release:**
1. Review app permissions and privacy requirements.
2. Validate supplier rate import and report totals against a small hand-checked sample.
3. Perform test-store user acceptance testing with fictional booking data.
4. Request explicit approval for live app installation and production data synchronisation.
5. Monitor issues and retain a rollback path.

**Important:** A Shopify Dev account does not by itself eliminate all deployment, Shopify CLI, hosting, or platform restrictions. Codex can write/build/test code, but deploying to Shopify requires authenticated developer access and the app's supported deployment flow. **Do not claim the app is live until deployment and installation have been verified.**

## 12. Non-goals for v1

- Replacing the existing storefront booking wizard or checkout.
- Supplier self-service portal or driver mobile app.
- Automated WhatsApp notifications (possible later, requires provider/API review).
- Automatically paying suppliers, refunding customers, or changing product prices.
- General GA4 marketing analytics (a separate dashboard initiative).

## 13. Open decisions / verification checklist

- [ ] Confirm canonical current brand and Shopify store: Haramain Move / `move.haramain.com` versus historical Haramain Travels / `travel.haramain.com`.
- [ ] Confirm developer app registration and custom distribution permissions.
- [ ] Confirm whether App Home extension-only architecture satisfies exports, data persistence, and role separation.
- [ ] Confirm Vista Travels rates are still current and approve city-specific train route mappings.
- [ ] Confirm actual vehicle capacities (especially GMC, Hiace, Coaster and Bus).
- [ ] Confirm how store currencies, taxes, discounts, gateway fees and refunds should be treated in management reporting.
- [ ] Confirm whether supplier invoicing is per journey or batched per statement.
- [ ] Confirm which staff roles may view margins and export customer information.
- [ ] Verify rate workbook and import results against Shopify products.
- [ ] Confirm order history access and protected customer-data eligibility.

---

**Codex working rule:** Treat this file as project context. Verify anything time-sensitive through current official Shopify documentation and the connected store before coding. If the current app platform cannot support a requirement, report the constraint, its implications and a safe alternative rather than silently dropping functionality or claiming success.
