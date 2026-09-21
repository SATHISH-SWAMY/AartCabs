# AART Cab — Product Plan & Database Design

**Stack:** React · Redux Toolkit · Node.js · Express · Firebase (Auth, Firestore, Storage, FCM, Cloud Functions)

Built from your 7 uploaded screens (Landing, Login, Register, Dashboard/MainPage, Carlist, PaymentProcess). Admin, employee and partner modules follow standard cab-dispatch software patterns (trip sheets, duty slips, vendor dispatch, payouts). They are not copied from Indecab; if you share screenshots of Indecab's admin, I can align the modules and field names.

---

## 0. Read this first: two things I found in your code

**1. Your code's "agent" is not your "agent".** In the uploaded screens, the person who books is called an *Agent* (Register has `customer | organization`, Payment has "Agent Email", Dashboard has "Markup Settings" and "Wallet"). In your brief, the *Agent* is the person who **receives** the order (a driver or a travel agency). These are opposite sides of the marketplace.

I use these names in the database so nothing gets mixed up:

| Your word | DB role | Who they are |
|---|---|---|
| Customer | `customer` | Books cabs. Two account types: `individual` or `organization` (travel agency booking for its clients, with markup and wallet, as in your Register flow) |
| Admin | `admin` | Owner. Sees all data, pricing, money, staff, partners |
| Admin's employee | `staff` | Works under admin. Assigns drivers, follows up |
| Agent | `partner` | Receives orders. Two types: `driver` (single owner-driver) or `agency` (travel agency or fleet owner with many drivers and vehicles) |

**2. Security items in the current code.** LoginPage has a pre-filled email and password (`info@aartcab.com` / `supersecretpass`) and the Login button is only a `<Link to="/dashboard">`, so there is no real auth. The left panel text says "powered by Savaari Car Rentals Pvt Ltd". Replace this with your own company name before launch, since it is another company's brand.

---

## 1. System overview

```
 React + Redux (Customer web)      React admin panel (Admin + Staff)      Partner portal / PWA
        │                                   │                                     │
        └──────────────┬────────────────────┴─────────────────────────────────────┘
                       │  HTTPS (Firebase ID token in Authorization header)
                ┌──────▼───────────┐
                │ Node/Express API │  ← Cloud Run. Single source of truth for money,
                │  (Admin SDK)     │    status changes, pricing, assignment
                └──────┬───────────┘
   ┌───────────────────┼──────────────────────────────────────────────┐
   │ Firebase Auth     │ Firestore (data)   Storage (KYC, invoices)   │
   │ (custom claims)   │ RTDB (live GPS)    FCM (push)                │
   └───────────────────┴──────────────────────────────────────────────┘
   Cloud Functions / Cloud Tasks: triggers, timers, webhooks, invoices, payouts
   External: Razorpay · Google Maps (or MapmyIndia/Ola Maps) · MSG91/Twilio (SMS/OTP) · WhatsApp API · SendGrid
```

**Golden rule:** clients may **read** from Firestore (realtime lists and tracking), but every **write** that involves money, booking status, pricing or assignment goes through Express. Firestore Security Rules block direct client writes on those collections.

---

## 2. Roles and permissions

Store the role in **Firebase Auth custom claims** (`{ role: "staff", perms: [...] }`) and mirror it in `users/{uid}`. Claims are readable in Security Rules without an extra database read.

| Capability | Customer | Staff | Admin | Partner |
|---|:-:|:-:|:-:|:-:|
| Search cars, book, pay | ✅ | ✅ (on behalf of a customer) | ✅ | ❌ |
| View own bookings/wallet | ✅ | – | – | – |
| View all bookings | ❌ | ✅ (region or assigned scope) | ✅ | ❌ |
| Assign partner/driver | ❌ | ✅ | ✅ | ❌ |
| Accept/reject offered trip | ❌ | ❌ | ❌ | ✅ (own offers) |
| Change fares/markup/rules | ❌ | ❌ | ✅ | ❌ |
| See customer phone/address | own | ✅ | ✅ | **Only after accept**, masked before |
| Payouts and settlements | ❌ | view | ✅ | own |
| Manage staff | ❌ | ❌ | ✅ | ❌ |

`staff.permissions[]` allows fine control (for example `can_cancel`, `can_refund`, `can_edit_fare`, `can_view_finance`).

---

## 3. Booking lifecycle (state machine)

```
draft → payment_pending → confirmed → dispatching → partner_accepted → driver_assigned
      → driver_enroute → arrived → in_trip → completed → settled
                     ↘ cancelled (from any state before in_trip, with refund rules)
                     ↘ no_show · failed_payment · refunded
```

