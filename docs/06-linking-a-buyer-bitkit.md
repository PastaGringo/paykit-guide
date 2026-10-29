# 6. Linking a buyer's Bitkit

A payment request is addressed to the buyer's **Pubky key** (`reader`). Asking the
buyer to paste it works, but proves nothing. Having Bitkit approve a Pubky sign-in
proves the key belongs to them.

## Use GrantAuthFlow, not AuthFlow

| Flow | Link | Handled by |
|---|---|---|
| `AuthFlow` | `pubkyauth://signin?…` | Pubky Ring |
| `GrantAuthFlow` | `pubkyauth://signin_grant?…` | **Bitkit** (and Pubky Ring) |

Bitkit's Android manifest registers `signin_grant` and `signup_grant`, not `signin`.

## At least one capability

Bitkit's Paykit SDK rejects a grant whose capability list is empty
(`"Pubky capabilities must contain at least one valid entry"`) and the app only shows
**"Unknown error"**. Ask for the least there is, e.g. read access to a folder you
never use:

```ts
import { AuthFlowKind, GrantAuthFlow } from '@synonymdev/pubky'

const flow = GrantAuthFlow.start('/pub/myservice/:r', AuthFlowKind.signin(), {
  clientId: 'myservice.example',          // shown to the user in Bitkit
})
showQrAndLink(flow.authorizationUrl)      // QR on desktop, link on the phone

const session = await flow.awaitApproval() // resolves when the user approves
const buyerPubky = session.info.publicKey.z32()
await session.signout().catch(() => {})    // only the key was needed
session.free(); flow.free()
```

Bitkit displays it as access to "your myservice data, to read and modify" whatever
the access level: explain on your page that you only keep the public key.

## Server-side notes

- `awaitApproval()` runs in your server process for up to ~5 minutes (the relay drops
  messages after that). It needs a long-lived process, not a serverless function.
- Keep pending flows in memory keyed by an id, bound to the signed-in account, and let
  the page poll that id.
- Approval goes through the buyer's homeserver: if it is down, linking fails, but
  nothing else in your service depends on it afterwards.
- Log approval failures: the wallet's message is generic.
