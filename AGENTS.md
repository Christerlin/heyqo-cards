# HeyQo virtual cards

Guidance for any coding agent working against the HeyQo card API.

Read **[SKILL.md](SKILL.md)** first: it carries the mental model, the cost
structure that decides the product, and the money-safety rules. Then whichever
reference matches the task:

- **[references/api.md](references/api.md)** — auth, cardholders, cards, money,
  webhooks. Endpoint by endpoint, field by field.
- **[references/secure-view.md](references/secure-view.md)** — the hosted page
  that shows the full card number, and how to place its fields over your own
  card art.

Read the relevant one **before** writing code against this API. Almost none of
the response shapes are guessable, and the failures are silent: a field read
under the wrong name leaves a card frozen at a number nobody chose, with no
error anywhere.

## The short version

- Responses are wrapped in `{ message, data, type }`. **A 200 can still be an
  error** — check `type`. The reason is in `message.error[]`, never the status
  line.
- Nothing is named what you expect. The balance is `amount`. The card's details
  are under `info`. The expiry is two digits, as strings.
- A card is created **before it exists**: the response carries no number and no
  expiry. Write the row as pending, never as a placeholder.
- Sandbox and production are separate worlds. A cardholder in one does not exist
  in the other.
- An empty float is **your** money, not the cardholder's. Never pass its wording
  through to a user; it states your balance. It arrives as a 402 with figures
  **or** a bare 400 `insufficient fund`, so match on both, and keep it apart from
  `please wait, try again later`, which is a throttle and means the opposite.
- One float covers every brand and every operation, so a run of failures on one
  card network is almost certainly the hour rather than the network.
- The balance is not what the card can spend. They keep a minimum inside it, and
  authorisation holds do not move it; the held figure appears only in the text of
  a refusal. Show `amount - pending - minimum` or the holder pays a fee per
  refused attempt.
- The PAN must never cross your servers. It renders in their hosted page, and
  that is what keeps the integration in PCI SAQ A rather than SAQ D.

## When exploring a response

Log **field names, never values**. These payloads carry a masked PAN and an
address, and a card number in a log file is the one thing this whole design
exists to prevent.