Rules enforced only on the server:
- Every transition is validated (`confirmed` cannot jump to `completed`).
- Every transition writes an event to `bookings/{id}/timeline`.
- Once `driver_assigned`, the partner's cost is locked. Changes require admin permission.

---

## 4. Firestore data model

Firestore is NoSQL. The design below uses **top-level collections** for things queried on their own, **subcollections** for things that only make sense under a parent, and **deliberate denormalization** (small snapshots copied into a booking) so lists render in one read and past bookings never change when a price or name changes later.

### 4.1 Identity

#### `users/{uid}` — one doc per login (all roles)
```js
{
  uid, role: "customer" | "admin" | "staff" | "partner",
  status: "active" | "pending_verification" | "suspended" | "deleted",
  firstName, lastName, displayName,
  email, emailVerified,
  phone,            // E.164, e.g. +919876543210
  phoneVerified,
  photoUrl,
  fcmTokens: [ { token, platform, lastSeenAt } ],
  lastLoginAt, createdAt, updatedAt
}
```

#### `customers/{uid}` — profile for role = customer
```js
{
  uid,
  accountType: "individual" | "organization",          // from Register.jsx
  city,
  // only when organization (travel agency booking for its clients)
  organization: { agencyName, pan, gstin, address, city, state, pincode, kycStatus: "pending"|"verified"|"rejected", kycDocs: [ { type, storagePath } ] },
  walletId,                       // → wallets/{walletId}
  markup: { type: "percent" | "flat", value: 0, appliesTo: "all" },   // "Markup Settings" in Dashboard
  commissionPlanId,               // → commissionPlans/{id}, for agency-type customers
  creditLimit: 0,                 // optional postpaid credit
  tags: ["vip", "corporate"],     // admin CRM
  source: "web" | "referral" | "staff_created",
  assignedStaffId,                // relationship owner for follow-up
  stats: { totalBookings, completedBookings, cancelledBookings, lifetimeValue, lastBookingAt },
  createdAt, updatedAt
}
```
`stats` is updated by a Cloud Function on booking completion. It is what the admin "collects" about customers, so no report has to scan every booking.

#### `staff/{uid}` — admin's employees
```js
{
  uid, employeeCode, adminUid,
  department: "dispatch" | "support" | "finance" | "sales",
  designation,
  permissions: ["assign_driver","cancel_booking","refund","view_finance","edit_fare"],
  regions: ["BLR","MYS"],          // cities/zones they handle; used to route bookings
  shift: { days: [1,2,3,4,5], start: "09:00", end: "18:00" },
  isOnDuty: true,
  workload: { openBookings: 0, openFollowUps: 0 },   // used for auto-assign
  createdAt, updatedAt
}
```

#### `admins/{uid}`
```js
{ uid, companyName, superAdmin: true, twoFactorEnabled: true, createdAt }
```

### 4.2 Partners (agents who receive orders)

#### `partners/{partnerId}`
```js
{
  partnerId, ownerUid,
  type: "driver" | "agency",
  displayName, businessName,
  phone, email,
  status: "pending" | "verified" | "active" | "suspended" | "rejected",
  serviceAreas: [ { cityId, cityName, type: "pickup"|"drop"|"both" } ],   // "particular location"
  tripTypes: ["oneway","roundtrip","local","airport"],
  vehicleCategories: ["hatch","sedan","muv","suv"],
  kyc: { pan, gstin, aadhaar_last4, docs: [ { type, storagePath, expiresAt, status } ] },
  bank: { accountHolder, accountNumberEnc, ifsc, upiId, razorpayFundAccountId },  // never store raw account no. in plain text
  commission: { type: "percent"|"flat", value: 10 },   // platform cut on partner jobs
  rating: { avg: 4.6, count: 128 },
  reliability: { acceptRate: 0.82, cancelRate: 0.02, onTimeRate: 0.96 },
  isOnline: true,
  createdAt, updatedAt
}
```
A **single driver** is a partner with `type: "driver"` and one linked driver/vehicle. An **agency** is a partner with many drivers and vehicles. Both are treated the same by dispatch.

#### `drivers/{driverId}`
```js
{
  driverId, partnerId,            // partnerId = the agency (or self for solo driver)
  uid,                            // Firebase login if the driver uses the app, else null
  name, phone, photoUrl,
  licence: { number, expiresAt, storagePath },
  status: "available" | "on_trip" | "off_duty" | "blocked",
  currentVehicleId,
  languages: ["en","hi","kn"],
  rating: { avg, count },
  createdAt, updatedAt
}
```

