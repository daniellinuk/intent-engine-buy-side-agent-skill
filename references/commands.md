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

## whoami

`intent whoami` prints the key and the service, with its limits. When this copy is behind the release in `/.well-known/intent`, it also prints `skill_update`: `installed`, `current`, and `command`. Install that release before the next step. When the field is absent, this copy is current.

## key init / key show

`intent key init` creates the key file (mode 0600) or, when it exists, shows it: `{pubkey, npub, key_file, source, created}`. `intent key show` prints the key in use. `intent key rotate --yes` cancels every open intent and switches to a new key; run it only if the key leaked.

## air bind / air credentials

`intent air bind [--new]` connects this agent to the user's AIR account. Do this as soon as the payment skill says the user has signed a payment instruction, before `intent fund`, unless a previous run already printed `status: bound`. The first run makes the agent's own AIR key and prints `{status: "pending", url, expires_at}`. Send the `url` to the user; they log in to AIR there and approve. Later runs check the same link and print `{status: "bound", user_id, key_id, partner_id}` once the user has approved; `error` holds AIR's code if their last try failed. An expired link is replaced on the next run. `--new` makes a new key and link, replacing the earlier binding.

The AIR key stays in `air.json` beside the key file (mode 0600) and is never printed; only its public half leaves the machine.

`intent air credentials [--limit 1..100]` lists the credentials in the user's AIR account: `{user_id, credentials, more}`, each with `schema_id`, `issuer`, `vct` and `created_at`.

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
| `--platform` | no | the name of this agent's runtime |
| `--expires-in` | no | `90m`, `12h`, `7d`; at most `30d`, default `7d` |
| `--claim` | no | reverse-domain claims the user could prove later, e.g. `com.example.coffee_club_member` |
| `--local` | no | check with the local rules only, without asking the service |

Output: `{draft, flags, service_checked, ok_to_post}`. Each flag is `{field, kind}`, with `kind` one of `money`, `email`, `phone`, `card_number`, `street_address`, `person_name`, `contact`.

## post

`intent post --draft draft.json` (or `--draft -` for stdin) signs and publishes. Output: `{id, address, d, status, seq, duplicate}`. `duplicate: true` means the draft was already posted and nothing changed. `--dry-run` prints the signed event without publishing.

## fund (temporary, demo only)

`intent fund <d> --email <AIR email> --instruction <file> [--dry-run]` reads the payment instruction JSON from the payment skill (a file, or `-` for stdin) and asks the service to put a funded instruction credential in that AIR account. Only an `ACTIVE` instruction works, and the service funds only the AIR emails on its demo list. For now nothing checks the instruction with the payment enabler.

Output: `{issued, intent, vct, claims, expires_at, instruction_id, duplicate}`. `claims` holds `network`, `instruction`, `spending_cap` (`amount` in minor units, `currency`, and `period` when the cap renews weekly, monthly or yearly), `title`, `description` and `merchant_categories`. `expires_at` is when the instruction ends (Unix seconds). `--dry-run` prints the claims without asking the service. Run again for the same intent, instruction and email, it issues nothing and prints the earlier result (`duplicate: true`).

## prove

`intent prove --badge <d> [--dry-run] [--program <id>]` asks AIR, over a session this agent signs with its AIR key, to present the funded credential in the user's AIR account, bound to the intent, and sends AIR's answer to the service. The service checks the issuer's signature, the disclosures, the key binding and whether the issuer revoked the credential, then publishes a funded badge. `--program` defaults to the program the service lists for funded badges.

Output: `{badged, address, funded, seq, duplicate}`. `funded` holds `cap` (`amount` in minor units, `currency`, and `period` for a renewing cap), `title`, `description`, `network`, `instruction`, `merchant_categories`, `issuer`, `assurance` (`holder-bound`), `checked_at`, `expires_at` and `badge_id`. `--dry-run` asks AIR and prints `{dry_run, address, program, would_show, note}` without sending anything to the service. Run again with the same credential, it prints the badge already published (`duplicate: true`); a newer credential gets a new badge.

`intent prove --request <request id> [--dry-run]` answers a merchant's `proof_request` from the inbox, privately. It answers only a request about one of this agent's open intents, before it expires, whose `program_id` the service lists for its `claim` in `/.well-known/intent`. AIR presents the credential bound to the merchant's nonce, and the proof goes only to that merchant.

Output: `{sent, id, type, request, merchant, intent, disclosed, duplicate}`. `disclosed` lists every credential the merchant receives, newest first, each with `cap`, `title`, `description`, `network`, `instruction`, `merchant_categories`, `issuer` and `credential_expires_at`. `--dry-run` asks AIR and prints `{dry_run, request, merchant, intent, claim, program, would_disclose, note}` without sending anything. Run again after it was sent, it sends nothing new (`sent: false, duplicate: true`).

## offers

