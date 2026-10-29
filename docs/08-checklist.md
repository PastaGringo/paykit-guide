# 8. Checklist and pitfalls

## Adding a service to an existing paykit-server

- [ ] Generate a dedicated Ed25519 signing key; store the seed in a secrets manager.
- [ ] Add its `pubky…` public key to `[signed_services] trusted_public_keys`.
- [ ] Add the service's origin to `[setup] allowed_origins` if it embeds `/setup`.
- [ ] **Restart** paykit-server (config is read at startup) and check the container
      really restarted.
- [ ] Signed `POST /setup/status` with the new key answers `200`, and the other
      services' keys still answer `200`.
- [ ] Configure the service: paykit URL, signing seed, Creator public key.

## Taking payments

- [ ] Amounts as base-unit strings (1 USDT = `"1000000"`).
- [ ] Unique `operation_id` per attempt.
- [ ] `invoice_id` and amount stored on your side at activation.
- [ ] Payment recorded idempotently (unique `invoice_id`), credit in the same transaction.
- [ ] Never trust an amount coming from the browser.
- [ ] The buyer's Pubky key comes from a Bitkit approval (chapter 6), not only a paste.

## Operations

- [ ] paykit-server version pinned (commit or tag), upgraded on purpose.
- [ ] Postgres **and** `PAYKIT_MASTER_KEY` backed up together.
- [ ] A health check on `/health/ready` and one signed `/setup/status`.
- [ ] On Dokploy, every file mount has a file name (chapter 2).

## Pitfalls, one line each

- Empty capabilities in a Bitkit sign-in → "Unknown error".
- `AuthFlow` (`signin`) → Bitkit does not open it; use `GrantAuthFlow`.
- Unsorted JSON keys → `400 invalid_request`.
- New key not loaded → `401 invalid_signature` until restart.
- Same `operation_id` with another body → `409 operation_conflict`.
- `unavailable` from `/setup/status` is transient: do not restart the setup for it.
- Expecting allowances to auto-pay USDT: they don't (BTC/Lightning only).