#### `vehicles/{vehicleId}`
```js
{
  vehicleId, partnerId,
  registrationNo, categoryId,     // → vehicleCategories
  make, model, year, color,
  fuel: "diesel" | "petrol" | "cng" | "ev",
  seats, bags,
  docs: { insuranceExpiry, permitExpiry, fitnessExpiry, pucExpiry },
  status: "active" | "maintenance" | "inactive",
  createdAt
}
```

### 4.3 Catalogue and pricing (admin-managed, cached heavily)

#### `vehicleCategories/{categoryId}` — powers the Carlist screen
```js
{
  categoryId: "sedan", variant: "sedan",          // matches CarArt variant in Carlist.jsx
  name: "Dzire, Etios", modelLabel: "or similar",  // "exact model" for Innova
  filterGroup: "Sedan",                            // Hatchback | Sedan | MUV | SUV
  fuelOptions: ["diesel","cng"],
  seats: 4, bags: 2,
  imageUrl, sortOrder, isActive: true
}
```

#### `cities/{cityId}` and `airports/{airportId}`
```js
// cities
{ cityId: "BLR", name: "Bangalore", state: "Karnataka", geo: GeoPoint, geohash, aliases: [], isActive: true, zone: "south" }
// airports
{ airportId: "BLR_KIA", cityId: "BLR", name: "Kempegowda International", iata: "BLR", geo: GeoPoint }
```

#### `fareRules/{ruleId}` — pricing engine input
```js
{
  ruleId, tripType: "oneway" | "roundtrip" | "local" | "airport",
  categoryId, fromCityId: null, toCityId: null,      // null = default; specific pair overrides
  baseKm: 0, baseFare: 0,
  perKm: 0, minKmPerDay: 250,
  driverAllowancePerDay: 0, nightAllowance: 0,
  tollIncluded: true, stateTaxIncluded: true, parkingIncluded: false,
  gstPercent: 5,
  extraKmRate: 0, extraHourRate: 0,
  packages: [ { hours: 8, km: 80, price: 0 } ],      // for "local"
  validFrom, validTo, priority: 0, isActive: true
}
```

#### `commissionPlans/{planId}` · `coupons/{code}` · `settings/global`
```js
// commissionPlans (for agency-type customers)
{ planId, name, type: "percent", value: 5, tiers: [ { minMonthlyBookings: 50, value: 7 } ] }

// coupons
{ code: "FIRST100", type: "flat"|"percent", value, maxDiscount, minFare, usageLimit, usedCount, perUserLimit, validFrom, validTo, applicableTo: ["oneway"], isActive }

// settings/global
{ zeroCashBufferPercent: 20, minAdvancePercent: 25, fullPaymentWithinHours: 48,
  offerExpiryMinutes: 10, gstRate: 5, supportPhone, cancellationPolicy: [ { hoursBefore: 24, refundPercent: 100 }, { hoursBefore: 4, refundPercent: 50 } ] }
```
The `zeroCashBufferPercent: 20`, `25%` minimum and `48h` values come straight from PaymentProcess.jsx. They are settings, not hard-coded numbers.

### 4.4 Bookings (core collection)

#### `bookings/{bookingId}`
`bookingId` is a readable ID such as `AC-260921-00123`, generated from a counter (see 4.7).

