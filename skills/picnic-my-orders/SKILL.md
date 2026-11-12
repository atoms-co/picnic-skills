---
name: picnic-my-orders
description: >-
  Look up the eater's Picnic orders through the Picnic MCP server — recent and
  upcoming orders, their statuses, receipts, cutoffs, and reordering something
  from before — and cancel a placed order. Use for "what did I order?", "did
  yesterday's lunch go out?", "do I have anything for Monday?", "send me the
  receipt", "order the same curry again", "show me something I ordered
  before", "can I still cancel it?", "cancel my lunch order". For a photo or
  picture of a past order, use this with `picnic-photos`.
model: opus
---

# My orders

```
picnic_get_order_history
```

It returns the eater's 25 most recent orders, newest delivery first: placed
orders **and** upcoming scheduled or rotation orders that haven't been placed
yet. Each has its delivery window, `placed_at` (absent until placed), `status`,
items with options, and `items_total`. An order that was part of a team order
also has `team_order` (its ID and name); `picnic_get_team_order` reads it
(`picnic-checkout`). If the note says Picnic couldn't tell for some orders, a
missing `team_order` isn't a no.

## Always read it fresh

Answer from the tool, never from earlier in the conversation or from what you
placed yourself: the eater may have ordered or cancelled in the app, and
statuses move on their own. If the call fails, say so — don't reconstruct the
history from memory.

## Statuses

`scheduled` and `rotation_scheduled` (upcoming), `pending`, `preparing`,
`prepared`, `out_for_delivery`, `delivered`, `paused`, and the canceled states
(`canceled_by_user`, `canceled_by_kitchen`, `canceled`). Put upcoming orders and
anything in flight first; a long tail of canceled orders can be summarized.

## Totals

`items_total` excludes tax, fees and discounts, so it's below what was charged —
say "items total", not "you paid".

## Receipts

Placed orders carry `receipt_url`. Give it exactly as returned — don't build one
from an ID. Upcoming orders that haven't been placed have no receipt yet. The
link opens in Picnic for the eater signed in with the same account.

## "What do I have coming up?"

Combine the history's upcoming orders with `picnic_get_delivery_windows` for the
days in question: show what's scheduled and, for empty days, the window and its
`order_by` cutoff so the eater knows how long they have to order.

## Reordering

"The same curry again", "my usual":

1. Find the past order in the history; take each item's `store_id`, `item_id`
   and options.
2. Look each item up with `picnic_get_item` for the **new** window — it may be
   unavailable, priced differently, or its options may have changed.
3. Add it with `picnic_cart_add_items`, then follow `picnic-checkout`.

Never reuse a past order's prices.

## Seeing what they ordered

"Let me see what I ordered", "show me a picture of Tuesday's lunch": the history
carries no photos. Take the item's `store_id` and `item_id` from the history row
and follow `picnic-photos` — in the Claude desktop and web apps that means
`picnic_get_item` for the next window with menus, which draws the dish card
inline; elsewhere it means `picnic_get_item_photo` for that item when the
client shows images, or its `image_url` as a link from a terminal. Don't call
`picnic_get_item_photo` in the Claude apps, and don't paste image links
expecting them to render.

## Cancelling an order

```
picnic_cancel_order  order_id=<the order's order_id from the history>
```

**Can it still be cancelled?** Each placed order in the history has
`cancellable`: `true` while Picnic would still accept a cancellation, `false`
once a restaurant has started on it, the hub's cancel cutoff has passed, or
the order is already cancelled.
When it's absent, Picnic didn't say; the cancel call itself will answer. Read
the history fresh before answering "can I cancel?" — it changes as the cutoff
approaches.

**Cancelling is the eater's decision, like placing.**

1. Read the history and find the order they mean. If more than one fits ("my
   lunch order" with two upcoming), ask which.
2. Tell them which order it is — items and delivery window — and get an
   explicit yes to cancelling that one. "Cancel my orders" is not a yes to each
   order; go through them.
3. Call `picnic_cancel_order`. Where the eater's client can show prompts, Picnic
   asks them to confirm there too; if they decline, nothing is cancelled and you
   don't retry.
4. On success Picnic has accepted the cancellation, and the status follows
   shortly. Read the history again and report the status it shows.

- A multi-restaurant order is cancelled whole; say so before they agree.
- Picnic handles any refund. Don't promise an amount or a date.
- A refusal says why (`details.reason`): past the cutoff, already cancelled or
  delivered, or an earlier cancellation that failed. Nothing was cancelled.
  Tell the eater what Picnic said; for an order past its cutoff, Picnic support
  on the website is the way (`picnic-connection-help` has the link rule).
- Only placed orders can be cancelled here. A scheduled or rotation entry that
  hasn't been placed yet is changed on the Picnic website.
- **Editing a placed order isn't available here.** Cancel and reorder only if
  the eater asks for exactly that.
