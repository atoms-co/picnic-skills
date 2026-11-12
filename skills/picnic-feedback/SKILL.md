---
name: picnic-feedback
description: >-
  Send feedback to the Picnic team with picnic_feedback, and keep the eater's
  consent to it — ask each time by default, "always send" once they've agreed
  to it, "never" when they've opted out, switchable at any time. Use whenever
  you're about to call picnic_feedback, after an order attempt, when a tool is
  missing or a result fell short, or when the eater says "tell the Picnic
  team…" / "stop sending feedback" / "always send feedback".
---

# Feedback and consent

Feedback goes to the Picnic team under the eater's name and email, so sending it
is the eater's call. The server keeps no record of their answer: **you hold the
eater's preference**, in the conversation and in your harness's memory if it has
one.

## When to send

```
picnic_feedback  message=…  [kind=…]  [tool_name=…]  [eater_prompt=…]  [eater_consent=…]
```

`eater_consent` is how the eater's answer travels: leave it out to let their
client ask them, `given` when they said yes to this one in chat, `always_send`
when they've chosen to always send. See "Consent" below.

- The eater asks to pass something on ("tell the Picnic team the checkout link
  opened the wrong window") — `kind: eater_request`.
- A tool the eater needed doesn't exist (edit a placed order, change a hub) —
  `kind: missing_tool`.
- A result was incomplete or you had to approximate what the eater wanted —
  `incomplete_result` / `approximation`.
- You tried to place an order, whether it was placed or not — see
  `picnic-checkout`, step 5.

Say what was needed and what happened; include the eater's request verbatim in
`eater_prompt` when it helps. Leave out payment details and anything personal
beyond what the request itself says. Combine related points into one submission.
There's no reply channel — tell the eater it was sent, not that someone will get
back to them.

## Consent: ask, always, never

The eater is in one of three states. **Ask each time is the default**, and they
can move to either of the others, or back, whenever they like.

| State | What you do before calling `picnic_feedback` |
|---|---|
| **Ask each time** (default) | If `picnic_get_profile` reported `prompts_in_client: true`, call with no `eater_consent`: the eater accepts or declines in their client. If it reported `false`, ask in chat first and pass `given` (see below). |
| **Always send** | Call with `eater_consent: always_send`: nobody is asked, in chat or in the client. |
| **Never send** | Don't call the tool at all. |

## How to ask

**Check `prompts_in_client` first.** Use the value returned by
`picnic_get_profile` to determine whether the client can ask for feedback consent.
Read it once
per conversation, before the first `picnic_feedback` call.

**Don't ask twice.** In "ask each time", when `prompts_in_client` is `true`,
call `picnic_feedback` with no `eater_consent`: that prompt is the ask — one
field with send / send and always / don't send / don't send and never, submitted
with Accept — and you must not also ask in chat.
Tell the eater in a line what you're sending, then call.

When `prompts_in_client` is `false`, ask in chat before sending feedback;
do not rely on a client prompt. Ask in this shape, and call with
`eater_consent: given` on a yes:

```
I'd like to send this to the Picnic team (it goes under your name and email):

> <the message, verbatim, as you'd pass it to picnic_feedback>

Send it? (yes / no)

You can also change how I handle feedback from now on:
- keep asking each time (current default)
- always send without asking
- never send feedback
```

Keep the three options in the note every time you ask, so the eater always knows
they can change the setting. Read the reply as follows:

| Eater says | Do |
|---|---|
| yes / send it | Call with `eater_consent: given`; stay in "ask each time". |
| no / don't | Drop this one; stay in "ask each time". |
| always / don't ask again / always send | Call with `eater_consent: always_send`; move to "always send". |
| never / stop sending feedback | Drop this one; move to "never send". |
| keep asking / ask each time | Treat as the current state; ask about this one if they didn't answer it. |

A plain "yes" consents to this submission only; only the explicit words move
the eater to "always". Don't send without an answer: no reply is not a yes.
Never pass `given` or `always_send` on your own judgement.

**Entering "always".** Only on the eater's explicit say-so, in reply to the
prompt above, in the client's prompt, or unprompted ("always send feedback",
"you don't need to ask"). From then on pass `eater_consent: always_send` on
every call and don't show the prompt again; the server trusts you for it and
skips its own prompt too. Never pass it on your own judgement.

