---
name: payment-system-design
description: |
  Guide for designing SaaS subscription + credit billing systems with Stripe. Use this skill whenever building payment flows, subscription management, credit/token billing, Stripe webhook handling, or usage-based pricing. Also trigger when the user mentions: billing, subscriptions, credits, Stripe integration, payment webhooks, metered billing, overage charging, subscription lifecycle, or cancellation flows. This skill captures battle-tested patterns and critical pitfalls from production systems — use it to avoid common mistakes that cause revenue loss or billing errors.
---

# SaaS Payment System Design Guide

A comprehensive reference for building subscription + credit billing systems with Stripe. Every pattern and pitfall documented here comes from real production incidents — not theory.

## When to Use This Skill

- Designing a new SaaS billing system
- Adding Stripe subscriptions to an existing product
- Implementing credit/token-based billing (like API credits)
- Building Stripe webhook handlers
- Debugging billing bugs (duplicate charges, missing credits, race conditions)

---

## Part 1: Subscription Lifecycle State Machine

This is the most important thing to get right. Every subscription system needs a clear state machine.

```
[free] ─── user subscribes ──→ [active]
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
        user cancels        payment fails       user upgrades/
        auto-renew                                downgrades
              │                   │                   │
              ▼                   ▼                   │
         [canceling]         [past_due]               │
              │                   │                   │
     period ends    ┌─────────────┼──────────┐        │
              │     │             │          │        │
              ▼     ▼             ▼          ▼        │
         [expired] [active]    [unpaid]  [suspended]  │
                  (retry ok)  (all fail) (dispute)    │
                                                      │
              user resubscribes ◄─────────────────────┘
```

### State Definitions

| State | API Access | Credits | Billing |
|-------|-----------|---------|---------|
| `free` | Yes (limited) | One-time grant (e.g., 100) | None |
| `active` | Yes | Monthly grant | Auto-renew |
| `canceling` | Yes (until period end) | Usable, no new grant | Won't renew |
| `past_due` | Yes (grace period) | Usable | Stripe retrying |
| `unpaid` | **No (403)** | Frozen | Stopped |
| `expired` | **No (403)** | Cleared | Ended |
| `suspended` | **No (403)** | Frozen | Dispute under review |

### Transitions That Are Easy to Miss

1. **`canceling` → `active`**: User cancels, then changes their mind same day. Stripe sets `cancel_at_period_end` back to `false`. Your webhook handler MUST detect this toggle and transition back to `active`.

2. **Upgrade while `past_due`**: User owes money on old plan but upgrades. Old invoice still retrying. Don't grant credits based on the old invoice — check which price the invoice corresponds to.

3. **Mid-cycle downgrade**: Stripe prorates and may generate a $0 or negative invoice. A $0 invoice may not fire `invoice.paid` depending on your Stripe settings. Don't rely solely on `invoice.paid` for credit adjustments on downgrades.

4. **`trialing` → `active`** or **`trialing` → `past_due`**: If you offer a Stripe trial period, handle `customer.subscription.trial_will_end` (fires 3 days before trial ends — send "trial ending" email) and the transition when trial converts or fails. During `trialing`, grant trial credits but don't charge. On conversion (`invoice.paid` for first real invoice), grant full monthly credits.

---

## Part 2: Critical Pitfalls (WILL Cause Revenue Loss If Ignored)

### Pitfall 1: Webhook Event Ordering

**The problem:** Stripe does NOT guarantee webhook delivery order. You can receive `invoice.paid` before `customer.subscription.created`, or `subscription.updated` before `invoice.paid`.

**What goes wrong:** You grant credits based on `invoice.paid`, but look up the tier from your local DB. If `subscription.updated` (which changes the tier) hasn't arrived yet, you grant credits for the wrong tier.

**The fix:** On `invoice.paid`, ALWAYS read the current subscription state from the Stripe API:

```python
# WRONG — uses local cache
subscription = db.get_subscription(user_id)
credits = TIERS[subscription.tier]["monthly_credits"]

# RIGHT — reads fresh from Stripe
stripe_sub = stripe.Subscription.retrieve(invoice.subscription)
tier = price_to_tier(stripe_sub.items.data[0].price.id)
credits = TIERS[tier]["monthly_credits"]
```

### Pitfall 2: Credit Balance Race Condition

**The problem:** User has 1 credit left, makes 2 API calls simultaneously. Both requests read `balance = 1`, both pass the check, both decrement. Balance goes to -1.

**What goes wrong:** You give away free API calls. At scale, this is systematic revenue leakage.

**The fix:** Atomic database operation. NEVER do SELECT then UPDATE separately:

```sql
-- WRONG (race condition)
SELECT balance FROM user_credits WHERE user_id = ?;
-- if balance > 0:
UPDATE user_credits SET balance = balance - 1 WHERE user_id = ?;

-- RIGHT (atomic)
UPDATE user_credits 
SET balance = balance - 1 
WHERE user_id = ? AND balance >= 1 
RETURNING balance;
-- If 0 rows affected → reject the request
```

### Pitfall 3: Missing 3D Secure / SCA Event

**The problem:** EU customers under PSD2/SCA may need to authenticate renewal payments. Stripe fires `invoice.payment_action_required` — NOT `invoice.paid` and NOT `invoice.payment_failed`. If you don't handle this, the subscription gets stuck: not paid, not failed, no credits granted.

**The fix:** Handle `invoice.payment_action_required`. Set subscription to a `requires_action` state. Email the user with `invoice.hosted_invoice_url` so they can complete 3D Secure.

### Pitfall 4: Overage Revenue Lost on Cancellation

**The problem:** User incurs overage (uses more credits than their plan includes), then cancels before the next invoice. If you batch-report usage to Stripe, the cancellation happens before your report → overage never gets billed.

**The fix:** Report metered usage to Stripe in real-time (or every few minutes via background job), NOT at end-of-cycle. When you receive a cancellation webhook, immediately report any unreported usage before processing the cancellation.

### Pitfall 5: Webhook Duplicate Delivery

**The problem:** Stripe can deliver the same event multiple times. If your `invoice.paid` handler isn't idempotent, you grant credits twice.

**The fix:** Store processed `event.id` values in a `processed_webhooks` table. Check before processing:

```python
async def handle_webhook(event):
    if await already_processed(event.id):
        return 200  # Already handled
    
    # Process the event...
    
    await mark_processed(event.id, source="stripe")
    return 200
```

### Pitfall 6: Missing Refund Handling

**The problem:** User requests a refund or you issue a partial refund via Stripe Dashboard. `charge.refunded` fires, but your system still shows the user as active with full credits. Or worse — you refund but don't reclaim the credits they already consumed.

**What goes wrong:** Users learn they can consume credits, request a refund, and keep the credits. Free money.

**The fix:** Handle `charge.refunded` and `charge.refund.updated`:
- **Full refund:** Set subscription to `expired`, clear remaining credits, deactivate API keys
- **Partial refund:** Log in credit_ledger as a negative adjustment, but don't necessarily revoke access — this is usually a goodwill gesture
- **Credits already consumed:** You generally can't claw back consumed credits. Accept this as cost of doing business, but track it. If a user repeatedly refunds after consuming, that's abuse — flag for review.

### Pitfall 7: `server_default` vs `default` in ORM

**The problem:** SQLAlchemy's `default=func.now()` only works through the ORM. Raw SQL (used in seeds, migrations, webhook handlers) won't get the default → NOT NULL violation.

**The fix:** Always use `server_default` alongside `default` for timestamp columns:

```python
created_at = mapped_column(
    DateTime(timezone=True), 
    default=_utcnow,              # ORM insert
    server_default=text("NOW()")  # Raw SQL / DB-level default
)
```