```js
{
  bookingId, bookingNo: "AC-260921-00123",
  status: "confirmed",                           // see state machine
  source: "web" | "admin_panel" | "api",

  // ---- WHO ----
  customerId,                                    // uid of the booker
  customerSnapshot: { name, phone, email, accountType, agencyName },
  createdBy: { uid, role },                      // customer or staff who created it

  // ---- TRIP (from MainPage / Carlist search) ----
  trip: {
    type: "oneway" | "roundtrip" | "local" | "airport",
    airportDirection: null | "drop" | "pickup",  // "Drop to Airport" / "Pickup from Airport"
    from: { cityId, name, address, geo },
    to:   { cityId, name, address, geo },
    stops: [ { cityId, name, order } ],          // round-trip multi-city chain
    returnCity: { cityId, name },
    pickupAt: Timestamp,                         // pickup date + time
    returnAt: null,
    localPackage: { hours, km } | null,
    distanceKm: 982, durationMin: 0,
    routePolyline: "..."
  },

  // ---- CAR (Carlist) ----
  vehicle: { categoryId, name, modelLabel, seats, bags, fuel },
  quantity: 1,                                   // "multi-car" stepper in Carlist

  // ---- PICKUP DETAILS FORM (PaymentProcess step 1) ----
  passenger: {
    name, phone,
    pickupAddress, dropAddress,
    landmark,                                    // "Landmark / Door Number / Building"
    flightOrTrain: "6E-204",
    passengerCount: 2, luggageCount: 1,
    notes
  },

  // ---- MONEY (all integers in paise; never floats) ----
  fare: {
    baseFare, tax, markup, discount, couponCode,
    total,                                       // customer-facing
    mrp, savings,                                // "Save ₹2,837" badge
    currency: "INR",
    inclusions: ["Fuel","Driver Allowance","Toll/State tax","GST 5%"],
    exclusions: ["Parking","Interstate permit","Night halt"]
  },
  paymentPlan: {
    type: "partial_to_driver" | "zero_cash",
    payNowPercent: 25, payNowAmount,
    collectByDriver,                             // "Customer → driver"
    bufferPercent: 20, bufferAmount,             // refundable buffer for zero_cash
    paidAmount, refundedAmount,
    paymentStatus: "unpaid" | "partial" | "paid" | "refunded"
  },
  partnerCost: { agreedAmount, platformCommission, netToPartner },   // hidden from customer

  // ---- ASSIGNMENT (staff work) ----
  assignment: {
    handledByStaffId,
    partnerId, partnerName,
    driverId, driverSnapshot: { name, phone, photoUrl },
    vehicleId, vehicleSnapshot: { registrationNo, model, color },
    assignedAt, assignedBy,
    mode: "manual" | "auto" | "broadcast"
  },

  // ---- TRIP EXECUTION ----
  execution: {
    startOdometer, endOdometer, actualKm,
    startedAt, endedAt,
    extras: { toll, parking, extraKm, extraHours, nightAllowance },
    startOtp,                                    // customer gives OTP to driver at pickup
    dutySlipUrl
  },

  // ---- SEARCH/REPORT HELPERS ----
  regionKey: "BLR",                              // for staff routing and filters
  fromCityId, toCityId, pickupDate: "2026-09-21",  // string for cheap day queries
  followUp: { nextAt, openCount },
  flags: ["vip","airport","urgent_within_48h"],
  cancellation: { by, reason, at, refundAmount } | null,
  invoiceId, rating: { customer: 5, partner: 4 },

  createdAt, updatedAt, version: 3               // optimistic-lock counter
}
```

#### Subcollections of a booking
```
bookings/{id}/timeline/{eventId}   { at, actorUid, actorRole, type: "STATUS_CHANGED"|"ASSIGNED"|"CALL_LOGGED"|"PAYMENT"..., from, to, note }
bookings/{id}/notes/{noteId}       { at, staffId, text, visibility: "internal" | "partner" }
bookings/{id}/messages/{msgId}     { at, from: "customer"|"staff"|"driver", text }    // optional in-app chat
```
Tracking data does **not** go here (see below).

#### `dispatchOffers/{offerId}` — sending a trip to outside partners
```js
{
  offerId, bookingId, partnerId,
  status: "offered" | "accepted" | "rejected" | "expired" | "withdrawn",
  offeredAmount, expiresAt,
  bookingPreview: { fromCity, toCity, pickupAt, category, distanceKm, partnerAmount },  // NO customer phone/address until accepted
  respondedAt, rejectReason,
  offeredBy, createdAt
}
```
This is the "some data goes to an outside travel agency partner for a particular location" requirement. The partner sees only the preview. Customer details are released to the partner (via a server-side copy to `bookings/{id}/partnerView`) only after the partner accepts.

#### `liveTracking/{bookingId}` — **Realtime Database**, not Firestore
```js
{ lat, lng, speed, heading, updatedAt, driverId }   // driver app writes every 5–10 seconds
```
Live GPS is high-write. Realtime DB is cheaper for this. Firestore would charge a write per ping.

### 4.5 Money

#### `payments/{paymentId}`
```js
{
  paymentId, bookingId, customerId,
  gateway: "razorpay", gatewayOrderId, gatewayPaymentId, gatewaySignature,
  method: "upi" | "card" | "netbanking" | "wallet",
  purpose: "advance" | "full" | "buffer" | "balance",
  amount, currency: "INR",
  status: "created" | "authorized" | "captured" | "failed" | "refunded" | "partially_refunded",
  idempotencyKey,
  refunds: [ { refundId, amount, reason, at } ],
  failureReason, webhookEventIds: [],            // dedupe repeated webhooks
  createdAt, capturedAt
}
```