**Entering "never".** On "stop sending feedback", "never send feedback", or a
no that's clearly general rather than about this one submission. From then on
don't call `picnic_feedback`, including the post-order feedback in
`picnic-checkout`; a later "tell the Picnic team…" is a one-off request, send
that one and stay in "never" unless they say otherwise.

**Switching.** Any state can change to any other on the eater's word: "ask me
each time again", "actually, always send it", "stop". The latest instruction
wins. Confirm the change in a few words so they know what will happen next time.

**Remembering.** Keep the state for the whole session. If your harness has a
persistent memory (Claude Code memory, an `AGENTS.md`, a notes file the eater
has approved), record "always" or "never" there with the date, so the eater
isn't asked to repeat themselves next session — and update it when they switch.
Never assume a preference you have no record of: with no record, they're in
"ask each time".

## The client's own prompt

When `prompts_in_client` is `true`, the Picnic connector shows the eater each
submission in the client
before it goes out, with the same choices in one field: send (default), send
and always send, don't send, don't send and never send. Accept submits the
choice; Decline only skips this one (a decline carries no form content, so a
standing choice is only ever made with Accept). The server stores none of it;
the result's `consent` tells you what the eater chose:

| `consent` | Meaning | Do |
|---|---|---|
| `{"source":"client_prompt","action":"accept","choice":"send"}` | Sent this one | Stay as you were. |
| `…"choice":"dont_send"` | Not sent | Don't retry it. Their state doesn't change. |
| `…"action":"decline"` (or `cancel`) | Not sent | Don't retry it. Their state doesn't change. |
| `…"choice":"send_and_always_send","from_now_on":"always_send"` | Sent, and they chose always | Move to "always send": from the next call on, pass `eater_consent: always_send` and ask nobody. |
| `…"choice":"dont_send_and_never_send","from_now_on":"never_send"` | Not sent, and they chose never | Move to "never send": don't call `picnic_feedback` again. |
| `{"source":"agent"}` after `eater_consent: given` | Sent on the eater's yes in chat | Stay as you were. |
| `{"source":"agent"}` with no `eater_consent` passed | No prompt could be shown and it went out on your word — this means `prompts_in_client` was `false` and you didn't check it | Ask in chat before every further submission for this eater. |
| `{"source":"agent","from_now_on":"always_send"}` | You passed `eater_consent: always_send` | Sent on the eater's standing choice; nobody was asked. |

A choice made in the client prompt is the eater's instruction, exactly as if
they'd said it to you: record it and act on it.

## When sending fails

`picnic_feedback` can fail after the eater has answered: Picnic's rate limit
(`UNAVAILABLE`, "has sent 5 feedback messages") or a delivery problem
(`UNAVAILABLE`, "couldn't be delivered"). The error's `details.consent` carries
the eater's answer and any "from now on" choice exactly as a success would:
read it and record it the same way. A failed send never undoes a choice the
eater made.

Who hears about the failure depends on whose feedback it was:

- **The eater's own feedback** (`kind: eater_request`, something they asked you
  to pass on): tell them it couldn't be sent, and offer to try again later. They
  wanted it delivered, so they should know it wasn't.
- **Feedback you raised** (a missing tool, an incomplete result, an
  approximation, the post-order report): don't mention the failure to the
  eater. It's between you and the Picnic team, and a note about undelivered
  internal feedback only adds noise to their conversation. Retry once later in
  the session if the error was retryable; otherwise drop it.
