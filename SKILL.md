---
name: heyqo-cards
description: >-
  Issue and operate virtual Visa cards through HeyQo, a white-label layer over
  Platnova. Use this whenever the work touches HeyQo: creating a cardholder,
  issuing a card, loading or withdrawing from one, showing the full card number,
  reconciling a balance, or debugging a card error like "customer was not
  found", "insufficient fund", a 402, a card stuck at 0000, a refusal on a card
  that visibly has the money, a fee that arrived with no event behind it, or a
  secure-view page that will not lay out. Also use it when choosing or pricing a
  virtual-card provider for Haiti or the Caribbean, because the cost structure
  here decides whether the product works. Everything in it was established
  against the live API; their documentation does not describe most of it.
---

# HeyQo virtual cards

HeyQo issues virtual Visa cards. It is a **white-label layer over Platnova**, and
the issuing bank sits at the far end of that chain, which decides more than it
first appears (see "Who owns the cardholder" below).

→ API quirks, field by field: [references/api.md](references/api.md)
→ The secure-view page and how to place its fields: [references/secure-view.md](references/secure-view.md)

Read the relevant reference **before** writing code against this API. Almost
nothing about the response shapes is guessable, and the failures are silent: a
field read under the wrong name leaves a card frozen at a number nobody chose.

---

## The things that cost the most time

**1. Their responses are wrapped, and the useful part is in the body.**
Every response is `{ message, data, type }`. Errors carry their reason in
`message.error[]`, not in the status line. Throw away the body and you are left
debugging "failed: 400" against an API that names the problem in plain words.

**2. Nothing is called what you would expect.**
The balance is `amount`. The card's own details: last four, expiry, name,
masked PAN, billing address: are nested under `info`. The expiry is two digits,
as strings: `"08"`, `"29"`. Read `balance`, or insist on a four-digit year, and
you get nothing back and no error.

**3. A card is created before it exists.**
`POST /cards` answers 201 with no number and no expiry while the dashboard shows
it "processing". Write the row as pending and fill it in later; do not invent a
placeholder. A literal `"0000"` reached a customer's welcome email that way.

**4. Sandbox and production are different worlds.**
A cardholder registered in one does not exist in the other, and the id you
stored will answer "customer was not found" for as long as you keep sending it.
Anything stored against this issuer has to be filed under the environment that
produced it.

**5. One Visa card per customer per month.**
`POST /cards` answers 400 with `only 1 visa card allowed in a month`. It is not
documented and there is no counter to read, so the only way to learn the limit
is to be refused. Surface it as the limit it is: a customer told "try again
later" will try again later, and again, for weeks.

**6. A float failure is your money, never the cardholder's, and it does not
always arrive as a 402.**
`Insufficient merchant balance` means your own float at HeyQo is empty, and it
stops cards for everybody at once. Their documented answer is a **402** carrying
the figures. What production actually saw was a bare **400 `insufficient
fund`**, on every call that spends: a load, a deposit and a card creation alike.
Match on both, because the status has already moved once. Log it as an incident,
show a service message, and never pass their wording through: it states your
balance.

**7. They hold one float for every brand and every operation.**
So a failure tells you nothing about the card network, and the correlation you
are about to believe is probably the clock. Four Mastercard loads failed while
every Visa load on record had worked, which looked conclusive and was not: a
card creation succeeded at 02:32 and another failed at 06:08, so what changed
was the hour, not the brand. It cost an afternoon, a withdrawn feature and a
refund to customers who had done nothing wrong. Before blaming the thing that
failed, find the earliest success and the earliest failure and ask whether
anything but time separates them.

**8. A throttle is not a float failure.**
"please wait for few minutes, try again later" is HeyQo slowing you down, seen
after seven deposit attempts on one card inside a day. Recognise it first so it
never reaches the float branch: the two are a few words apart in their API and a
long way apart in meaning, one being your money gone and every customer stopped,
the other one customer asked to wait. Raising the float alarm for a throttle is
how an alarm stops being believed.

---

## Who owns the cardholder

The KYC requirement comes from **the bank that issues the card**, at the end of
the chain. Neither HeyQo nor Platnova can waive it, and there is nobody
downstream to negotiate with.

Two consequences worth internalising before planning anything:

- **Reusing an existing KYC is not on offer.** A bank exercises its own customer
  identification programme; it does not import a third party's verified session.
  Plan to send the evidence, not to avoid sending it.