#### `wallets/{walletId}` and `walletTransactions/{txnId}` (append-only ledger)
```js
// wallets
{ walletId, ownerUid, ownerRole: "customer"|"partner", balance, holdAmount, currency, updatedAt }

// walletTransactions — NEVER updated or deleted, only added
{ txnId, walletId, type: "credit"|"debit"|"hold"|"release"|"commission"|"refund"|"payout"|"topup",
  amount, balanceAfter, bookingId, paymentId, note, createdBy, createdAt }
```
The balance in `wallets` is derived from the ledger inside a Firestore **transaction**. This is what makes "Zero cash" refunds of unused buffer safe.

#### `payouts/{payoutId}` — paying partners
```js
{ payoutId, partnerId, periodStart, periodEnd,
  bookingIds: [], grossAmount, commission, tds, adjustments, netAmount,
  status: "pending"|"approved"|"processing"|"paid"|"failed",
  utr, paidAt, approvedBy }
```

#### `invoices/{invoiceId}`
```js
{ invoiceId, invoiceNo, bookingId, customerId, type: "customer"|"partner",
  gstin, hsn: "9966", lineItems: [], cgst, sgst, igst, total, pdfPath, createdAt }
```
This supports the "GST invoices" promised in your Register benefits.

### 4.6 Operations and support

```js
// followUps/{id} — staff task list (the "follow up of driver and customer")
{ followUpId, bookingId, assignedTo: staffId, target: "driver"|"customer"|"partner",
  reason: "confirm_pickup" | "driver_not_reachable" | "payment_pending" | "feedback",
  dueAt, status: "open"|"done"|"snoozed", outcome, createdAt, completedAt }

// supportTickets/{id}
{ ticketId, bookingId, raisedBy, category, priority, status, assignedTo, messages: [], createdAt }

// notifications/{id}
{ notificationId, toUid, channel: "push"|"sms"|"whatsapp"|"email", template, payload, status, sentAt, readAt }

// feedback/{id}        (Dashboard "Feedback" menu)
{ bookingId, fromUid, toType: "driver"|"platform", rating, comment, createdAt }

// auditLogs/{id}       (every admin/staff write to sensitive data)
{ actorUid, actorRole, action: "FARE_EDIT", collection, docId, before, after, ip, at }

// counters/{name}      (for bookingNo, invoiceNo; use sharded counters if > 1 write/sec)
{ name: "booking_2026-09-21", value: 123 }

// otpSessions/{id}     (only if you don't use Firebase Phone Auth; store hash, TTL 10 min)
{ target, channel, codeHash, attempts, expiresAt }
```

### 4.7 Relationships at a glance

```
users ──1:1── customers | staff | admins | partners
partners ──1:N── drivers ──N:1── vehicles
customers ──1:N── bookings ──1:N── timeline / notes / messages
bookings ──1:N── dispatchOffers ──N:1── partners
bookings ──1:N── payments ; bookings ──1:1── invoice
wallets ──1:N── walletTransactions ; partners ──1:N── payouts
staff ──1:N── bookings (handledBy) ; staff ──1:N── followUps
vehicleCategories + fareRules + cities → pricing → bookings.fare (snapshot)
```

### 4.8 Composite indexes you will need
| Collection | Fields | Used by |
|---|---|---|
| bookings | `customerId` ↑, `createdAt` ↓ | Customer "My Bookings", Recent Bookings |
| bookings | `status` ↑, `pickupAt` ↑ | Dispatch board (upcoming trips per status) |
| bookings | `regionKey` ↑, `status` ↑, `pickupAt` ↑ | Staff region queue |
| bookings | `assignment.handledByStaffId` ↑, `status` ↑ | Staff "my bookings" |
| bookings | `assignment.partnerId` ↑, `pickupAt` ↓ | Partner trip history |
| dispatchOffers | `partnerId` ↑, `status` ↑, `expiresAt` ↑ | Partner inbox |
| followUps | `assignedTo` ↑, `status` ↑, `dueAt` ↑ | Staff follow-up list |
| payments | `bookingId` ↑, `createdAt` ↓ | Payment history |
| walletTransactions | `walletId` ↑, `createdAt` ↓ | Wallet statement |
| partners | `status` ↑, `serviceAreas.cityId` (array-contains) | Find partners for a city |

---

## 5. Screen → collection mapping (your uploaded files)

