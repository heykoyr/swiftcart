# SwiftCart — Product Overview

**Last updated:** 2026-07-20 · **Status:** pre-launch (not yet in public app stores)

---

## What SwiftCart is, in one paragraph

SwiftCart is a **nationwide consumer delivery marketplace** whose deliveries are
fulfilled by **independent, licensed delivery companies**. Customers open one
SwiftCart-branded app, see every nearby store, and order. Behind the scenes each store
is serviced by a delivery company that runs its own riders, holds its own payment
account, and sets its own economics. SwiftCart itself is the **software platform and
consumer brand** — it licenses the whole stack to those companies and takes no cut of
any order.

> **The one-line version:** SwiftCart is the storefront and the software; delivery
> companies are the operators. Customers see SwiftCart. Companies do the delivering.

---

## Why it is built this way

This structure is not an abstract design preference — it comes from a hard constraint.

Handling customer payments in Nigeria requires an operating licence that unlocks a
payment-processor (Paystack) account. SwiftCart could not obtain that licence. Rather
than stall, the product was restructured so that **the businesses that already hold that
licence do the money-handling**, and SwiftCart provides everything else.

That single decision defines the whole architecture:

- A "company" on SwiftCart is, by definition, **a licensed business with its own
  Paystack account**.
- **Customer payments never touch SwiftCart.** They go directly into the fulfilling
  company's own payment account.
- SwiftCart therefore needs no payment licence, holds no customer funds, and carries no
  settlement risk.
- SwiftCart earns from **software subscriptions paid by the companies**, not from order
  volume.

---

## The four apps

| App | Who uses it | Branding |
|---|---|---|
| **Customer app** | Shoppers, nationwide | Always SwiftCart |
| **Vendor app** | Shops/restaurants selling on the marketplace | Always SwiftCart |
| **Rider app** | Delivery riders employed by a company | **The company's brand** |
| **Company Admin app** | A delivery company's operations staff — and the SwiftCart platform owner | **The company's brand** |

**Why only two apps are white-labelled:** SwiftCart is the *consumer* brand. A customer
should never have to know or care which company delivers their order — they just see
SwiftCart. But a rider works *for* a specific company, and a company's ops team manages
*their own* business, so those two surfaces carry the company's identity.

There is no separate "super admin" app. The Company Admin app has **two faces**: a
delivery company's admin sees only their own company, while the platform owner sees
everything and can manage all companies. The difference is enforced in the database, not
just the interface.

---

## Who each party is and what they get

### Customers
One nationwide app. Browse every nearby store sorted by distance, order, pay, track the
rider live, rate the delivery. They never see companies at all. → [for-customers.md](./briefs/for-customers.md)

### Vendors (shops & restaurants)
List products, receive and accept orders, manage stock and promotions, get paid out.
They pay SwiftCart nothing. → [for-vendors.md](./briefs/for-vendors.md)

### Riders
Work for one delivery company at a time. Accept jobs, navigate to store and customer,
get paid a net delivery fee into a wallet they withdraw from. Can transfer between
companies. → [for-riders.md](./briefs/for-riders.md)

### Delivery companies
License the whole platform. Bring their own payment account and riders, onboard their
own vendors, set their own commission and delivery rates, brand their rider and admin
apps. Pay SwiftCart a subscription. → [for-companies.md](./briefs/for-companies.md)

### SwiftCart (the platform)
Owns the consumer brand, the customer base, and the software. Onboards companies,
manages their subscriptions, and can suspend a company that stops paying or operating
properly. → [monetization.md](./monetization.md)

---

## How money moves (important, and often misunderstood)

```
Customer pays for an order
        │
        ▼
The FULFILLING COMPANY's own Paystack account        ← SwiftCart never touches this
        │
        ├── vendor's subtotal        → owed to the shop
        ├── rider's net delivery fee → owed to the rider
        └── commission               → the company keeps this
                                        (each company sets its own rate)

SwiftCart's revenue is entirely separate:
        Company ──── monthly/yearly subscription ────▶ SwiftCart
                     (invoiced off-app, recorded manually)
```

Two consequences worth internalising:

1. **Wallet balances are bookkeeping, not cash.** A rider's or vendor's "wallet balance"
   is a record of *what they are owed*. The actual money sits in their company's Paystack
   account and is paid out from there. This is why a company must settle its balances
   before it can be closed, and why a rider must withdraw before transferring companies.

2. **Each company is a closed economy.** Company A's money, orders, riders and vendors
   are entirely separate from Company B's — enforced at the database level, not by UI
   convention.

---

## Current state

**Built and live in the production database:**

| | |
|---|---|
| Apps | 4 (234 screens total) |
| Edge functions | 25 |
| Database tables | 69 — **all** with row-level security |
| RLS policies | 192 |
| Company-scoped tables | 29 |
| Companies on the platform | 3 — all seeded tenants, not signed operators |

**Working end-to-end:** customer ordering and payment, per-company payment routing, order
dispatch to the right company's riders, delivery tracking, settlement with per-company
commission, vendor and rider payouts, disputes, refunds, loyalty, promotions, marketing,
company onboarding/suspension, rider transfer between companies.

**Not yet done:**
- Not released to app stores (no public users yet; the only tester is the owner)
- Company subscriptions are recorded manually — no automated billing

---

## Where to read more

| Document | What it covers |
|---|---|
| [architecture.md](./architecture.md) | How it is built: apps, database, tenancy, security, money |
| [monetization.md](./monetization.md) | The business model and how SwiftCart earns |
| [for-companies.md](./briefs/for-companies.md) | Pitch and brief for delivery companies |
| [for-vendors.md](./briefs/for-vendors.md) | What vendors get |
| [for-riders.md](./briefs/for-riders.md) | What riders get |
| [for-customers.md](./briefs/for-customers.md) | What customers get |
| [case-study.md](./case-study.md) | The pivot story: constraint → architecture |
| [design-system.md](./design-system.md) | Tokens, type scale, motion, components, contrast |
