---
name: picnic-checkout
description: >-
  Place a Picnic order through the Picnic MCP server safely — preview, get the
  eater's explicit approval of the exact total, place, and handle every refusal
  (price changed, delivery window changed, payment declined) — and answer what
  a stipend, team order or discount covers. Use whenever an eater says "place
  it", "order it", "check out", a batch request will end in placed orders, or
  the eater asks "is lunch covered?", "what's my stipend?", "is there a team
  order?", "who's ordered in my team order?" or "any discounts?".
model: opus
---

# Checkout

Placing an order charges the eater's real saved payment method. The server is
built so that nothing an agent invents can change what is charged — but only if
the agent follows this sequence.

## Flow

### 1. Preview

```
picnic_preview_order  delivery_date=…  window_id=…
```

Nothing is placed or charged. It returns the cart's lines with options, the
delivery window, `pricing` (subtotal, discount, tax, delivery fee, total) and an
`order_id`. The total is already after stipends and automatic discounts — it can
be $0.

If `validation_issues` is not empty (an item became unavailable, a stale price),
fix the cart first (`picnic-build-cart`) and preview again.

### 2. Show it and ask

Show the items, the delivery window (in the hub's time zone) and the **exact
`pricing.total`**, and ask the eater to confirm. Wait for an explicit yes.

### 3. Place

Only after the eater's yes, call `picnic_place_order` as its description
directs, for the previewed order and the total the eater approved. On success,
you get the order, the amount charged and a receipt link; the ordered lines are
removed from the cart.

If the call fails as `UNAVAILABLE` saying Picnic didn't confirm the order in
time, call again with the **same `order_id`**: it can never charge twice. Picnic
checks for the earlier attempt first, so a retry either reports the order it
already placed (`already_placed: true`, nothing charged again, the eater not
asked again) or goes on to place it. If the retry itself says Picnic couldn't
tell whether the earlier attempt placed the order, don't mint a new `order_id`
and don't retry blindly: check `picnic_get_order_history`, then retry with the
same `order_id` once the history is readable.

### 4. Report

Items, window, amount charged, and the receipt link exactly as returned. If the
response says the cart wasn't cleared, remove the ordered items before building
the next order.

### 5. Send feedback

After every placement attempt — placed, refused or failed — tell the Picnic
team how it went, with the eater's consent as `picnic-feedback` describes: by
default their client's prompt asks them when `picnic_get_profile` reported
`prompts_in_client: true` (don't also ask in chat); when it reported `false`,
ask in chat and pass `eater_consent: given`; `eater_consent: always_send` if
they've chosen always, and no call at all if they've said never:

```
picnic_feedback  message=…  tool_name=picnic_place_order  [kind=…]  [eater_prompt=…]  [eater_consent=…]
```

- **Placed:** a sentence on how the order went — what the eater asked for, how
  many tries it took, anything that was slow, confusing or needed a workaround.
  A smooth order is worth reporting too.
- **Refused or failed:** what Picnic said, what you and the eater did next, and
  whether the order was placed in the end. Use `kind: incomplete_result` when a
  tool didn't give you what you needed.

Include the eater's original request in `eater_prompt` when it helps. Leave out
payment details and anything personal beyond what the request itself says. For
a batch of orders, send one submission at the end that covers all of them. If
the send fails, keep any consent choice from the error's `details.consent` and
don't trouble the eater with it: this report is yours, not theirs (see
`picnic-feedback`, "When sending fails").

## Stipends, team orders and discounts

Three tools read what Picnic would take off an order and the team orders behind
it. None changes anything, so call them whenever the eater asks what's covered,
or to explain a discount on a preview.

| The eater asks | Call | What it tells you |
|---|---|---|
| "Is lunch covered Thursday?", "what's left of my stipend?", "is there a team order this week?" | `picnic_get_benefits  delivery_date=…  [days=1..7]` | Per delivery window: the stipends usable there and what's left of them, the discounts Picnic would apply (stipend, team order, meal pass, other funded meals) with amounts, and, when the cart has items, what the eater would still pay |
| "Any discounts?", "do I have a coupon?" | `picnic_list_discounts` | Promotions the restaurants in the cart advertise, and the team order and other funded coupons the eater created |
| "Who's ordered in my team order?", "what do I owe for it?", "when's the cutoff for the team lunch?" | `picnic_get_team_order  [team_order_id=…]` | Without an ID, the team orders the eater organised. With one, that team order: window, order-by cutoff, what the organiser covers — and, for its organiser only, each eater's order, what is owed and who was invited |

- **Coverage depends on the window.** A stipend or a team order someone else
  funds applies to particular hubs, days and times, so ask `picnic_get_benefits`
  about the window in question; `days` covers up to a week when the eater wants
  to find a covered one. A window with no stipends means none can be used there.
- **Picnic applies most of it for you.** At placement it takes the eater's best
  stipend and the biggest restaurant promotion the cart qualifies for; there is
  nothing to enter. A coupon the eater created (`own_coupon`) is the exception:
  it isn't applied here, so send them to `picnic_get_checkout_link` to enter its
  code in Picnic.
- **These are estimates; the preview is the total.** Amounts from either tool
  explain a price, but the number the eater approves is
  `picnic_preview_order`'s `pricing.total`.
- **Empty is an answer.** No stipends and no discounts means nothing covers that
  window or cart; say so instead of guessing at a benefit.
- **A team order's ID** comes from `picnic_get_team_order`'s own list, from
  `team_order` on a past order in `picnic_get_order_history`, or from the invite
  link the eater was sent. Someone who was only invited sees the team order's
  details, not who else ordered.
- **Other eaters' names, orders and emails are the organiser's to see.** Show
  them to the organiser who asked; don't repeat them elsewhere or send them in
  feedback.
- **Reading only.** Creating a team order, inviting people to one or cancelling
  it happens on the Picnic website (`picnic-connection-help`).

## When placement is refused

If Picnic refuses to place the order — the total changed, the delivery window
changed, the payment didn't go through, or ordering for the window has closed —
nothing is charged. **Tell the eater what Picnic said and let them decide; never
decide it yourself.** Give them the Picnic website link as well
(`picnic_get_checkout_link` for the order's window), so they can finish there if
they'd rather; `picnic-connection-help` has the rule. Then follow the refusal's
own instructions, and send feedback once it's settled (step 5).

## Rules

- **The yes must be for the exact total.** "Order me lunch", "place the order for
  me" or "place 5 random orders" authorize building and previewing, not charging
  a total the eater hasn't seen.
- **Refusals reset consent.** A refusal names a new fact, such as a new total or
  window; the earlier yes doesn't cover it — even when the new total is lower.
- **One preview per order.** Preview again whenever the cart, window or total
  changes.
- **Don't announce success early.** Only report an order once `picnic_place_order`
  returned it; check `picnic_get_order_history` if the result was unclear.
