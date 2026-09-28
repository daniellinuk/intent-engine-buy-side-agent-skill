# From the handoff to a paid order

`intent accept` prints `handoff`: the checked terms the payment enabler needs.

```json
{
  "v": 1,
  "enabler": "clink",
  "offer": "<offer id>",
  "intent": "30468:<buyer pubkey>:<d>",
  "merchant_pubkey": "<hex>",
  "ucp_endpoint": "https://shop.example.com/ucp",
  "checkout_id": "chk_123",
  "live_session": true,
  "currency": "HKD",
  "total": 11000,
  "line_items": [
    {
      "id": "li_1",
      "item": { "id": "sku_flat_white_oat_l", "title": "Flat white, oat milk, large", "price": 2200 },
      "quantity": 5,
      "item_url": "https://shop.example.com/products/flat-white"
    }
  ],
  "discount_codes": [],
  "eligibility": [],
  "continue_url": null
}
```

## Pay with the payment skill

Use the payment skill's UCP checkout flow. For Clink, read its `references/clink-ucp-checkout.md` first: it owns the route, its own purchase confirmation, the card and mandate checks and 3-D Secure. Give it these terms:

| The payment skill needs | Take it from |
| --- | --- |
| product URL | `line_items[].item_url` |
| quantity | `line_items[].quantity` |
| unit price | `line_items[].item.price`, converted to major units |
| currency | `currency` |
| total, which must match exactly | `total`, converted to major units |

The payment skill asks the user to confirm again. That is expected: the user agreed to the offer, and now confirms the payment.

## Amounts

The handoff uses minor units, as UCP does: `2200` in HKD is HK$22.00. Clink's `--line-items` takes `item.price` as a major-unit decimal string, `"22.00"`. Convert exactly once:

- two-decimal currencies (HKD, USD, EUR, SGD, GBP): divide by 100, `2200` is `"22.00"`
- zero-decimal currencies (JPY, KRW): no change, `2200` is `"2200"`
- three-decimal currencies (BHD, KWD): divide by 1000

Passing minor units straight through charges 100 times the price.

## When the totals disagree

If the checkout the payment skill builds does not total exactly `handoff.total`, stop. The merchant's price changed, or a discount did not apply. Tell the user; do not pay a different amount without asking.

When `discount_codes` or `eligibility` is not empty, the offer's price relies on them. If the payment skill cannot apply them, the totals will disagree.

## When the merchant has no live session

`intent accept` fails with `CHECKOUT_NOT_LIVE`, and the handoff is in `error.details.handoff` with `live_session: false`. Pay from that handoff the same way; the payment skill creates the session. Then save the checkout it created to a file and send the accept, which checks that checkout against the offer first:

```bash
intent accept <offer id> --checkout created-checkout.json
```

## After paying

- Decline the other offers: `intent decline <id> --reason chose-another` for each id in `other_offers`.
- When the order arrives: `intent fulfil <d>`.
- Refunds and payment problems go through the payment skill, not the Intent Network.
