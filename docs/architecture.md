# SwiftCart — Architecture

**Last updated:** 2026-07-20

This describes how SwiftCart is actually built, verified against the live database and
codebase. It is a technical companion to [product-overview.md](./product-overview.md).

---

## 1. Stack

| Layer | Technology |
|---|---|
| Apps | React Native · Expo SDK 54 · expo-router |
| Backend | Supabase (Postgres + Auth + Storage + Realtime) |
| Server logic | Postgres functions/triggers + 25 Deno edge functions |
| Payments | Paystack — **one account per delivery company** |
| Push | Expo Push |

Four separate repos, one shared Supabase project:

```
SwiftCart-customer   51 screens ·  7 edge fns   consumer marketplace
SwiftCart-vendor     65 screens ·  5 edge fns   shop/restaurant tools
SwiftCart-rider      51 screens ·  9 edge fns   delivery app (company-branded)
SwiftCart-admin      67 screens ·  4 edge fns   company admin + platform owner
```

The **admin repo is also the home of the shared layer**: the platform-wide docs
(`docs/`) and the shared database migrations (`supabase/migrations/`, 57 files covering
the multi-tenant work).

---

## 2. The tenancy model

The central idea: **a `company` is a tenant.** Almost everything operational hangs off a
`company_id`.

```
companies ── company_settings   (commission, delivery rates, branding, subscription)
         ├─ company_admins      (who administers it; is_platform flag = platform owner)
         ├─ company_secrets     (encrypted Paystack key — RLS-locked, no client access)
         ├─ merchants           (1 merchant belongs to exactly 1 company)
         ├─ rider_company_memberships  (riders can hold several, ONE active shift)
         └─ orders, earnings, withdrawals, promotions, disputes … (29 tables carry company_id)
```

**What is company-scoped:** merchants, orders, riders' memberships, earnings, payouts,
promotions, inventory, disputes, support tickets, marketing, branding, payment keys.

**What is platform-level (never company-scoped):** the customer roster and everything
hanging off it (addresses, wallets, loyalty, customer support tickets), platform
settings, and the platform admin roster. **The customer base is SwiftCart's asset — no
delivery company can see it.**

### The one deliberate exception: the storefront is nationwide

Customers are **not** tenant-scoped. A customer sees every approved store in the country
regardless of which company delivers it, because SwiftCart is a single consumer
marketplace. `merchants.company_id` means *"which company fulfils and takes commission on
this store's orders"* — **not** *"which storefront it appears in."*

This is served by `browse_merchants()`, a `SECURITY DEFINER` function that returns only
public storefront columns (never a merchant's phone, email, wallet balance or push
token) and hides merchants that are unapproved, subscription-expired, or belong to a
suspended company.

---

## 3. Security model

Every one of the **69 tables has row-level security enabled**, with **192 policies**.
Access is decided by three helper functions:

| Function | Answers |
|---|---|
| `is_platform_admin()` | Is this the SwiftCart owner? (sees everything) |
| `is_company_admin(company_id)` | Does this admin belong to that company? |
| `rider_active_company_ids()` | Which company is this rider on shift for? |

### Role vs scope are orthogonal — and this matters

- **Role** (`super_admin`, `ops_admin`, `support_admin`, `finance_admin`) decides *which
  actions* are allowed.
- **`company_id` + `is_platform`** decide *which data* is in scope.

A delivery company's admin legitimately holds `super_admin` — full power **over their own
company**. Therefore **platform-only screens gate on `is_platform`, never on role**;
gating on role would let every company admin walk straight through.

### A lesson encoded in the schema

RLS is permissive-OR: if *any* policy grants a row, access is granted. Twice during
development, a table had a correct company-scoped policy **OR'd with a legacy blanket
policy**, which silently defeated it. Both were found by impersonating a second company's
users, not by reading policy definitions.

Two rules came out of that, and they are documented in the migrations:

1. **A linter cannot find this class of bug.** Supabase's advisors never flagged either
   leak. Only actually simulating another tenant's traffic did.
2. **A policy on table A must not read a table B whose policies read A** — that causes
   infinite recursion and takes the table down. (This happened once, for ~10 minutes.)

---

## 4. Money architecture

**SwiftCart never touches order money.** Verified in `init-payment`, which refuses to
route a company's customer payment through the platform's key:

```
customer pays
   → init-payment resolves the merchant's company
   → charges that COMPANY's Paystack secret (decrypted server-side from company_secrets)
   → money lands in that company's Paystack balance
```

A company with no key stored is simply **not payment-ready** — there is no fallback to a
platform key.

### Settlement

On delivery, `handle_order_delivered` runs in-database and **freezes the economics onto
the order row**:

- `platform_commission_percentage` — the company's own rate, read from `company_settings`
- `platform_commission_amount`, `rider_earning_net`, `delivery_fee_gross`

Because it is snapshotted, changing a company's commission rate never rewrites history.
Verified: an order in a 15% company settles at 15% while a 10% company's order settles at
10%, from the same code path.

### Wallets are ledgers

`riders.wallet_balance` and merchant balances record **what is owed**, not cash held. The
cash is in the company's Paystack. Consequences:

- A company must **settle its obligations before it can be closed**
  (`company_outstanding_obligations()` reports them).
- A rider must **withdraw before transferring** to another company — otherwise a balance
  owed by company A would be orphaned once they belong to company B.

---

## 5. Company lifecycle

| Operation | Function | Rule |
|---|---|---|
| Create | `create_company()` | Platform admin only. Makes company + settings + admin row atomically. |
| Suspend | `deactivate_company()` | Soft, reversible. **Cannot suspend the platform home company.** |
| Reactivate | `reactivate_company()` | Restores everything. |
| Delete | `delete_company()` | **Refuses unless the company never operated** (zero orders/earnings/withdrawals/merchants/riders). |

**Suspension is enforced, not cosmetic.** A suspended company's stores disappear from the
customer app, new orders are blocked by a database trigger, and invitation claims are
refused. Its Paystack stays live so outstanding balances can still be paid out.

Hard delete exists only for demo/mistake tenants. Real companies are suspended — nothing
is ever destroyed, because 25 of the 26 foreign keys into `companies` are `NO ACTION`,
meaning Postgres itself refuses to delete a company with any history.

---

## 6. Onboarding & rider transfer

Onboarding is **invitation-based** (`invitations` + `claim_invitation`), keyed to a phone
number and claimed after normal OTP sign-in — so no extra SMS is sent (the project shares
a 30 SMS/hour cap).

**Rider transfer** rides the same flow. If a rider who is already active in company A
claims an invite to company B, it is treated as a transfer:

- `rider_transfer_preflight()` tells the rider app what will happen *before* they commit
- the transfer is blocked unless: **balance withdrawn** and **no delivery in progress**
- on success the old membership is deactivated and the new one becomes the active shift

The rider app explains this in plain language with tickable blockers, rather than failing
with an error code.

---

## 7. Known gaps

| Gap | Severity |
|---|---|
| `platform_settings` has a blanket authenticated read — a company admin can read platform commission defaults via direct API | Low (no PII, no cross-company data) |
| A customer/rider can read all columns of a merchant they have transacted with (RLS grants rows, not columns) | Low, bounded |
| Merchant transfer between companies — specified, not built | Feature gap |
| Company subscriptions recorded manually; no automated billing | By design for now |
| A few admin screens show harmless empty widgets for company admins (platform-only data) | Cosmetic |
