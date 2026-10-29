# Paykit field guide

How to take payments in a web service with [Paykit](https://github.com/pubky/paykit-rs)
and a self-hosted [paykit-server](https://github.com/pubky/paykit-server), with
[Bitkit](https://bitkit.to) as the wallet on both sides.

It is written from two services running in production on one shared paykit-server:
a subscription (1 USDT a month) and prepaid credits (top-ups of 5–20 USDT). Every
statement here was checked against the code of `pubky/paykit-server`,
`pubky/paykit-rs` and `synonymdev/bitkit-android` on **2026-10-29**, or observed on
the live services. Paykit is pre-production and moves fast: re-check anything that
matters against the current sources.

Agents: start with [`llms.txt`](llms.txt).

## Contents

1. [Architecture](docs/01-architecture.md): who holds what, and how a payment flows.
2. [Running paykit-server](docs/02-paykit-server.md): config, deployment, the pitfalls we hit.
3. [Signed requests](docs/03-signed-requests.md): signing key, signature scheme, canonical JSON.
4. [Seller (Creator) setup](docs/04-creator-setup.md): connecting the Bitkit that receives payments.
5. [Payment requests](docs/05-payment-requests.md): prepare, activate, poll, record.
6. [Linking a buyer's Bitkit](docs/06-linking-a-buyer-bitkit.md): proving which Pubky key pays.
7. [Bitkit facts](docs/07-bitkit-facts.md): fees, allowances, error messages.
8. [Checklist and pitfalls](docs/08-checklist.md).

## Vocabulary

| Term | Meaning |
|---|---|
| **Creator** | The seller. A Pubky identity in a Bitkit that receives the payments. |
| **Reader** | The buyer. A Pubky identity in a Bitkit that pays. |
| **paykit-server** | Receiver-side service. Holds delegated Paykit credentials for Creators, sends payment requests to Readers, watches the chains. Never spends funds. |
| **Service** | Your backend (the shop, the SaaS). Calls paykit-server with signed HTTP requests. |
| **Payment request** | A request sent over Paykit from Creator to Reader, which appears in the Reader's Bitkit. |

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Code snippets: MIT.