`intent offers <d> [--all]` lists the latest offer from each merchant, best first: offers that fit, then offers whose fulfillment does not fail the intent's, then in the intent's currency, then the lowest total. `--all` includes expired offers.

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
| `loyalty` | memberships the merchant recognizes, keyed by claim: `name`, `tiers` with `benefits`, `rewards` with `earning_forecast`, and `provisional` until verified. `null` when the merchant sent none |
| `totals` | the checkout's price lines as the merchant sent them, in order: `type`, `amount` in minor units, `display_text`. Discounts are negative |
| `policies` | return, warranty and other terms: `type`, `description`, `applies_to`, `url`. `[]` when none |
| `messages` | the merchant's notes as sent: `type` (`info`, `warning`, `error`), `code`, `content`, `path`, and `presentation` on warnings. `[]` when none |
| `disclosures` | the warnings among `messages` with `presentation: "disclosure"`: show each one, word for word, beside what its `path` names |
| `fulfillment` | how the order arrives: `methods`, each with a `type` (`shipping`, `pickup`, ...), the line items, and `groups` with `options` (`title`, `description`, `totals`, `earliest_fulfillment_time`, `latest_fulfillment_time`) and the `selected_option_id`. `null` when the merchant sent none |
| `fulfillment_match` | `{type, window}`: `true` when the offer arrives the way and in the time the intent asked, `false` when not, `null` when the offer or the intent does not say |
| `payment_terms`, `selected_term_id` | every payment term offered, each with `title` and `schedules` (`description`, `amount`, `type`, `due_at`), and the one in effect. `null` when payment is due in full at checkout |
| `due_now` | what is charged at checkout, in minor units: the selected term's `immediate` schedules, or `total` when there are no terms |
| `endpoint`, `checkout_id` | where the merchant's checkout lives |
| `supersedes` | the earlier offer this one replaces |

## accept

`intent accept <offer id> [--dry-run] [--checkout <file>]` checks the offer against the checkout that will be paid: the merchant's live session, re-read from the offer's endpoint, or the file an enabler created. Only when they match does it send the accept.

Output: `{sent, id, duplicate, offer, checkout, handoff, other_offers}`. `checkout` has `live_session`, `source` and `verified`, and the terms of the checkout that will be paid, in the same fields as in `offers`: `totals`, `policies`, `messages`, `disclosures`, `fulfillment`, `fulfillment_match`, `payment_terms`, `selected_term_id` and `due_now`. A dry run prints the same without sending; when the merchant has no session, it shows the offer's terms. Run again after it was sent, it sends nothing new (`sent: false, duplicate: true`) and prints the handoff again. The handoff is described in [payment.md](payment.md).

The checkout must keep the offer's items, prices, total, the fulfillment option selected for each item and the selected payment term. A `false` in `fulfillment_match` does not stop the accept; show it to the user.

## decline

`intent decline <offer id> --reason <reason>`, where `<reason>` is `price`, `timing`, `fit`, `chose-another`, `expired` or `other`. Output: `{sent, id, type, duplicate}`.

## fulfil / cancel

`intent fulfil <d>` and `intent cancel <d>` publish the closed version. Output: `{id, address, d, status, seq, duplicate}`; `duplicate: true` means it was already closed that way. Cancelling an expired intent changes nothing.

## board / inbox

`intent board --author me --status all` lists this agent's intents, with `funded` on those that have a funded badge. `intent inbox --new` shows private messages that arrived since the previous `--new` run: offers, which also appear in `intent offers <d>`, and merchants' proof requests (`type: proof_request`, with `claim`, `program_id`, `fields`, `nonce` and `expires_at` in `content`).

## Errors

Every error has a `hint` with the next step. The common ones:

| Code | What happened |
| --- | --- |
| `KEY_MISSING` | no key yet: `intent key init` |
| `NETWORK`, `TIMEOUT` | the service is unreachable; retrying is safe |
| `DRAFT_FLAGGED`, `INTENT_FORBIDDEN_DATA` | the text holds private data; `details.flags` names the fields |
| `INTENT_CLOSED` | the intent is cancelled, fulfilled or expired |
| `RATE_LIMITED` | too many posts or open intents for this key; wait |
| `OFFER_MISMATCH` | the checkout no longer matches the offer: items, prices, total, fulfillment option or payment term; `details.mismatches` says how |
| `OFFER_SUPERSEDED` | the merchant sent a newer offer; `details.latest` is its id |
| `OFFER_EXPIRED` | the offer lapsed |
| `CHECKOUT_NOT_LIVE` | the merchant has no session; `details.handoff` has the terms for the payment enabler |
| `UCP_UNAVAILABLE` | the merchant's checkout endpoint failed; try later |
| `AIR_NOT_BOUND` | this agent is not connected to AIR yet: `intent air bind` |
| `AIR_SESSION_FAILED` | AIR no longer accepts the agent's key: `intent air bind --new` |
| `AIR_BIND_UNAVAILABLE` | this service does not connect agents to AIR |
| `INSTRUCTION_INVALID` | `fund` got JSON that is not a payment instruction |
| `INSTRUCTION_NOT_ACTIVE` | the payment instruction is not signed yet, or has ended |
| `FUNDING_NOT_ALLOWED`, `FUNDING_UNAVAILABLE` | this service does not fund this user; the intent stays up unfunded |
| `ISSUER_FAILED` | the credential issuer failed; try again in a minute |
| `CREDENTIAL_NOT_FOUND` | the user's AIR account holds no funded credential yet: fund the intent first |
| `PROOF_NOT_COMPLIANT`, `PROOF_REJECTED` | AIR's presentation did not pass the checks; the intent stays up without a badge |
| `CREDENTIAL_REVOKED` | the issuer revoked the credential, so its instruction has ended |
| `BADGES_UNAVAILABLE` | this service gives no funded badges |
| `ISSUER_UNAVAILABLE` | the credential issuer did not answer; try again in a minute |
| `REQUEST_NOT_FOUND` | no proof request with that id in the inbox |
| `PROOF_NOT_ALLOWED` | the request asks through a program the service does not list; do not answer |
| `REQUEST_EXPIRED`, `REQUEST_NOT_YOURS` | the request has lapsed, or is about another agent's intent; ignore it |
| `USAGE` | a wrong flag; `details.examples` shows correct calls |
