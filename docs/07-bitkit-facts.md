# 7. Bitkit facts

Checked in `synonymdev/bitkit-android` on 2026-10-29.

## USDT fees are paid in USDT

Bitkit sends USDT on Arbitrum from an ERC-4337 smart account through a bundler. The
gas is covered for the user and charged back **in USDT**; the app says "All fees are
paid in USDT. Final fee may be lower." The buyer needs **no ETH**, only a little more
USDT than the amount. The seller receives the full amount.

If the balance cannot cover amount + fee: "Unable to pay the transaction fee in USDT.
Try a smaller amount."

## Allowances: BTC over Lightning only

Paykit allowances (a buyer pre-approves recurring requests from one seller, within
limits) exist in the protocol for any asset, but Bitkit only applies them to **BTC
over Lightning**:

- terms whose asset is not `btc` are ignored;
- a request whose amount is not BTC is never paid automatically;
- an on-chain payment is never automatic ("an on-chain fee falls outside every
  allowance limit").

So **USDT payment requests are always approved by hand** in Bitkit. Bitkit's limits
for allowances: 1–50 USD per payment, 10–500 USD per month, converted to sats.

## Error messages

Bitkit maps known errors to specific messages and everything else to "Unknown error"
("Erreur inconnue"). Seen in practice: an empty capability list in a sign-in grant
(chapter 6).

## Pubky identity

Bitkit holds its own Pubky identity (the profile). It is the key to use as `reader`
for a buyer and as `creator` for a seller.
