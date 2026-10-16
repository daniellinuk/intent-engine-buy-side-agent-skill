# From the handoff to a paid order

`intent accept` prints `handoff`: the checked terms the payment enabler needs.

```json
{
  "v": 1,
  "enabler": "<payment enabler>",
  "offer": "<offer id>",
  "intent": "30468:<buyer pubkey>:<d>",
  "merchant_pubkey": "<hex>",
  "ucp_endpoint": "https://shop.example.com/ucp",
  "checkout_id": "chk_123",
  "live_session": true,
  "currency": "HKD",
  "total": 11000,
  "due_now": 11000,
  "payment_term": null,
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "sku_flat_white_oat_l", "title": "Flat white, oat milk, large", "price": 2200 },
      "quantity": 5,
      "item_url": "https://shop.example.com/products/flat-white"
    }
  ],
  "fulfillment": null,
  "discount_codes": [],
  "eligibility": [],
  "continue_url": null
}
```

- `total` is everything the user owes for this order. `due_now` is what is charged when the checkout completes.
- `payment_term` is `null` when everything is due at checkout; then `due_now` equals `total`. Otherwise it is the term the merchant selected, with its `schedules`: each payment's `description`, `amount`, `type` and, when known, `due_at`. A schedule whose `type` is not `immediate` is charged later.
- `fulfillment` is how the order arrives, as the merchant sent it, or `null`. The merchant priced it for the intent's area, not an address.

## Pay with the payment skill

Use the payment skill's UCP checkout flow. Read that skill first: it owns the route, its own purchase confirmation, the card and mandate checks, and 3-D Secure.

When `live_session` is true, complete that checkout. Give the payment skill the endpoint, `checkout_id`, the currency, the total, and `due_now`. On a funded intent, complete the session with the active payment instruction. Do not create a second checkout from the product URL.

The payment skill asks the user to confirm again. That is expected: the user agreed to the offer, and now confirms the payment.

## Payments due later

When `payment_term` has a schedule whose `type` is not `immediate`, part of the total is charged after checkout: a deposit now and the balance later, payments in installments, or an invoice due in 30 days. Before paying, give the payment skill the term's schedules and ask whether it can fund every one of them with the same payment method. Pay only if it says yes. Otherwise do not pay: decline the offer with `intent decline <offer id> --reason other` and tell the user that the payment method cannot cover the later payments.

On a funded intent the payment instruction must cover the whole `total`, not only `due_now`.

Create a session from the line items only when `live_session` is false. Then the payment skill needs:

| The payment skill needs | Take it from |
| --- | --- |
| the URL of that exact item | `line_items[].item_url` |
| quantity | `line_items[].quantity` |
| unit price | `line_items[].item.price`, converted to major units |
| currency | `currency` |
| total, which must match exactly | `total`, converted to major units |
| the delivery option, when the offer has one | the `selected_option_id` of each group in `fulfillment` |

The created checkout must keep that delivery option and the offer's payment term, or `intent accept --checkout` refuses it.

## Amounts

The handoff uses minor units, as UCP does: `2200` in HKD is HK$22.00. The payment skill takes `item.price` as a major-unit decimal string, `"22.00"`. Convert exactly once:

- two-decimal currencies (HKD, USD, EUR, SGD, GBP): divide by 100, `2200` is `"22.00"`
- zero-decimal currencies (JPY, KRW): no change, `2200` is `"2200"`
- three-decimal currencies (BHD, KWD): divide by 1000

Passing minor units straight through charges 100 times the price.

## When the totals disagree

If the checkout the payment skill builds does not total exactly `handoff.total`, stop. The merchant's price changed, or a discount did not apply. Tell the user; do not pay a different amount without asking.

When `discount_codes` or `eligibility` is not empty, the offer's price relies on them. If the payment skill cannot apply them, the totals will disagree.

## When the merchant has no live session

This is the fallback, for a merchant with no UCP session. `intent accept` fails with `CHECKOUT_NOT_LIVE`, and the handoff is in `error.details.handoff` with `live_session: false`. The payment skill creates the session from the item URLs. If the created total is not exactly `handoff.total`, stop. Tax or shipping that the offer left out is a mismatch. Tell the user. Do not pay it. When the totals match, save the checkout and send the accept, which checks that checkout against the offer first:

```bash
intent accept <offer id> --checkout created-checkout.json
```

## Funding from a payment instruction (temporary, demo only)

`intent fund` turns one payment instruction into the funded credential merchants will see. As soon as the payment skill says the user has signed the instruction, connect this agent to their AIR account if it is not connected yet: run `intent air bind`, send the `url` to the user, and wait until it prints `status: bound`. Then ask the payment skill for that instruction as JSON, and save what it prints. Use an instruction that is active.

`intent fund` reads the instruction's title, description, `amountLimit` and `currencyCode` (or its first mandate's), its mandates' `merchantCategoryCode`, its recurring frequency and `effectiveUntilTime`. `amountLimit` is in major units (`150.00`); the credential holds minor units (`15000`). The CLI converts once, so pass the payment skill's JSON unchanged.

When the user later pays for an offer on this intent, the payment skill's instruction matching should find this same instruction.

## After paying

- Decline the other offers: `intent decline <id> --reason chose-another` for each id in `other_offers`.
- When the order arrives: `intent fulfil <d>`.
- Refunds and payment problems go through the payment skill, not the Intent Network.
