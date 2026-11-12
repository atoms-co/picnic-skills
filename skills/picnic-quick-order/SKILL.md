---
name: picnic-quick-order
description: >-
  Order a single Picnic meal end to end through the Picnic MCP server — e.g.
  "order me lunch tomorrow", "get me something with a fried egg for Friday",
  "order a noodle combo for tomorrow's lunch". Covers picking the window,
  finding the food, filling the cart, and placing it with the eater's approval.
model: opus
---

# Quick order

The default flow for "get me food for <day>". It strings the other skills
together in the order that has proven fastest — a clean run is about a dozen
tool calls and under two minutes.

## Flow

### 1. Load the eater's profile (once per conversation)

```
picnic_get_profile
```

Keep what it returns for the whole conversation, without being asked:

- `dietary_preferences` → pass one as `dietary_filter` when searching.
- `allergens` → steer away from items listing them.
- `include_tax_in_prices: true` → quote `price_with_tax` instead of `price`.
- `hub.time_zone` → read "tomorrow", "Friday", "lunch" in this time zone.

### 2. Pick the delivery window

```
picnic_get_delivery_windows  delivery_date=YYYY-MM-DD
```

- Map "lunch"/"dinner"/"afternoon" to the window's `display_name` and times.
- Check `order_by`: if it has passed, that window can't be ordered — say so and
  offer the next one rather than discovering it at checkout.
- If `earliest_restaurant_order_by` is present and has passed, the window is
  still open but some restaurants have closed; mention it, and if placement
  refuses a restaurant, pick another rather than giving up on the window.
- If the day has no windows (weekends, holidays), say so and offer the next day.

### 3. Find the food

Follow `picnic-find-food`. For a single obvious request one search is usually
enough; don't tour restaurants unless the ask is open-ended.

### 4. Look at the item before adding it

```
picnic_get_item  store_id=…  item_id=…  delivery_date=…  window_id=…
```

Required option groups must get a choice. Map what the eater asked for to real
`option_id`s; choose sensible defaults only when the eater said "anything" or
"random", and tell them what you picked. See `picnic-build-cart`.

### 5. Add to cart

```
picnic_cart_add_items  delivery_date=…  window_id=…  items=[…]
```

If the eater only asked to put it in the cart, **stop here** and report the cart.

### 6. Preview, get an explicit yes, place

Follow `picnic-checkout` exactly — preview, show the items, window and exact
total, wait for the eater's yes, then place with the preview's `order_id` and
`pricing.total`.

### 7. Report back and send feedback

Give the items, delivery window, amount charged and the `receipt_url` the
place call returned. Then send feedback on how the order went, whether it was
placed or not — see `picnic-checkout`, step 5.

## Guardrails

- **Casual phrasing is not approval.** "Order me lunch" authorizes building and
  previewing an order, not charging a total the eater hasn't seen.
- **One window per cart.** If the cart already holds items for another window,
  ask before clearing it — they may be someone's unplaced order.
- **Don't fabricate.** If the eater wants something the menu can't express
  (an option that doesn't exist, a note on an item that doesn't take notes),
  say so plainly.
- **When it can't be done here** (an option marked `needs_own_options`, a payment
  problem), hand over `picnic_get_checkout_link` so the eater finishes in Picnic.
