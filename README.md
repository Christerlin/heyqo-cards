# heyqo-cards

A [Claude Code](https://claude.com/claude-code) skill for integrating **HeyQo**
virtual Visa cards.

Everything here was established against the live API. Their documentation does
not describe most of it, and several field names are not what you would guess:
the balance is `amount`, the card's details are nested under `info`, and card
creation wants a different customer id than the one their listing puts first.

Written while taking a Haitian wallet from sandbox to production, so it covers
more than response shapes: who owns the cardholder KYC, what a flat per-load fee
does to pricing, and the rules that keep a ledger and an issuer from drifting
apart.

## Install

```bash
git clone https://github.com/Christerlin/heyqo-cards ~/.claude/skills/heyqo-cards
```

Claude Code picks it up on the next session.

## What's in it

| | |
|---|---|
| `SKILL.md` | The mental model, the cost structure, the money-safety rules |
| `references/api.md` | Endpoint by endpoint, field by field |
| `references/secure-view.md` | The hosted page that shows the full card number |

## Using it outside Claude Code

The files are plain Markdown with no tooling in them, so nothing here is locked
to one agent. Only the automatic triggering is: the `name` and `description` in
`SKILL.md`'s frontmatter are what tells a Claude agent when to reach for this,
and another tool will not read them.

Everywhere else, point at it or paste it:

- **Claude Code, the Claude Agent SDK, claude.ai** — drop it in the skills
  directory and it loads itself when the work touches HeyQo.
- **Cursor, Copilot, Windsurf, anything else** — `AGENTS.md` in this repo is
  read by several of them directly. Otherwise reference `SKILL.md` and the file
  under `references/` that matches what you are doing, or copy the relevant
  section into whatever instruction file your tool uses.
- **A person** — it is written to be read. Start with `SKILL.md`; the references
  are for when you are actually writing the request.

## A note on figures

Provider rates are deliberately left as placeholders. They differ per account
and are not ours to publish — the reasoning and the formulas are here, the
numbers come from your own contract.

## Contributing

If the API changes or you hit a quirk that is not here, open a PR. Real, tested
behaviour beats the official docs; when they disagree, document what the live
API did and note the date.

MIT.
