# HeyQo API — what the live API actually does

Established against the running API, sandbox and production. Where this
contradicts their documentation, this is what the API did.

Base URLs — the key's mode must match the URL, or you get a 403 with nothing in
the body that says so:

```
sandbox  https://heyqo.cash/business/sandbox/v1
live     https://heyqo.cash/business/v1
```

---

## 1. Auth — `POST /authentication/token`

```json
{ "client_id": "...", "secret_id": "..." }
```

**`secret_id`, not `client_secret`.** The token is at `data.access_token`, with
`data.expires_in` in seconds. Cache it and renew a minute early.

A **403** here almost always means a production key on the sandbox URL or the
reverse. Say so in the error; nothing else will.

---

## 2. The envelope

Every response:

```json
{ "message": { "code": 200, "error": [], "success": [] }, "data": {}, "type": "success" }
```

- The payload is under `data`, never at the root.
- Errors are in `message.error[]` and `type: "error"` — **a 200 can still be an
  error**, so check both.
- Their messages are the useful part: "The selected brand is invalid",
  "Customer with this external_ref already exists". Keep them.

---

## 3. Cardholders — `POST /customers`

Takes the identity evidence: names, date of birth, document type and number,
document images as base64, address, and the AML declarations.

- **409** `Customer with this external_ref already exists` on a repeat. Adopt the
  existing one rather than failing — otherwise that person can never be issued a
  card again.
- **`?external_ref=` is accepted and ignored** on the listing. Asking for a
  reference that does not exist returns *every* customer. Match client-side or
  you will hand somebody else's cardholder to a card.

### Which id to use

This one is expensive to get wrong. The listing returns:

```json
{ "id": "...", "local_id": 14, "external_ref": "...", "status": "ACTIVE", ... }
```

**Card creation wants `local_id`.** Passing the listing's `id` returns
`customer was not found` from `POST /cards` — for a customer that plainly
exists, matched correctly, with status `ACTIVE`. `GET /customers/{id}` answers
400 "something went wrong, please contact support", so the detail record is no
help either.

### Environment

A cardholder registered in sandbox does not exist in production. Store the
environment alongside the id and key your record on both, or switching the base
URL will hand production a sandbox id forever.

---

## 4. Cards — `POST /cards`

```json
{ "customer_id": "<local_id>", "currency": "usd", "brand": "visa", "label": "..." }
```

`brand` must be **lowercase**; `"Visa"` is a 422.

The response:

```json
{
  "id": "...", "local_id": 14, "customer_id": "...",
  "fee_plan": {...}, "fees": {...},
  "card": { "customer": ..., "card_id": "...", "card_status": "...",
            "card_brand": "...", "card_currency": "...",
            "amount_debited": "...", "display": {...} }
}
```

**No number and no expiry.** The card is still being provisioned — their
dashboard shows "processing". Write the row as pending, send no welcome email
quoting digits, and fill it in from `GET /cards/{id}` afterwards.

`amount_debited` is what they charged you for this card. Worth recording: it is
the real cost, and it is not always what you were quoted.

---

## 5. Reading a card — `GET /cards/{id}`

```json
{ "card": {
    "id": "...", "amount": "15.9", "currency": "usd", "status": "active",
    "title": "General", "notes": ..., "created_at": "...",
    "display": { "mode": "...", "hint": "..." },
    "info": {
      "masked_pan": "411111******1111", "last4": "1111",
      "expiry_month": "08", "expiry_year": "29",
      "name_on_card": "...", "brand": "visa", "type": "virtual",
      "address": "1007 N Orange St. 4th Floor", "city": "Wilmington",
      "state": "Delaware", "zip_code": "19801"
    } } }
```

Three things to take from this shape:

- **The balance is `amount`**, a string in units. There is no `balance` field.
  Reading the name you expected leaves every card frozen at whatever you last
  wrote yourself, and spending invisible.
- **The card's details are under `info`**, not at the top level and not in
  `display`. `display` is about how their hosted page renders.
- **The expiry is two digits, as strings.** `"08"` and `"29"`. A parser
  insisting on four digits silently returns nothing.

There is **no transaction list**. The balance is the record: pull it, compare it
with yours, and record the difference as one movement. Two purchases between two
pulls arrive as one line — say so rather than inventing per-merchant detail.

---

## 6. Money — deposit and withdraw

```
POST /cards/{id}/deposit    { "amount": 36 }
POST /cards/{id}/withdraw   { "amount": 10 }
```

Amounts are in **units, not cents**. Their fee is charged **on top** of the
amount, out of your merchant float: with a flat deposit fee of `f`, depositing
`n` costs you `n + f`. Withdrawing carries a flat fee too, so money pulled back
off a card is not free.

A **402** means your float is empty:

```
Insufficient merchant balance. Required X.XX USD (X.XX USD). Current balance: 0.00 USD.
```

That is your balance, not the cardholder's. Surface a service-unavailable
message and keep their wording in the log.

---

## 7. Card state

```
PUT /cards/{id}/freeze
PUT /cards/{id}/unfreeze
PUT /cards/{id}/terminate
```

`terminate` answers **404 "Card not found"** for a card that belongs to another
issuer — which is what happens if you route every card through whatever provider
is configured today rather than the one that issued it.

---

## 8. Webhooks

Signed with **HMAC-SHA256 over the raw body**, hex, in `X-HeyQo-Signature`.
Compute over the bytes as sent; re-serialising the parsed object will not match.

Events seen in production:

| Event | What to do with it |
|---|---|
| `customer.approved` / `customer.rejected` | the issuing bank's verdict on a cardholder |
| `card.charged` / `card.funded` | money moved — **re-read the balance**, see below |
| `card.declined` | nothing moved, so nothing to reconcile |
| `card.terminated` | the card is closed |

**Do not add up the amount on a spend event.** There is no event id and no
timestamp, so deliveries cannot be deduplicated or ordered, and arithmetic on
those terms drifts. Read `GET /cards/{id}` and take the balance it reports; the
issuer's figure is the record and a duplicate delivery then costs nothing.

- **No timestamp and no event id**, so there is no replay protection and nothing
  to deduplicate on. Make applying the same verdict twice a no-op by
  construction, comparing state rather than remembering deliveries.
- One webhook configuration per account, with no sandbox/live switch. Confirm
  whether the same signing secret covers both.
- Fail closed on a bad signature, and log the header *names* you received — a
  mismatch between the header they send and the one you read will otherwise
  401 every event in silence.
