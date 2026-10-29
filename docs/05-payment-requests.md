# 5. Payment requests

Three signed calls: **prepare**, **activate**, then **status** until paid or expired.

## Prepare

`POST /marketplace/payment-requests/prepare`

```json
{
  "accepted_assets": ["USDT"],
  "amount": "5000000",
  "asset": "USDT",
  "creator": "pubky…seller",
  "description": "Credits · 5 USDT",
  "operation_id": "myservice-<customer>-<unique>",
  "payment_window_seconds": 3600,
  "reader": "pubky…buyer",
  "reference": "<uuid>"
}
```

- `amount` is a **base-unit integer string**: USDT in millionths (`"1000000"` = 1 USDT),
  BTC in sats, USD in cents.
- `operation_id` makes the call idempotent: the same body replays the same
  preparation; a different body under the same id is `409 operation_conflict`.
  **Include something unique** (a timestamp, a uuid) so that a request that expired
  unpaid does not block a new one.
- `accepted_assets` only narrows what the Creator has set up (`["USDT"]`, `["BTC"]`
  or both). Omit it to offer everything.
- The answer carries `invoice_id`, `payment_amounts` and `prepare_expires_at`
  (activate within about 15 minutes).

## Activate

`POST /marketplace/payment-requests/activate`

```json
{"amount":"5000000","asset":"USDT","creator":"pubky…seller","invoice_id":"…"}
```

Amount and asset must match the preparation (`409 total_mismatch` otherwise).
Activation starts the payment window and sends the request to the Reader's Bitkit.

**Store `invoice_id` with the amount you asked for, on your side.** When the
payment confirms, credit the amount from your own record, never from the browser.

## Status

`POST /marketplace/payment-requests/status`

```json
{"creator":"pubky…seller","invoice_id":"…"}
```

Fields you will use:

| Field | Values |
|---|---|
| `state` | `prepared`, `active`, `voided`… |
| `proposal_delivery_state` | whether the request reached the Reader |
| `request_state` | `null` (not projected yet), `proposed`, accepted… |
| `payment_state` | `detected` (receipt, 0 confirmations), `confirmed`, `expired` |
| `paid_asset` | `"USDT"` or `"BTC"` once a qualifying payment exists |
| `usdt_arbitrum` | `null`, or `{tx_hash, confirmations, amount_matched, paid_on_time, finalized}` |

`request_state: null` is not a rejection: it only means nothing was projected yet.

## Recording a payment once

Polls overlap (several tabs, reloads, a background job). Make recording idempotent
with a unique constraint on `invoice_id`:

```sql
INSERT INTO payments (invoice_id, account_id, amount, tx_hash)
VALUES ($1, $2, $3, $4)
ON CONFLICT (invoice_id) DO NOTHING
RETURNING id;   -- credit the account only when a row came back
```

Do it in the same transaction as the credit.

## Windows

- A request paid "on the spot" from your page: `payment_window_seconds: 3600`.
- An automatic renewal the subscriber may approve later: up to the server's cap
  (`payment_window`, 24 h by default). Above it: `400 payment_window_exceeds_cap`.

## Other errors

| Code | Meaning |
|---|---|
| `503 seller_setup_pending` | The Creator has no rail for what you asked: run the setup. |
| `409 asset_unsupported` | USDT asked but the server has no `[usdt]` section. |
| `409 prepare_expired` | Prepare again. |
| `503 dependency_unavailable` | Transient, retry. |
