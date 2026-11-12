---
name: picnic-connection-help
description: >-
  Diagnose problems using the Picnic MCP server — missing tools, sign-in
  errors, unclear failures — and point the eater to the Picnic website for
  anything the tools can't do. Use when a Picnic tool errors, the tool list is
  empty or stale, or the eater asks for something no tool does. For sending
  feedback to the Picnic team, see picnic-feedback.
model: opus
---

# Connection help

## Calling Picnic tools

Use the Picnic tools exposed by the connected client. Tool names may include a connector prefix; use the exact names provided by the client. If an expected tool is missing, refresh or reconnect the Picnic connector.

## Symptoms

| Symptom | Likely cause | Fix |
|---|---|---|
| Every call returns 401, or "needs you to sign in again" | Sign-in needs refreshing | Reconnect the Picnic connector in your client settings and sign in again. If that fails, disconnect it and add it again. |
| Tool list is empty after a successful sign-in | The connection may need refreshing, or Picnic MCP access may not be available for the account | Reconnect; if no tools appear, contact Picnic support. Meanwhile the eater can still order in the Picnic app or at order.trypicnic.com. |
| `PERMISSION_DENIED` saying the server "is not enabled for this eater" (`details.reason: mcp_disabled`) | Picnic MCP access is unavailable for this account | Contact Picnic support and pass on the message's pointer: the eater can still order in the Picnic app or at order.trypicnic.com. |
| A tool you expect is missing | A stale tool list | Reconnect |
| `FAILED_PRECONDITION` saying the account has no hub | No default hub on the Picnic account | The eater picks one at order.trypicnic.com |
| `UNAVAILABLE` / "temporarily unavailable" | A backend is restarting or busy | Retry shortly; don't retry in a tight loop |
| `RESOURCE_EXHAUSTED` saying a tool was called too often | Picnic's per-eater rate limit for that tool (`details.window`, `details.limit`) | Stop calling that tool; wait `details.retry_after_seconds` when given, else until the window passes. Tell the eater, and don't retry in a loop or spread the calls across other tools |
| `FAILED_PRECONDITION` for anything else (an order refused, a cart Picnic can't price, an empty or moved cart) | Picnic needs something settled first; the message says what | Tell the eater what the tool said, and give them the website link too (see below) |
| `INTERNAL` | Something broke on Picnic's side | Retry once; if it persists, report it with `picnic_feedback` and give the eater the website link (see below) |
| `NOT_FOUND` naming valid window ids | A stale or made-up `window_id` | Use one of the ids listed, or call `picnic_get_delivery_windows` again |

Report errors as what the tool said. Don't guess at causes you can't see, and
don't paper over a failure by answering from memory.

**Give the website link on unexpected and precondition errors.** When a tool
returns `INTERNAL`, or `FAILED_PRECONDITION` for anything other than a missing
hub, the eater shouldn't be stuck with you: alongside what the tool said, hand
them the Picnic website link from `picnic_get_checkout_link`, fetched as
described in "When no tool does what the eater asked" below, so they can finish
in Picnic themselves. The one exception is the no-hub precondition, since the
link needs a hub: send them to order.trypicnic.com to pick one.

## When no tool does what the eater asked

The tools cover browsing, the cart, ordering, order history, cancelling a placed
order (`picnic-my-orders`) and discounts. For
anything else the eater asks for — editing a placed order,
changing their hub, payment methods or notification preferences, favourites,
rotations, creating or managing a team order, support — say plainly that it
isn't something you can do here, and send them to the Picnic website to do it
themselves. (What a team order or stipend covers is readable: see
`picnic-checkout`.)

Get a delivery window first, then ask for the link with it: a bare
`picnic_get_checkout_link` call uses the cart's window, which needs items in
the cart and the `picnic:order` scope, and these requests usually come with an
empty cart.

```
picnic_get_delivery_windows  delivery_date=<today, or the next day with windows>
picnic_get_checkout_link     delivery_date=…  window_id=<one of them>
```

Give the eater the `url` it returns, as returned. It opens Picnic at their hub;
they may need to sign in there, to the same account. If what they want depends
on the delivery window, pass on the tool's note that Picnic may show a
different window than the one asked for. If no upcoming window is available,
say you can't produce a link right now rather than guessing one. Don't invent
a different Picnic URL or describe where the feature lives on the site; just
hand over the link.

Then, with the eater's consent as in `picnic-feedback`, report the gap with
`picnic_feedback` (`kind: missing_tool`) so the team hears about it, leaving
out personal or payment details the request may have contained.

Use `picnic_feedback` to report a missing tool, a result that fell short, or
something the eater asked you to pass on. How to write it, and how the eater's
consent works — their client's prompt asks by default, "always" or "never" on
their say-so — is in `picnic-feedback`.
