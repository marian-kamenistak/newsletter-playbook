# Language switch: "Prefer English? Next issue in English →"

You guessed each reader's language at import. Some guesses are wrong. One button in every issue
lets the reader fix it, with no login and no preference centre.

## Why a signed token

The link must say which contact to move, but Resend does not merge `{{{contact.id}}}` (it renders
empty) and an email address in a URL leaks into logs and analytics. So the send job writes a
property on every contact:

```
lang_token = "<contact id>.<first 24 hex chars of HMAC-SHA256(secret, contact id)>"
```

and the issue links to `https://your.site/api/newsletter-language?t={{{contact.lang_token}}}&to=cs`.
The site checks the signature with the same secret. Anyone without the secret can't forge a token
for someone else. Run the token job daily so new contacts get one too, and only show the button
once every contact has a token (a config flag).

Python (send job):

```python
import hmac, hashlib

def lang_token(contact_id: str, secret: str) -> str:
    sig = hmac.new(secret.encode(), contact_id.encode(), hashlib.sha256).hexdigest()[:24]
    return f'{contact_id}.{sig}'

# pin one test vector in BOTH languages' tests so they can't drift:
assert lang_token('00000000-0000-4000-8000-000000000000', 'k').endswith('.95aac85cc4e727ec61381736')
```

TypeScript (Cloudflare Worker / Astro endpoint, Web Crypto):

```ts
const UUID = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/;

async function hmacHex(secret: string, msg: string) {
  const key = await crypto.subtle.importKey('raw', new TextEncoder().encode(secret),
    { name: 'HMAC', hash: 'SHA-256' }, false, ['sign']);
  const sig = await crypto.subtle.sign('HMAC', key, new TextEncoder().encode(msg));
  return [...new Uint8Array(sig)].map((b) => b.toString(16).padStart(2, '0')).join('');
}

export async function verifyLangToken(token: string, secret: string): Promise<string | null> {
  const [id, sig] = (token || '').split('.');
  if (!id || !sig || !UUID.test(id) || sig.length !== 24) return null;
  const want = (await hmacHex(secret, id)).slice(0, 24);
  let diff = 0;                                   // constant-time compare
  for (let i = 0; i < 24; i++) diff |= want.charCodeAt(i) ^ sig.charCodeAt(i);
  return diff === 0 ? id : null;
}
```

## GET confirms, POST switches

The first version switched on GET. It worked in every test and would have broken in production:
Outlook Safe Links, Proofpoint, Mimecast and similar scanners open every link in a mail at
delivery. A switching GET would flip every reader behind such a scanner to the other language
before they read a word.

So:

- `GET ?t=…&to=cs` verifies the token and shows one question and one button ("Switch to Czech").
  A bad token returns 400 and a "write to me and I'll switch you by hand" page. Nothing changes.
- The button POSTs `t` and `to` (a plain form, same origin). The POST verifies again, then
  `POST /contacts/{id}/segments/{target}` and `DELETE /contacts/{id}/segments/{other}` on Resend,
  and shows "Done. Next issue in Czech."
- Both pages `noindex`. A missing secret returns 503 and logs an error; a Resend failure 502.
- Escape the token when you echo it into the form.

## Test it live

With one test contact: GET → page shows the button, segments unchanged. Forged token → 400.
POST to `cs` → contact is in the CS segment only. POST to `en` → back. Then clean the test contact
out of both language segments.
