# Safety rails for a newsletter job an AI agent can run

The send job runs unattended once a week, and an AI agent edits it between runs. Each rail below
is a test in the reference build, so it can't be removed by accident. They exist because the
damage of one wrong send is permanent: you can't unsend 3,000 mails, and you can't un-resubscribe
someone who opted out.

| Rail | Test asserts |
|---|---|
| Dry run goes to the test segment only | Send refuses if the test segment holds any address not matching `you+nl\d*@your.domain` |
| GO_LIVE flag | Without it, real segments are never targeted |
| PAUSE file | Present → every command exits before any API call |
| Unsubscribe link present | Rendered HTML without `{{{RESEND_UNSUBSCRIBE_URL}}}` → no send |
| Draft first, send second | Broadcast is created as a draft, read back, segment and first link checked, then sent |
| One send per ISO week + language | Ledger hit → skip; duplicate broadcast name → refuse, re-checked right before create |
| Ledger after acceptance | Ledger written only after Resend returned the send id |
| Never mail a post twice | Picked posts exclude every slug in the ledger |
| Never resubscribe | Contact create path does a GET first; POST only on 404 |
| Lock | Second concurrent run exits on the file lock |
| 429 | Retried with backoff; bulk segment writes paced |
| LLM copy | Invalid JSON, length, banned words, em dash → fallback copy, issue still sends |
| Resend to non-openers | ≥ 48 h, totals match, excludes openers/clickers/unsubscribed/bounced/complained, > 85 % non-openers → skip, segment verified by set equality |

Run every command with `--dry-run` first. In dry runs the job prints what it would send and to whom,
sends to the test segment, and writes nothing to the ledger or the queue.