---

## Part 3: Credit System Design

### Reset vs Accumulate

**Reset (recommended for most SaaS):** User gets a fresh allocation each billing cycle. Unused credits expire.

Pros:
- Simple billing model, easy to explain
- Prevents credit hoarding → predictable revenue
- No financial liability from accumulated unused credits

Cons:
- "I paid for credits I didn't use" feels wasteful to users
- Creates end-of-month usage spikes (use-it-or-lose-it)

**Recommended hybrid: Reset + 20% rollover cap**

```python
unused = current_balance  # Before reset
new_balance = min(
    monthly_credits + int(unused * 0.2), 
    int(monthly_credits * 1.2)
)
```

This softens the perception without creating unbounded liability.

### Credit Ledger Pattern

Use an append-only ledger for all credit changes. Every operation (grant, deduction, adjustment) is a row with `balance_after`:

```sql
CREATE TABLE credit_ledger (
    id BIGSERIAL PRIMARY KEY,
    user_id TEXT NOT NULL,
    delta INT NOT NULL,           -- positive = grant, negative = deduction
    balance_after INT NOT NULL,   -- snapshot after this operation
    reason TEXT NOT NULL,         -- "monthly_grant" | "api_call" | "overage" | "manual"
    reference_id TEXT,            -- invoice_id, api_call_id, etc.
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

The actual balance lives in a separate `user_credits` table for fast atomic updates. The ledger is the audit trail.

### Metered Overage with Stripe

For credits-included-then-overage pricing, use Stripe's metered billing:

1. Create a metered price in Stripe (usage_type: "metered", aggregate_usage: "sum")
2. When user exceeds included credits, report overage units via `stripe.SubscriptionItem.create_usage_record()`
3. Stripe includes overage on the next invoice automatically

Report usage frequently (every few minutes), not at end of cycle — this prevents the cancellation revenue loss pitfall.

---

## Part 4: Stripe Webhook Event Checklist

Every SaaS billing system should handle these events. Missing any one of them creates a real failure mode.

| Event | What to Do | Why It Matters |
|-------|-----------|---------------|
| `customer.subscription.created` | Create subscription record, set tier | New paying customer |
| `customer.subscription.updated` | Update tier/status, detect `cancel_at_period_end` toggle | Upgrades, downgrades, cancel/uncancel |
| `customer.subscription.deleted` | Set expired, report unreported usage, clear credits | Subscription ended |
| `invoice.paid` | Grant credits (read tier from Stripe API, not cache), apply 20% rollover | New billing cycle |
| `invoice.payment_failed` | Set `past_due`, notify user | Card expired / declined |
| `invoice.payment_action_required` | Set `requires_action`, email user with payment link | 3D Secure / SCA (EU customers) |
| `charge.dispute.created` | IMMEDIATELY suspend user (deactivate all API keys) | Chargeback = fraud signal |
| `customer.subscription.paused` | Suspend API access | Stripe pause feature |
| `customer.subscription.trial_will_end` | Email "trial ending in 3 days" | Convert trial users |
| `charge.refunded` | Full: expire + clear credits. Partial: log adjustment | Refund handling |

### Webhook Handler Template

```python
@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig = request.headers.get("stripe-signature")
    
    try:
        event = stripe.Webhook.construct_event(payload, sig, WEBHOOK_SECRET)
    except (ValueError, stripe.SignatureVerificationError):
        raise HTTPException(400, "Invalid signature")
    
    # Idempotency check
    if await already_processed(event.id):
        return {"status": "already_processed"}
    
    # Route to handler
    handler = HANDLERS.get(event.type)
    if handler:
        await handler(event.data.object)
    
    await mark_processed(event.id, source="stripe")
    return {"status": "ok"}
