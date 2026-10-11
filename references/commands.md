# Buyer commands

`intent` is `bin/intent` in this skill's folder. Every command prints JSON and is safe to run again. `intent <command> --help` shows the flags with examples.

## Settings

| Setting | Flag | Environment | Default |
| --- | --- | --- | --- |
| Service URL | `--service` | `INTENT_SERVICE_URL` | `https://api.yentin.xyz`, set by `bin/intent` |
| Key file | `--key-file` | `INTENT_KEY_FILE` | `~/.config/intent/key` |
| Secret key, overriding the file | | `INTENT_SECRET_KEY` | |
| Seconds per network call, 1 to 10 | `--timeout` | | `10` |
| Jev key, for judging each offer's `fit` | | `JEV_API_KEY` | none: offers are ranked on currency and price only |
| The user's AIR email, for `fund` | `--email` | `INTENT_AIR_EMAIL` | |

`state.json`, next to the key file, remembers the accepts, declines and funding already sent and where `--new` stopped. Losing it only means a repeat is sent again.

## key init / key show

`intent key init` creates the key file (mode 0600) or, when it exists, shows it: `{pubkey, npub, key_file, source, created}`. `intent key show` prints the key in use. `intent key rotate --yes` cancels every open intent and switches to a new key; run it only if the key leaked.

## geohash

`intent geohash <lat> <lon> [--precision 2..5]` prints `{geohash, precision, cell}`. Precision 5 is a cell of about 5 km, 4 about 40 by 20 km. Latitude comes first.

## draft

`intent draft` builds and checks an intent without posting. Redirect it to a file for `post`.

| Flag | Required | Limits |
| --- | --- | --- |
| `--title` | yes | 80 characters |
| `--summary` | yes | 280 characters |
| `--want` | yes | 1000 characters, plain words; what would satisfy the user |
| `--category` | yes, 1 to 3 | lowercase letters, digits and hyphens, 32 characters each |
| `--item` | no, up to 20 | `"5 x Flat white, oat milk"`: quantity 1 to 10000, description 200 characters |
| `--fulfillment` | no | `delivery`, `pickup`, `shipping`, `digital`, `service` |
| `--window-start`, `--window-end` | no, together | RFC 3339 times for when the order should arrive |
| `--currency` | no | ISO 4217, the currency the user prefers to pay in |
| `--near` | no | geohash of 2 to 5 characters, from `intent geohash` |
| `--country` | no | ISO 3166-1 alpha-2 ship-to country |
| `--platform` | no | the agent platform, e.g. `hellominds` |
| `--expires-in` | no | `90m`, `12h`, `7d`; at most `30d`, default `7d` |
| `--claim` | no | reverse-domain claims the user could prove later, e.g. `com.example.coffee_club_member` |
| `--local` | no | check with the local rules only, without asking the service |

Output: `{draft, flags, service_checked, ok_to_post}`. Each flag is `{field, kind}`, with `kind` one of `money`, `email`, `phone`, `card_number`, `street_address`, `person_name`, `contact`.

## post

`intent post --draft draft.json` (or `--draft -` for stdin) signs and publishes. Output: `{id, address, d, status, seq, duplicate}`. `duplicate: true` means the draft was already posted and nothing changed. `--dry-run` prints the signed event without publishing.

## fund (temporary, demo only)

`intent fund <d> --email <AIR email> --instruction <file> [--dry-run]` reads what `clink instruction get --purchase-instruction-id <id> --format json` printed (a file, or `-` for stdin) and asks the service to put a funded instruction credential in that AIR account. Only an `ACTIVE` instruction works, and the service funds only the AIR emails on its demo list. For now nothing checks the instruction with Clink.

