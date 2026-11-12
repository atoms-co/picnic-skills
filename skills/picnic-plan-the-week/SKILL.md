---
name: picnic-plan-the-week
description: >-
  Place several Picnic orders in one go through the Picnic MCP server — lunch
  for every day next week, a few test or random orders, one order per day.
  Use for "order lunch for each day next week", "place 5 random orders for
  tomorrow", "order something different for Thursday and Friday".
model: opus
---

# Plan the week

A Picnic cart holds **one delivery window at a time**, so several orders are
several complete cycles, one after another — never one big cart.

## Flow

### 1. Plan before touching the cart

- `picnic_get_profile` once (dietary settings, tax-inclusive prices).
- `picnic_get_delivery_windows` for each day; drop days with no windows or whose
  `order_by` has passed, and say which. A passed `earliest_restaurant_order_by`
  doesn't drop the day — some restaurants are closed, the rest still take orders.
- `picnic_get_order_history` to see what's already scheduled on those days, so
  you don't double-book.
- Pick a dish per order (`picnic-find-food`). For "something different" each
  time, vary restaurants as well as dishes.

### 2. Get approval for the whole plan

Show the plan as a table — day, window, dish, restaurant, estimated price — and
ask the eater to confirm it, or adjust.

### 3. For each order, one complete cycle

1. Make sure the cart is empty (`picnic_cart_get`); stop and ask if it holds
   items you didn't add.
2. Add the order's items for its window.
3. `picnic_preview_order` → get the exact total.
4. Place it — see the approval rule below.
5. Confirm the cart is clear before the next order.

## Approval for many orders

Each order still needs consent to what it will charge. Because the cart holds
one window, you can only preview an order once its cart is built, so:

- **Default — order by order:** build, preview, show the exact total, wait for a
  yes, place, then move on to the next order.
- **Faster — a cap the eater states:** if the eater explicitly sets a limit
  ("place them as long as each is under $15"), place each order whose preview
  total is within it without asking again, and stop to ask on any order above it.
  A cap must come from the eater; never infer one.

Either way, **any refusal stops the batch at that order** and goes back to the
eater (`picnic-checkout`): a new total — even a lower one — or a changed window
is their call, not something the batch instruction already covered.

## Report

A table of every order: day, window, items, amount charged, `receipt_url` — plus
any day skipped and why.

Then send one `picnic_feedback` covering the whole batch — what was placed, what
was refused or failed and why, and anything that slowed it down (see
`picnic-checkout`, step 5).

## Guardrails

- **"Place 5 random orders" is not approval of five totals.** It authorizes
  choosing and previewing.
- **Don't leave a half-built cart.** If you stop mid-batch, say what was placed,
  what wasn't, and what is still sitting in the cart.