| Screen | Reads | Writes (via Express) |
|---|---|---|
| **LandingPage** | `settings/global`, cached stats | – |
| **Login** | Firebase Auth → `users`, custom claims | `users.lastLoginAt` |
| **Register** | – | `users` + `customers` (+ `organization`), OTP verify, `wallets` |
| **MainPage/Dashboard** (One Way, Round Trip, Local, Airport tabs) | `cities`, `airports`, `wallets`, last `bookings` | `quotes` (optional) |
| **Carlist** (filters: type, fuel, sort; tabs: Inclusions/Exclusions/Facilities/T&C) | `vehicleCategories`, `fareRules` → computed price | – |
| **PaymentProcess step 1** (pickup details) | – | `bookings` (status `draft`) |
| **PaymentProcess step 2** (partial vs zero-cash, coupon, wallet/card/UPI) | `settings/global`, `coupons` | `payments`, `walletTransactions`, `bookings.paymentPlan` |
| **Dashboard menu:** My Account, Reports, Wallet, Markup Settings, Feedback | `customers`, `bookings`, `wallets`, `feedback` | `customers.markup`, `feedback` |

The car prices on Carlist (₹18,877, "Save ₹2,837") must be computed by the server from `fareRules`, and the client must never send the price. Otherwise anyone can change the fare in the browser and pay less.

---

## 6. Backend structure (Node + Express)

```
server/
  src/
    config/          firebase.js, razorpay.js, env.js
    middleware/      auth.js (verifyIdToken), requireRole.js, rateLimit.js, validate.js (zod), errorHandler.js
    modules/
      auth/          register, otp, me
      catalog/       cities, vehicleCategories, search
      pricing/       fareEngine.js  ← pure function: (trip, category, rules, coupon, markup) → fare
      bookings/      create, get, list, cancel, status transitions
      payments/      createOrder, verify, webhook, refund
      wallet/        topup, ledger, hold/release
      dispatch/      offerToPartner, accept, reject, autoAssign, reassign
      partners/      onboarding, drivers, vehicles, availability
      staff/         CRUD, workload, followUps
      admin/         reports, pricing rules, payouts, settings
      notifications/ push, sms, whatsapp, email
    jobs/            offerExpiry, reminders, payoutRun (Cloud Tasks / Scheduler)
    utils/           idGenerator, money.js (paise helpers), logger
  functions/         Firestore triggers: onBookingWrite → stats, timeline, notifications
```

### Key API endpoints
```
POST /auth/register                    GET  /auth/me
GET  /catalog/cities?q=                POST /search/quote          → list of cars with computed price
POST /bookings                         GET  /bookings?status=&cursor=
POST /bookings/:id/cancel              POST /bookings/:id/status
POST /payments/order                   POST /payments/verify        POST /webhooks/razorpay
POST /dispatch/:bookingId/offer        POST /dispatch/offers/:id/accept | reject
POST /dispatch/:bookingId/assign       (staff picks partner + driver + vehicle)
GET  /staff/followups                  POST /staff/followups/:id/complete
GET  /admin/reports/revenue            PUT  /admin/fare-rules/:id
POST /partners/drivers                 POST /partners/vehicles      PUT /partners/availability
```

---

## 7. Frontend structure (React + Redux Toolkit)

```
src/
  app/            store.js, routes.jsx, guards (RequireAuth, RequireRole)
  features/
    auth/         authSlice (user, role, token)   ← replaces the hard-coded <Link to="/dashboard">
    search/       searchSlice (tripType, from, to, stops, pickupAt) ← MainPage form state
    cars/         carsApi (RTK Query), filters, sort, quantity
    booking/      bookingDraftSlice (passenger, plan, paymentMethod)  ← PaymentProcess state
    wallet/       walletApi
    admin/        dispatch board, pricing, reports
    partner/      offers inbox, my drivers, my vehicles
  shared/         ui components (Field, Card, Modal), hooks, formatters (money, date)
  apps/           customer/  admin/  partner/  (three route trees, code-split)
```

Recommendations:
- **RTK Query** for API calls (caching, invalidation, loading states) instead of hand-written thunks.
- **Firestore `onSnapshot`** for live data (partner offers, dispatch board, tracking). Push snapshots into RTK Query using `onCacheEntryAdded`.
- Persist only the `search` and `bookingDraft` slices (redux-persist) so a refresh on the payment page does not lose the customer's data.
- Keep **money as integer paise** in the store, and format only when rendering.
- Split the bundle by role with `React.lazy` so customers do not download admin code.

---

## 8. Admin, staff and partner modules (what to build)