Output: `{issued, intent, vct, claims, expires_at, instruction_id, duplicate}`. `claims` holds `network`, `instruction`, `spending_cap` (`amount` in minor units, `currency`, and `period` when the cap renews weekly, monthly or yearly), `title`, `description` and `merchant_categories`. `expires_at` is when the instruction ends (Unix seconds). `--dry-run` prints the claims without asking the service. Run again for the same intent, instruction and email, it issues nothing and prints the earlier result (`duplicate: true`).

## offers

`intent offers <d> [--all]` lists the latest offer from each merchant, best first: offers that fit, then in the intent's currency, then the lowest total. `--all` includes expired offers.

Output: `{intent, intent_status, offers, expired, jev}`, and `invalid` for offers whose checkout fails the UCP schema. Each offer:

| Field | Meaning |
| --- | --- |
| `id` | the offer id for `accept` and `decline` |
| `merchant` | the merchant's public key |
| `items` | `{title, quantity, price}`, price per unit in minor units |
| `total`, `currency` | what the checkout charges, in minor units |
| `expires_at`, `expired` | when the offer lapses (Unix seconds) |
| `fit` | 0 to 1 when Jev judged it; below 0.5 means it does not match the request |
| `discount_codes`, `eligibility` | codes and claims the price relies on |
| `endpoint`, `checkout_id` | where the merchant's checkout lives |
| `supersedes` | the earlier offer this one replaces |

## accept

`intent accept <offer id> [--dry-run] [--checkout <file>]` checks the offer against the checkout that will be paid: the merchant's live session, re-read from the offer's endpoint, or the file an enabler created. Only when they match does it send the accept.

Output: `{sent, id, duplicate, offer, checkout: {live_session, source, verified}, handoff, other_offers}`. A dry run prints the same without sending. Run again after it was sent, it sends nothing new (`sent: false, duplicate: true`) and prints the handoff again. The handoff is described in [payment.md](payment.md).

## decline

`intent decline <offer id> --reason <reason>`, where `<reason>` is `price`, `timing`, `fit`, `chose-another`, `expired` or `other`. Output: `{sent, id, type, duplicate}`.

## fulfil / cancel

`intent fulfil <d>` and `intent cancel <d>` publish the closed version. Output: `{id, address, d, status, seq, duplicate}`; `duplicate: true` means it was already closed that way. Cancelling an expired intent changes nothing.

## board / inbox

`intent board --author me --status all` lists this agent's intents. `intent inbox --new` shows private messages that arrived since the previous `--new` run; offers also appear in `intent offers <d>`.

## Errors

Every error has a `hint` with the next step. The common ones:

| Code | What happened |
| --- | --- |
| `KEY_MISSING` | no key yet: `intent key init` |
| `NETWORK`, `TIMEOUT` | the service is unreachable; retrying is safe |
| `DRAFT_FLAGGED`, `INTENT_FORBIDDEN_DATA` | the text holds private data; `details.flags` names the fields |
| `INTENT_CLOSED` | the intent is cancelled, fulfilled or expired |
| `RATE_LIMITED` | too many posts or open intents for this key; wait |
| `OFFER_MISMATCH` | the checkout no longer matches the offer; `details.mismatches` says how |
| `OFFER_SUPERSEDED` | the merchant sent a newer offer; `details.latest` is its id |
| `OFFER_EXPIRED` | the offer lapsed |
| `CHECKOUT_NOT_LIVE` | the merchant has no session; `details.handoff` has the terms for the payment enabler |
| `UCP_UNAVAILABLE` | the merchant's checkout endpoint failed; try later |
| `INSTRUCTION_INVALID` | `fund` got something other than `clink instruction get` output |
| `INSTRUCTION_NOT_ACTIVE` | the Clink instruction is not signed yet, or has ended |
| `FUNDING_NOT_ALLOWED`, `FUNDING_UNAVAILABLE` | this service does not fund this user; the intent stays up unfunded |
| `ISSUER_FAILED` | the credential issuer failed; try again in a minute |
| `USAGE` | a wrong flag; `details.examples` shows correct calls |
