# 4. Seller (Creator) setup

The Creator is the Bitkit that receives payments. It connects to paykit-server once,
through paykit-server's own setup page; your service only embeds it.

## The /setup iframe

Embed `GET {paykit}/setup?return_to=<your page>&state=<random>` in an iframe on an
admin page whose origin is listed in `[setup] allowed_origins`.

- On desktop it shows a QR code; on phones, a "Continue with Bitkit" link.
- Bitkit is asked for `/pub/paykit/:rw` plus companion claims
  (`x-bitkit-claim=paykit-access-v1.watch-only-account-v1[.usdt-address-v1]`), so
  paykit-server can derive Bitcoin addresses and learn the USDT address.
- When it finishes, the iframe posts `{ type: "paykit-setup-callback", state }`
  (with `error` on failure) to your exact origin. Check `event.origin` and `state`.
- Use a fresh `state` per attempt so a late callback from an old one is ignored.

To add USDT to a Creator first set up without it (Bitkit only shares the USDT address
once its USDT wallet exists): `GET {paykit}/setup/reconnect?creator=pubky…&asset=USDT&return_to=…&state=…`.

## Check readiness

Signed `POST /setup/status`:

```json
{"accepted_asset":"USDT","creator":"pubky…"}
```

| Answer | Meaning |
|---|---|
| `ready` | Payment requests can be sent for this asset. |
| `setup_required` | Run the setup (or reconnect) again. |
| `unavailable` | Transient: storage, rate limit, network. Retry; do **not** start a new setup because of it. |

Store the Creator's Pubky key in your service's config (e.g. `CREATOR_PUBKY`) once it
reads `ready`.