**Admin panel**
1. Live dispatch board (Kanban by status; filter by city, date, staff).
2. Customer CRM: all customer data, tags, lifetime value, assigned staff.
3. Staff management: create, roles and permissions, workload, shifts.
4. Partner management: KYC approval, service areas, ratings, block or unblock.
5. Pricing: fare rules, city-pair overrides, coupons, commission plans.
6. Finance: payments, refunds, wallet ledger, payouts, GST invoices.
7. Reports: bookings, revenue, cancellations, partner performance, staff performance. Export CSV.
8. Audit log and settings.

**Staff (employee) workspace**
1. "My queue": new bookings in their regions, sorted by pickup time (urgent first).
2. Assign flow: pick partner, driver and vehicle, or "send offer to partners in this city".
3. Follow-up tasks with auto-created reminders (e.g. 2 h before pickup: confirm driver; driver late: call).
4. Call/notes log on each booking, with timeline.
5. Create a booking on behalf of a customer (phone orders).

**Partner portal (mobile-first PWA)**
1. Offers inbox with countdown timer, accept or reject.
2. Agency: manage drivers and vehicles, assign own driver to an accepted trip.
3. Trip screen for driver: start trip (OTP), navigation, live location sharing, end trip, add extras (toll/parking).
4. Earnings and payout statements.

---

## 9. Phased delivery plan

| Phase | Weeks | Deliverable |
|---|---|---|
| **0. Foundation** | 1–2 | Firebase project (dev/stage/prod), Express skeleton on Cloud Run, CI/CD, Firebase Auth with custom claims, Security Rules v1, env and secrets management |
| **1. Customer MVP** | 3–6 | Real Register/Login/OTP, city search, `fareEngine`, Carlist from DB, booking draft, Razorpay payment (both plans), booking confirmation, My Bookings, wallet basic |
| **2. Admin and Staff** | 7–10 | Admin panel, dispatch board, manual assignment, follow-ups, timeline, notes, customer CRM, notifications (SMS/WhatsApp/push) |
| **3. Partners** | 11–14 | Partner onboarding and KYC, offers and accept flow, driver/vehicle management, partner PWA, trip start OTP, live tracking, extras, duty slip |
| **4. Finance and reports** | 15–17 | Payouts, GST invoices, refunds/cancellation policy engine, reports and exports, audit log |
| **5. Scale and polish** | 18+ | Auto-dispatch, dynamic pricing, ratings, coupons, referral, analytics in BigQuery, load tests, security review |

Ship Phases 0–2 with a **manual dispatch process** first, since one small ops team can run bookings using the admin panel. Automate dispatch only after you see real data.

---

## 10. Suggestions for better performance, safety and quality

### Performance
1. **Cache the catalogue.** `cities`, `vehicleCategories` and `fareRules` change rarely. Serve them from an in-memory or Redis (Memorystore/Upstash) cache in Express plus CDN caching, and use short TTLs, so the search page does not cost Firestore reads.
2. **City autocomplete** should not hit Firestore on every keystroke. Load the city list once (a few thousand rows is small), or use Typesense/Algolia. Debounce input by 250 ms.
3. **Paginate everything** with cursors (`startAfter`), 20–25 rows per page. Never `onSnapshot` a whole collection; listen to a filtered query with `limit`.
4. **Denormalize for reads.** The snapshots inside `bookings` (customer, driver, vehicle) mean a list needs one query and no joins. Update snapshots by Cloud Function only when the change matters.
5. **Aggregate counters**, not scans. Dashboard numbers (like the "Bookings" stat in your MainPage tabs) come from `stats` docs or `daily_stats/{date}`, updated by triggers. Firestore `count()` aggregation is fine for small ones.
6. **Live GPS in Realtime Database**, throttled (every 5–10 s, only while a trip is active).
7. **Image and asset optimization.** Serve car images in WebP through a CDN with lazy loading. Your `CarArt` SVGs are already light; keep them for the default.
8. **Frontend:** code-split by role and route, memoize list rows, virtualize long tables (`react-window`) for the admin panel.
9. **Avoid hot documents.** Firestore handles about 1 write/sec per document. Use sharded counters for `counters` if booking volume grows.
10. **Background jobs** (offer expiry, reminders, invoice PDFs, payouts) belong in **Cloud Tasks / Scheduler**, not inside request handlers.

### Reliability and money safety
11. **Idempotency keys** on payment creation and booking creation, so a double click or retry cannot create two bookings or two charges.
12. **Verify Razorpay webhooks** (signature and event id dedupe). Treat the webhook, not the browser redirect, as the truth that a payment succeeded.
13. **Firestore transactions** for wallet changes and for "assign partner" (so two staff cannot assign the same trip). Use the `version` field for optimistic locking.
14. **Auto-expire offers** (`expiresAt` plus a scheduled task), and move to the next partner automatically.
15. **Never trust the client for price.** Recompute the fare on the server at booking and payment time. Store the fare snapshot on the booking.
16. **Soft delete only** for users, partners and bookings. Keep `auditLogs` immutable.

