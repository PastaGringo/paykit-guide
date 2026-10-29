# 3. Signed requests

Every business route of paykit-server (`/marketplace/…`, `/setup/status`,
`/invoices`…) requires a signature from a key listed in
`signed_services.trusted_public_keys`.

## 1. Generate the service key

A 32-byte Ed25519 seed. Its public key, in Pubky format (`pubky` + z-base-32), goes
into `trusted_public_keys`; the seed goes into your service's secrets, never into
browser code.

```js
// node gen-key.mjs   (npm i @synonymdev/pubky)
import { Keypair } from '@synonymdev/pubky'
const seed = crypto.getRandomValues(new Uint8Array(32))
const pub = 'pubky' + Keypair.fromSecret(seed).publicKey.z32()
console.log('SECRET (base64url):', Buffer.from(seed).toString('base64url'))
console.log('PUBLIC (trusted_public_keys):', pub)
```

One key per service: you can then revoke one service without touching the others.

## 2. The signature

```
preimage  = "paykit-http-signature-v1\0" + UPPERCASE_METHOD + "\0" + path + "\0" + raw_body
signature = Ed25519(seed, preimage)
header    x-paykit-signature: base64url(signature), no padding
```

- `path` is the exact path, without query string.
- `raw_body` is the exact bytes you send.
- There is no body-only fallback.

## 3. Canonical JSON

paykit-server re-canonicalizes the body (RFC 8785) and rejects it with
`invalid_request` unless your bytes are already canonical. For bodies made of
strings, integers, arrays and objects, sorting object keys is enough.

## TypeScript client (Web Crypto, no dependency)

```ts
const DOMAIN = 'paykit-http-signature-v1\0'
// PKCS#8 header for a raw 32-byte Ed25519 seed (RFC 8410).
const PKCS8 = Uint8Array.from([0x30,0x2e,0x02,0x01,0x00,0x30,0x05,0x06,0x03,0x2b,0x65,0x70,0x04,0x22,0x04,0x20])
const enc = new TextEncoder()

const b64url = (b: Uint8Array) => btoa(String.fromCharCode(...b)).replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '')
const fromB64url = (s: string) => Uint8Array.from(atob(s.replace(/-/g, '+').replace(/_/g, '/')), (c) => c.charCodeAt(0))

function canonical(v: unknown): string {
  if (Array.isArray(v)) return `[${v.map(canonical).join(',')}]`
  if (v !== null && typeof v === 'object') {
    const e = Object.entries(v as Record<string, unknown>).filter(([, x]) => x !== undefined).sort(([a], [b]) => (a < b ? -1 : a > b ? 1 : 0))
    return `{${e.map(([k, x]) => `${JSON.stringify(k)}:${canonical(x)}`).join(',')}}`
  }
  return JSON.stringify(v)
}

export async function paykitPost<T>(baseUrl: string, seedB64: string, path: string, body: unknown): Promise<T> {
  const der = new Uint8Array(48); der.set(PKCS8); der.set(fromB64url(seedB64), 16)
  const key = await crypto.subtle.importKey('pkcs8', der, { name: 'Ed25519' }, false, ['sign'])
  const raw = enc.encode(canonical(body))
  const sig = new Uint8Array(await crypto.subtle.sign({ name: 'Ed25519' }, key, new Uint8Array([...enc.encode(`${DOMAIN}POST\0${path}\0`), ...raw])))
  const res = await fetch(baseUrl + path, { method: 'POST', headers: { 'content-type': 'application/json', 'x-paykit-signature': b64url(sig) }, body: raw })
  const data = await res.json().catch(() => null)
  if (!res.ok) throw new Error(`${res.status} ${data?.error?.code ?? ''}: ${data?.error?.message ?? ''}`)
  return data as T
}
```

Errors come back as `{"error":{"code":"…","message":"…"}}`.

## Troubleshooting

| Answer | Cause |
|---|---|
| `401 invalid_signature` | Key not in `trusted_public_keys` (or config not reloaded, see chapter 2), wrong path, or body bytes differ from what was signed. |
| `400 invalid_request` | Body not canonical (keys unsorted), unknown field, or wrong type. |
