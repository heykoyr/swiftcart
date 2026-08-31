# SwiftCart — Business & Monetization Model

**Last updated:** 2026-07-20 · **Decided:** 2026-07-20

---

## The model in one sentence

**SwiftCart is a SaaS landlord.** Delivery companies pay a subscription to run their
business on SwiftCart's platform, apps and consumer brand. SwiftCart takes **no cut of
any order** and never handles customer money.

---

## Why not take a cut of orders?

Because SwiftCart legally cannot. Processing customer payments in Nigeria requires an
operating licence that unlocks a Paystack account, and SwiftCart could not obtain one.

That constraint became the business model. Instead of SwiftCart collecting money and
paying everyone out, **each delivery company brings its own licensed payment account**
and runs its own economy. SwiftCart sells the software and the demand.

This turns out to be a *stronger* position than a commission model:

| | Commission model | SwiftCart's model |
|---|---|---|
| Payment licence needed | Yes — blocker | **No** |
| Holds customer funds | Yes — float + liability | **No** |
| Settlement/chargeback risk | SwiftCart's | **The company's** |
| Revenue predictability | Volatile with order volume | **Fixed recurring** |
| Who owns the customer | SwiftCart | **SwiftCart** (unchanged) |

SwiftCart keeps the most valuable asset — the consumer brand and customer base — while
offloading the licensing burden, the float, and the payout risk to operators who are
already licensed to carry it.

---

## Who pays whom

```
Customer ──payment──▶ Delivery company's own Paystack
                          ├── vendor subtotal   → vendor
                          ├── rider net fee     → rider
                          └── commission        → the company keeps

Delivery company ──subscription──▶ SwiftCart      ← the ONLY SwiftCart revenue
```

- **Customers** pay for orders. Nothing to SwiftCart.
- **Vendors** pay nothing to SwiftCart. (A company may choose to charge its own vendors —
  that is the company's business decision, not the platform's.)
- **Riders** pay nothing. They earn a net delivery fee.
- **Delivery companies** pay SwiftCart a recurring subscription.

### Retired: the vendor subscription

Before the pivot, SwiftCart charged vendors to be listed. **That is retired.** The
platform does not earn from vendors. The `merchants.subscription_*` columns remain in the
database so historical data stays readable, but they are no longer a revenue mechanism
and the related dashboard alerts have been removed.

---

## How the subscription works

Deliberately simple, because it is **B2B with a handful of customers** — not a
self-service consumer product.

1. **Agree a price** with the company (monthly or yearly).
2. **They pay off-app** — bank transfer or invoice. No Paystack, no card on file. SwiftCart
   handles no money in-app, and this is no exception.
3. **Record it** in the Company Admin app → Settings → Companies → *Mark paid*. This
   writes a payment-ledger row and extends the paid period.
4. **They get warned** near expiry — a push notification you send from the same screen.
5. **On lapse**, suspend the company. Its stores leave the customer app, no new orders,
   no onboarding — but its payment account stays live so it can settle what it owes its
   riders and vendors. Fully reversible when they pay.

### Subscription states

Derived from the expiry date alone — a single source of truth:

| State | Meaning |
|---|---|
| `none` | No plan recorded yet |
| `active` | Paid, more than 7 days remaining |
| `expiring` | Paid, less than 7 days remaining |
| `expired` | Past the paid period |

This is kept **separate** from the company's operational status (`active` / `suspended`).
A company can be behind on payment but still operating (grace period), or suspended for a
non-payment reason. The two never got conflated into one column on purpose.

---

## Pricing

**Not yet set** — it is market-dependent and the owner's call. The recommended *structure*:

- **Start with one flat monthly (or yearly) fee per company.** Simplicity beats
  optimisation when you have fewer than five customers.
- **Add tiers only under real pricing pressure** — e.g. by rider count, vendor count or
  order volume, once there are 3–4 companies and the spread between the smallest and
  largest is obviously unfair.
- Yearly should carry a discount versus 12× monthly: it improves cash flow and reduces
  the churn conversation to once a year.

The system already stores `subscription_plan` and `subscription_amount` per company, so
different companies can be on different prices without any code change.

---

## What SwiftCart is actually selling

Worth being clear about, because it shapes the pitch:

1. **The consumer app and brand** — a nationwide storefront a company would otherwise
   have to build and market from scratch.
2. **The customer base** — SwiftCart's, not theirs. This is the moat. A company that
   leaves loses access to those customers.
3. **Four production applications** — customer, vendor, rider and admin, roughly 210
   screens, already built and working.
4. **The operational backend** — dispatch, live tracking, settlement, payouts, disputes,
   refunds, loyalty, promotions, marketing, fraud flags, audit logs.
5. **Their own brand where it counts** — the rider and admin apps carry the company's
   logo and colours, so their staff and riders experience it as *their* system.

A delivery company buying this is buying a working national marketplace and a full
software department for a fixed monthly fee.

---

## Unit economics (illustrative)

SwiftCart's costs are essentially fixed infrastructure — Supabase, push, maps — and do
**not** scale with order volume, because SwiftCart processes no payments and moves no
money. Each additional company is close to pure margin, with support being the main
variable cost.

The practical growth question is therefore not "how many orders?" but **"how many
licensed delivery companies can be signed?"** — a much smaller, more tractable sales
target than consumer acquisition.
