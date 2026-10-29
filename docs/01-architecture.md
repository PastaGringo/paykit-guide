# 1. Architecture

```
 Buyer's Bitkit (Reader)                    Seller's Bitkit (Creator)
        ▲   │ pays USDT / BTC                        │ one-time setup:
        │   ▼ on-chain or Lightning                  │ delegates /pub/paykit/:rw
        │                                            ▼
        │   payment request over Paykit      ┌────────────────┐
        └────────────────────────────────────│  paykit-server │◄── Postgres
                                             │ (watches chains)│
                                             └───────▲────────┘
                                                     │ signed HTTP
                                             ┌───────┴────────┐
            browser ─────────────────────────│ your service   │◄── its own DB
                                             └────────────────┘
```

## Who holds what

| Party | Holds | Never holds |
|---|---|---|
| Your service | Its Ed25519 **signing key**, the Creator's public key, the Reader's public key per customer, invoice ids, its own ledger | Any wallet key, any Pubky secret key of a user |
| paykit-server | Delegated Paykit credentials of each Creator (encrypted with `PAYKIT_MASTER_KEY`), invoice state, chain observations | Spending keys |
| Creator's Bitkit | The seller's wallet and Pubky identity | — |
| Reader's Bitkit | The buyer's wallet and Pubky identity | — |

## How one payment goes

1. Your service calls `POST /marketplace/payment-requests/prepare`, then `.../activate`
   (signed), naming the Creator, the Reader and the amount.
2. paykit-server delivers the payment request over Paykit; it shows up in the
   Reader's Bitkit within seconds.
3. The buyer approves in Bitkit, which pays on-chain (USDT on Arbitrum, or BTC).
4. paykit-server sees the transfer and the payment proof, and moves the request to
   `payment_state: "confirmed"`.
5. Your service polls `.../status` and records the payment once (idempotently).

## One paykit-server or several?

One server can serve several services: each service gets its **own signing key**
in `trusted_public_keys`, and payments are told apart by Creator and invoice id.

| | One shared server | One per service |
|---|---|---|
| Ops | one config, one DB, one upgrade | ×N |
| New service | add a key | new server, new DB, new Creator setup |
| Outage | every service stops taking payments | only that one |
| Bad upgrade / master-key leak | hits every service | isolated |

Share while the services belong to the same owner and Creator and volume is low.
Split when a service takes real volume, collects for someone else, or needs its own
upgrade schedule. Note that every listed key may call **every** signed route: there
is no per-route or per-Creator scoping between services.