### Security
17. **Firebase App Check** and **rate limiting** (`express-rate-limit`, per IP and per user), plus `helmet` and strict CORS.
18. **Field masking:** partners get a stripped `bookingPreview` until they accept. Staff see full data. Log every time staff view a customer's phone if compliance matters.
19. **Encrypt sensitive fields** (bank account, Aadhaar) and store KYC docs in private Storage with signed URLs and short expiry.
20. **Remove the pre-filled credentials** in LoginPage, add reCAPTCHA or phone OTP for login and registration, and add **2FA for admin**.
21. **India compliance:** GST invoicing (and e-invoice if turnover requires it), TDS on partner payouts, DPDP Act consent and data-retention policy.

### Product and growth
22. **Driver mobile experience** is the biggest gap for a dispatch business. A PWA is enough at the start; later use React Native.
23. **Auto-dispatch:** rank partners by distance, rating, accept rate and price, and send offers in waves (top 3, then top 10).
24. **Notifications on WhatsApp** (booking confirmed, driver details, pickup reminders). Customers respond to WhatsApp far more than email.
25. **Dynamic pricing and surge rules**, festival/peak multipliers, and airport-specific pricing in `fareRules.priority`.
26. **Observability:** Sentry (frontend and backend), Cloud Logging with a booking-id correlation, uptime checks, alerts for failed payments and stuck bookings.
27. **BigQuery export** (Firebase extension) for analytics, so heavy reports never load Firestore.
28. **Testing:** unit tests for `fareEngine` and status transitions (these are where money bugs live), Firebase Emulator Suite for Rules tests, and Playwright for booking-to-payment happy path.
29. **Multi-tenant / white-label** (your Landing page lists "White-Label Solutions"): add `tenantId` to every top-level document from day one. It is very hard to add later.
30. **Environments:** separate Firebase projects for dev, staging and production, with seed scripts for cities, categories and fare rules.

---

## 11. Firestore Security Rules (starting sketch)

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {
    function signedIn()   { return request.auth != null; }
    function role()       { return request.auth.token.role; }
    function isStaff()    { return signedIn() && (role() == 'staff' || role() == 'admin'); }
    function isAdmin()    { return signedIn() && role() == 'admin'; }

    match /users/{uid}      { allow read: if signedIn() && (request.auth.uid == uid || isStaff()); allow write: if false; }
    match /customers/{uid}  { allow read: if signedIn() && (request.auth.uid == uid || isStaff()); allow write: if false; }
    match /bookings/{id} {
      allow read: if signedIn() && (
        resource.data.customerId == request.auth.uid ||
        isStaff() ||
        resource.data.assignment.partnerId == request.auth.token.partnerId   // partner only after assignment
      );
      allow write: if false;                       // all writes via Express (Admin SDK bypasses rules)
      match /timeline/{e} { allow read: if isStaff(); allow write: if false; }
    }
    match /dispatchOffers/{id} { allow read: if signedIn() && (isStaff() || resource.data.partnerId == request.auth.token.partnerId); allow write: if false; }
    match /vehicleCategories/{id} { allow read: if true; allow write: if false; }
    match /cities/{id}            { allow read: if true; allow write: if false; }
    match /fareRules/{id}         { allow read: if isStaff(); allow write: if false; }   // customers get computed prices only
    match /wallets/{id}           { allow read: if signedIn() && resource.data.ownerUid == request.auth.uid; allow write: if false; }
    match /{document=**}          { allow read, write: if false; }
  }
}
```
Because the Express server uses the Admin SDK, it bypasses these rules. That is why the rules can be strict, and why the API must check role and ownership itself on every route.

---

## 12. Decisions I need from you

1. **Confirm the roles table in Section 0.** Should a travel agency that *books* (your Register "organization") and a travel agency that *fulfils* (your "agent") be the same account type with two capabilities, or separate accounts? I designed them as separate (`customer.organization` and `partner.type = agency`), and the same person can hold both by having two profiles.
2. **Dispatch model:** manual assignment by staff only, offers to partners who then accept, or both? The schema supports both.
3. **Multi-tenant / white-label** needed? If yes, add `tenantId` now.
4. **Payments:** Razorpay only? Is a credit (postpaid) limit needed for agencies?
5. **Booking on behalf:** should staff be able to create bookings for phone customers? (Supported via `createdBy`.)
