---
name: picnic-build-cart
description: >-
  Build and edit the eater's Picnic cart through the Picnic MCP server —
  choosing required and optional options (including nested choices), notes,
  quantities, removing items, and clearing the cart. Use for "add it with a
  soft-boiled egg", "make it two", "put it in my cart but don't order",
  "clear my cart".
model: opus
---

# Build the cart

The cart is the eater's real Picnic cart: what you add here shows up in the
Picnic app and website, and the app's changes show up here.

## Cart rules to know first

- **One hub and one delivery window per cart.** Adding for another window fails.
  To order for a different window, place or clear the current cart first —
  and ask before clearing items the eater didn't ask you to add.
- **Many restaurants per cart are fine.**
- **A passed window can't take items.** Check `order_by` from
  `picnic_get_delivery_windows`. A passed `earliest_restaurant_order_by` only
  means some restaurants have closed; the window itself is still open.
- **Read it, don't remember it.** Call `picnic_cart_get` for the current
  contents and pricing; the eater may have changed the cart in the app.

## Options

Look the item up first:

```
picnic_get_item  store_id=…  item_id=…  delivery_date=…  window_id=…
```

- Every group with `required: true` needs a choice; respect `min_choices` and
  `max_choices`.
- An option can carry its own `option_groups` (a combo's noodle choice with its
  own egg choice). Pass those choices in that option's `options`:

```
items=[{"store_id":"…","item_id":"…","quantity":1,
  "options":[{"group_id":"<noodles>","option_id":"<scallion>",
    "options":[{"group_id":"<egg>","option_id":"<fried egg>"}]}]}]
```

- An option marked `needs_own_options: true` nests too deep to configure here.
  Tell the eater and offer `picnic_get_checkout_link` to finish in Picnic.
- Map the eater's words to real `option_id`s. If what they want isn't an option,
  say so — don't drop it silently or invent an ID. Removals ("no sauce") are
  usually options; look for them before reaching for a note.
- When the eater left choices open ("random", "anything"), pick available
  options and tell them what you picked.

## Notes

`note` (special instructions, up to 200 characters) only works on items whose
`accepts_note` is true in `picnic_get_item`; otherwise use an option or tell the
eater the restaurant doesn't take notes on that item.

## Quantities

- Adding an item that's already in the cart with the same options and note
  raises that line's quantity; different options make a separate line.
- `picnic_cart_update_item  line_id=…  quantity=N` sets a line's quantity
  directly (0 removes it).
- At most 20 per line.

## Removing and clearing

- `picnic_cart_remove_item  store_id=…  item_id=…  [line_id=…]` — without
  `line_id` it removes every line for that item.
- To clear the cart, remove each line from `picnic_cart_get`.

## Guardrails

- **"Put it in my cart" means stop at the cart.** Report the line and price; don't
  preview or place unless asked.
- **Prices change.** `picnic_cart_get` reprices the cart; quote it, not an earlier
  search result.