```

---

## Part 5: Database Schema Essentials

### Minimum Tables

1. **`subscriptions`** — Stripe sync (status, tier, period_start/end, cancel_at_period_end, stripe_subscription_id, stripe_customer_id)
2. **`user_credits`** — Current balance (atomic updates via SQL)
3. **`credit_ledger`** — Append-only audit trail
4. **`api_keys`** — Per-user keys with tier + rate_limit
5. **`usage_logs`** — Per-call tracking for billing reconciliation
6. **`processed_webhooks`** — Event deduplication (event_id + source)

### API Key Design

- Use a project-specific prefix for identification: `{project}_live_` (production), `{project}_test_` (sandbox). Examples: `acme_live_`, `tickr_live_`. The prefix makes keys instantly identifiable in logs and support tickets.
- Generate with `secrets.token_urlsafe(32)`
- Store tier + rate_limit on the key itself (avoid JOIN on every request)
- Soft-delete (set `is_active=false`), never hard-delete
- Show masked in UI: `acme_live_a8f3...x9k2` (first 8 + last 4 chars)

### Rate Limiting

In-memory sliding window per API key is fine for single-process deployments:

```python
from collections import defaultdict, deque
import time

_windows: dict[str, deque] = defaultdict(deque)

def check_rate_limit(key_id: str, max_per_minute: int):
    now = time.time()
    window = _windows[key_id]
    while window and window[0] < now - 60:
        window.popleft()
    if len(window) >= max_per_minute:
        raise HTTPException(429, "Rate limit exceeded")
    window.append(now)
```

For multi-process: use Redis with `MULTI/EXEC` or a Lua script.

---

## Part 6: Tax & Multi-Currency

These feel like "later" problems but bite hard if not considered early.

### Stripe Tax

Stripe Tax automatically calculates and collects sales tax, VAT, and GST. Enable it early — retrofitting tax onto existing subscriptions is painful.

- Enable in Stripe Dashboard → Settings → Tax
- Set your business address (determines nexus/obligations)
- Add `automatic_tax: {"enabled": True}` to Checkout Session creation
- Stripe handles which customers owe tax and how much

If you're selling to EU customers, you likely need to collect VAT. Stripe Tax handles this, but you need to register for VAT in relevant jurisdictions (or use Stripe's "one-stop shop" registration).

### Multi-Currency

Stripe supports 135+ currencies. Decide early:

- **Single currency (simplest):** Price everything in USD. International customers pay in USD. No currency logic needed.
- **Multi-currency pricing:** Create separate Stripe Prices per currency (e.g., `price_usd_49`, `price_eur_45`). Frontend detects locale and shows the right price. More work but better conversion.

For MVP, single currency (USD) is fine. Add multi-currency when international revenue exceeds 20% of total.

---

## Part 7: Common Mistakes Checklist

Before launching, verify:

- [ ] Webhook handlers are idempotent (processed_webhooks table)
- [ ] Webhook handlers don't rely on event ordering (read from Stripe API)
- [ ] Credit deduction uses atomic SQL (UPDATE WHERE balance >= cost)
- [ ] `invoice.payment_action_required` is handled (3DS/SCA)
- [ ] `charge.refunded` is handled (full refund → expire, partial → log)
- [ ] `customer.subscription.trial_will_end` is handled (if using trials)
- [ ] Overage usage is reported to Stripe in near-real-time
- [ ] Cancellation flow reports unreported usage before processing
- [ ] `cancel_at_period_end` toggle is detected (canceling ↔ active)
- [ ] All timestamp columns have `server_default` for raw SQL compatibility
- [ ] API keys use soft-delete, not hard-delete
- [ ] API key prefix is project-specific (`{project}_live_`)
- [ ] Global exception handler doesn't leak Stripe keys or internal state
- [ ] Chargeback (`charge.dispute.created`) immediately suspends user
- [ ] Stripe Tax enabled if selling to EU customers (VAT requirement)
- [ ] Checkout Session includes `automatic_tax: {"enabled": True}` if tax is active
