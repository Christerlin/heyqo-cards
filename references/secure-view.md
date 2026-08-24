# The secure-view page

How a cardholder sees their full number without it crossing your servers. This
is what keeps the integration in **PCI SAQ A** rather than SAQ D: HeyQo renders
the number in their own page, you embed it, and no PAN ever reaches your code.

Never add an API that returns the number itself, however convenient. That single
call moves the whole system into a different compliance regime.

---

## Creating one — `POST /cards/{id}/secure-view`

```json
{
  "layout": "free",
  "align": "left",
  "positions": {
    "pan":        { "top": "57%", "left": "6%" },
    "brand":      { "top": "57%", "right": "6%" },
    "cardholder": { "top": "71%", "left": "6%" },
    "expiry":     { "top": "71%", "left": "60%" },
    "cvv":        { "top": "71%", "left": "78%" }
  },
  "text_color": "#FFFFFF",
  "label_color": "#CAFF4C",
  "show_branding": false
}
```

→ `{ "iframe_url": "...", "expires_in": 90 }`

**Rate limited to about 20 calls an hour.** Exploring the options burns through
that fast; the header `x-ratelimit-remaining` tells you where you stand.

---

## What the options do

| Option | Values | Notes |
|---|---|---|
| `layout` | `stack`, `card`, `compact`, `free` | `free` is the only one that lets you place fields |
| `align` | `left`, `center` | Moves the countdown chip **and** the copy buttons together |
| `positions` | per field: `top`/`left`/`right`/`bottom` | CSS percentages |
| `fields_order` | pan, cvv, expiry, cardholder, brand | `brand` is re-added if you omit it |
| `text_color`, `label_color` | any CSS colour | |
| `button_color`, `button_text_color` | | the copy buttons |
| `brand_label` | string | renders an extra label; leave unset |
| `show_branding` | boolean | |

An unrecognised `layout` returns no URL at all rather than an error.

**Not supported, though they look like they should be:** `show_labels`,
`labels`, `font_size`, `show_timer`, positioning the copy buttons. Passing them
is silently ignored. `label_color: "transparent"` falls back to the default
rather than hiding anything.

---

## Drawing it over your own card

The point is that revealing a card changes the digits and nothing else. That
takes more care than it sounds.

**Make the overlay transparent and place their fields where your card already
puts the same ones.** Their labels sit above each value; yours may not, so their
fields need to sit a few points higher than your own block or the bottom row
lands on the card's edge.

**Empty your own number row while it is up, and nothing else.** Hiding the chip,
the balance or the brand as well makes the card rearrange itself the moment it
is tapped, which reads as a different card rather than the same one showing its
number. Move *their* fields out of the way instead.

**Positions are absolute, so a long name has to be planned for.** "CARMEN
ESPANOLA ESPANOLA" runs straight through an expiry placed at mid-width. Give the
name a line, or push the expiry and CVV close to the right edge.

**Their page is not card-shaped.** When it carries a countdown chip it is taller
than a card, and a container at the card's own ratio clips the copy buttons off
the bottom. Either give the overlay a taller box or accept losing them.

**Do not try to restyle their page from outside.** Injecting CSS into the frame
works until they deploy, and then it breaks in ways nobody can see from your
side. Ask them for the option instead.

---

## The lifetime

`expires_in` is 90 seconds, single use. When it lapses the page does not go
blank — it paints its own "this secure card view has expired" panel over your
card. Take the overlay down a second or two early so nobody sees it.

Carry the lifetime back with the URL rather than hardcoding 90 anywhere.
