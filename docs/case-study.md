# Case Study — Turning a Licensing Blocker into an Architecture

**SwiftCart · 2026**

---

## The problem

SwiftCart was built as a single-operator delivery marketplace: one company (SwiftCart)
with its own riders, its own vendors, and its own customers. Four production apps, ~210
screens, a full operational backend — customer ordering, dispatch, live tracking,
settlement, payouts, disputes, loyalty, promotions, marketing.

Then it hit a wall that no amount of code could solve.

**To take customer payments in Nigeria, you need an operating licence** that unlocks a
Paystack merchant account. SwiftCart could not obtain one. A finished marketplace could
not legally collect a naira.

The obvious responses were both bad:

- **Wait for the licence.** Indefinite, outside the team's control, with a finished
  product idle.
- **Rebuild as something smaller.** Throw away a working system to fit the constraint.

---

## The insight

The licence problem was not *SwiftCart's* problem to solve. **Other businesses already
had it.**

There are licensed delivery companies with Paystack accounts, riders on the road, and no
software worth using. SwiftCart had the software and no licence. The two are exactly
complementary.

So the product was inverted:

> **SwiftCart stops being an operator and becomes a platform.** Licensed delivery
> companies become the operators. They bring the licence and the riders; SwiftCart brings
> the apps, the consumer brand and the customers.

Critically, this was **not** a retreat. SwiftCart keeps the most defensible asset — the
consumer brand and customer base — and offloads the licensing burden, the float, and the
payout liability to parties already regulated to carry them.

---

## What that required

Turning a single-tenant product into a multi-tenant platform, **on a live database with
real orders and real money**, without breaking the existing operation.

### 1. Tenancy
A `company` became the tenant. 29 tables gained a `company_id`. Merchants, orders, riders,
earnings, payouts, promotions, disputes and branding all became company-scoped — across 69
tables, every one with row-level security, and ~190 policies.

### 2. Money routing
The hardest inversion. Customer payments had gone to one platform Paystack account. Now
each payment must reach **the fulfilling company's own account**, resolved server-side from
the merchant's company. The payment function was written to *refuse* to fall back to a
platform key — a keyless company is simply not payment-ready.

Settlement had to follow: each company sets its own commission, snapshotted onto the order
at delivery so rate changes never rewrite history.

### 3. One deliberate exception
Almost everything is tenant-scoped — but **the customer storefront is not**. SwiftCart is a
single consumer brand; a customer sees every nearby shop regardless of who delivers it.
Getting this wrong in the first pass (scoping customers to companies) and correcting it was
one of the more valuable course-corrections in the project.

### 4. Monetization
The old vendor subscription was retired — the platform earns nothing from vendors now.
Revenue became a **company subscription**: fixed, recurring, invoiced off-app. SwiftCart's
costs no longer scale with order volume, because it processes no payments at all.

---

## What the rehearsal found

Rather than trust that the isolation worked, a second company ("CarryGo") was created on
the live database with deliberately different economics, and every role was **impersonated
at the database level** — company admin, rider, platform admin — to see what each could
actually reach.

It found real cross-tenant leaks that reading the policies had not:

- Any authenticated user of any company could read **every merchant's phone number and
  wallet balance**.
- A brand-new company's admin could see another company's **products and promotions**.

The pattern was the same each time: a correct company-scoped policy **OR'd with a legacy
blanket policy**, which silently defeated it. All were closed and re-verified.

**Two lessons worth generalising:**

1. **Automated security linters did not find any of it.** Supabase's advisors flagged
   nothing before or after. A linter sees that a policy exists; it cannot know that
   `auth.role() = 'authenticated'` on a tenant table means "everyone sees everyone." Only
   simulating a second tenant's traffic exposed it.

2. **Proof beats reasoning.** Every claim about isolation was checked by impersonation and
   transactional dry-runs that rolled back, not by inspecting SQL. That discipline is what
   turned "it should be isolated" into "here is the row count each identity sees."

One genuine outage happened during the work — a new policy created infinite recursion and
took merchant reads down for about ten minutes. It was caught by a dry-run, reverted, the
cause documented in the migration, and the rule ("a policy on A must not read a B whose
policies read A") written into the codebase so it cannot recur silently.

---

## Where it landed

| Before | After |
|---|---|
| Single operator, blocked on a licence | Platform licensed to any number of licensed operators |
| SwiftCart holds all customer money | SwiftCart holds **none** |
| Revenue = commission (unobtainable) | Revenue = fixed company subscriptions |
| Costs scale with order volume | Costs are ~fixed; each company is near-pure margin |
| Growth = consumer acquisition | Growth = **sign licensed companies** |

Three companies exist on the platform today — all seeded tenants, used to prove isolation
rather than signed operators. Onboarding a new one — company, branding,
payment key, riders, vendors — takes **under an hour**, and the exact sequence is written
down as a runbook that requires no interpretation.

---

## The transferable lesson

A regulatory blocker looked like the end of the product. Treated as an architectural
input instead, it produced a business that is **less capital-intensive, less risky and
more defensible** than the one originally planned.

The constraint didn't shrink the product. It clarified what the product actually was:
not a delivery company, but the software and the demand behind many of them.
