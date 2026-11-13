# Resend traps (checked live, November 2026)

Each line cost time once. Most of them fail silently.

| Trap | What happens | Do this |
|---|---|---|
| `POST /contacts` on an existing address | Upsert. Resets `unsubscribed` to **false**: you resubscribed someone. | `GET /contacts/{email}` first, create only on 404. |
| `{{{contact.id}}}` in a broadcast | Renders **empty**. | Store what you need as a contact property. |
| `{{{contact.email}}}`, `{{{contact.<property>\|fallback}}}` | Work. | Fallback is the only condition you get. |
| `{{{RESEND_UNSUBSCRIBE_URL}}}` | Filled per contact; Resend also adds `List-Unsubscribe` + One-Click headers. | Refuse to send without it. |
| `/emails/metrics?broadcast_id=X` | Returns zeros. | Add `&dimensions=broadcast` and read the matching row. |
| `/broadcasts/{id}/recipients` | Cached up to 15 min. | Don't decide anything on a fresh read right after a send. |
| Free plan | 3 segments max ("Your plan includes 3 segments"). | Pro, or reuse a segment temporarily. |
| Rate limit | 10 requests/s per team, shared with every other mail the team sends. | Retry 429 with backoff, pace bulk writes. |
| One-click unsubscribe | Team-wide. Topics scope only the footer link. | One team per newsletter that must stay isolated. |
| CSV import | `on_conflict=skip` leaves existing contacts untouched; imports fire no webhooks. | Use skip; send your own events if you need them. |
| `preview_text` on broadcast create | Accepted, not documented. | Set it, check it in a real inbox. |
| Undefined contact property | Write fails. | `POST /contact-properties` before import. |
| Cloudflare in front of your endpoint | Blocks a `Python-urllib` user agent with 403. | Test with a browser user agent before you debug your code. |