- **The evidence has to be transmitted, so consent has to be explicit.** ID
  images, a selfie, date of birth, address, and the AML declarations all cross
  to a third party and its bank. Ask before the first transmission, record that
  you asked, and refuse to proceed without it. Once sent there is no recall.

Keep the bundle that carries this evidence **provider-neutral**. Every issuing
bank carries the same obligation, so the same structure serves the next one.

---

## The cost structure decides the product

Get these numbers in writing before designing any pricing, because they are
flat and they dominate:

| | Get in writing | Why it matters |
|---|---|---|
| Card issuance | the fee, **and** any amount pre-loaded onto the card | The two are debited together, and only one of them is a fee |
| Card load (deposit) | a **flat amount**, whatever the size | The single number that decides who can afford this product |
| Card withdrawal | a flat amount again | Pulling money back off a card is not free: price it or do not offer it |
| Monthly | ask; there may be none | If there is none, a monthly fee of your own is margin end to end |
| Funding your float | possibly a % on deposits | Confirm it; it changes every load's true cost |

Their published rates are not the whole story. Ask for each of these explicitly
and confirm the answers against what actually leaves your float: an invoice
that comes back 3% above the amount you asked to fund is telling you something
the rate card did not.

A flat per-load fee is the whole design problem. Charge a percentage and small
loads lose money; charge a flat fee and they are punitive. The fee has to be
**the larger of a floor and a percentage**: the floor covers the flat cost, the
percentage takes over on amounts large enough for it to be the fairer of the
two.

And the arithmetic that follows from it: for a fee to stay under `p` of the
amount while covering a flat cost `c`, the load must exceed `c / p`. A 3 USD
cost under a 10% ceiling means a 30 USD minimum; double the cost and you double
the minimum. Decide it from that, not from what feels tidy.

Watch what a pre-load does to the cost figure you guard against. If they debit
ten to place five on the card, your cost of *providing a card* is five, but ten
left your float: guard against the ten, or a price of six passes a check it
should fail.

**Record the provider's cost next to your price**, and refuse to save a price
below it. Fees collected are not earnings, and a product that loses money on
every use is otherwise invisible in the revenue figures.

---

## Money-safety rules

- **Load compensates on failure.** Debit the wallet, then ask the issuer. If the
  issuer refuses, put it back in the same request and say so: money must never
  sit between two systems.
- **Never refund a card balance you cannot verify.** Cancelling refunds the
  card's balance to the wallet. If your figure is stale and the issuer's is
  lower, the difference is money you just invented. On a closed card the figure
  you want is on the `card.termination.refund` event, which arrives a few
  seconds *before* the cancellation: pay back what they actually returned, and
  keep your own balance only as a floor for the case where that event never
  comes. A card showing $5.17 returned $0.69, and the holder was credited the
  $5.17.
- **The issuer's balance is the record.** There is no transaction list, so a
  balance cannot be explained by adding up movements: it can only be read. They
  do send an event when a card is charged, and the right response to it is to
  re-read the balance rather than to trust the amount on the event.
- **Show what can be spent, not the balance.** They refuse any authorisation that
  would take a card below its minimum, and that minimum is inside the figure
  their API reports. Show `balance - pending - minimum`, with the balance beside
  it so nothing is hidden. A holder shown $6.75 tried $6.20 four times in ten
  seconds, every attempt within the number on screen, every one refused, and
  each refusal charged: $1.58 gone on a purchase that was never going to clear.
- **A refusal is the only place an authorisation hold is visible.** A hold does
  not move the balance they report, so a card holding $26.00 with $17.93 held
  reads as $26.00 and refuses $17.93. The figure exists in one place, the
  narration of the refusal: "No sufficient funds for transaction and minimum
  balance @ $1.00 - Available balance: 8.07". Parse it, store it, prefer the
  lower of the two, and expire it after a day: a hold either settles, and the
  balance drops on its own, or it lapses, and nothing announces which.
- **Price the refusals.** Each one costs a flat fee, so a card with a balance
  just under what somebody keeps trying to spend drains itself a few tens of
  cents at a time with no purchase ever completing. Show the spendable figure to
  prevent it; think hard before freezing the card to prevent it, because a frozen
  card is a worse outcome than a fee the holder can see and understand.
- **File the card under its issuer.** A card outlives the setting that created
  it; route cancel, freeze, load and reveal by the card's own provider, never by
  whatever is configured today.
- **Never store the PAN.** The number is shown in their hosted page and must not
  cross your servers: that is what keeps you out of PCI scope. Log field
  *names* when exploring a response, never values.
