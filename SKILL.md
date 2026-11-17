---
name: newsletter-playbook
description: Build and launch your own automated weekly email newsletter on top of an existing blog, with Resend Broadcasts for sending, your site (Astro on Cloudflare Workers in the reference build) for signup and language switch, and PostHog + Slack for analytics. Covers vendor choice and unsubscribe isolation, sending subdomain and DNS, audience import and language split, a content queue that mixes new and archive posts, the issue template, a weekly send job with safety rails and dry runs, reporting, warm-up and go-live. Use when someone says "set up a newsletter", "weekly email from my blog", "newsletter on Resend", "launch a newsletter", "newsletter automation", or asks how the Coffee with Marian newsletter was built. NOT for one-to-one mail or cold outreach.
---

# Newsletter playbook (Resend Broadcasts + your own blog)

Distilled from building "Grab a coffee with Marian", the weekly newsletter of marian.coach, in
November 2026. Every API behaviour below was checked live against Resend at that time, not read
from docs alone. Where the docs and the API disagreed, the API won and the trap is written down in
`references/resend-traps.md`.

Work order: settle the questions in section 0 with the owner → write a short spec → write a plan
with one testable task each → build with tests first → one review of the whole branch → go live,
with the owner's yes on every step that touches real subscribers.

## 0. Questions to settle with the owner first (one batch)

1. Brand, sender name and address. A first name plus the brand reads better than a full name.
2. Who is on the list and where they came from (signup form, an export from another tool). The
   consent question is the owner's call; write their answer into the spec.
3. Unsubscribe scope: must unsubscribing from this newsletter keep the person on other lists the
   owner runs? This decides the vendor (section 1).
4. Languages, cadence, day and hour, max items per issue, new vs archive content mix.
5. Where the queue lives (a Notion database works) and who can skip an item.
6. Which numbers matter (opens, clicks, site visits, bookings) and where reports go (a Slack channel).

## 1. Vendor and unsubscribe isolation (decide before any code)

The one-click unsubscribe button that Gmail and Apple Mail show (RFC 8058 `List-Unsubscribe`) is
the part people skip, and it decides your architecture:

| Tool | What the mail-app one-click button unsubscribes from |
|---|---|
| Resend Broadcasts | **Every broadcast of the Resend team.** Topics only scope the footer link. Suppression list is team-wide across domains. |
| Ecomail (one account) | All lists in the account, even with "unsubscribe from all" off. Footer link is per list. |
| Kit, SmartEmailing | All lists (vendor answers, Nov 2026) |
| Ghost, Customer.io, beehiiv, Postmark streams | Per newsletter / list / stream |
| Cloudflare Email Sending | Not applicable: the FAQ says transactional only, and it has no open or click events |

**Rule:** a newsletter that must not share unsubscribes with another list gets its own vendor or
its own Resend team. Inside one team every broadcast also shares one bounce and complaint budget;
cross it and sending pauses for the whole team, one-to-one mail included.

Resend Marketing is billed per contact with unlimited sends: Free = 1,000 contacts and **3
segments**; Pro = $40/month for 5,000 contacts and unlimited segments (resend.com/pricing, Nov
2026). You need at least 4 segments (two languages, one test, one resend), so plan on Pro.

## 2. Domain and DNS

- **Send from a subdomain** (`coffee@with.example.com`). Open and click tracking is set per
  domain and cannot be removed later; on the root domain it would rewrite links in your normal mail.
- `POST /domains {name, region: "eu-west-1", open_tracking, click_tracking, tracking_subdomain: "links"}`,
  then create the records it returns in your DNS (**DNS only, never proxied** on Cloudflare):
  DKIM TXT, SPF MX + TXT on `send.<sub>`, the return-path CNAME, the tracking CNAME. Then
  `POST /domains/{id}/verify`. It verified within minutes.
- Check `dig CAA <root>` first. Keep **one** DMARC record on the root; without `sp=` it covers
  subdomains.
- A new subdomain counts as a new domain for warm-up (section 9).

## 3. Contacts

- **Create contact properties first** (`POST /contact-properties`, type string). Writing an
  undefined property fails.
- Merge tags have a fallback but no condition: `{{{contact.first_name|there}}}`. If you want a
  localized greeting, store the whole greeting per language as a property. In the end we dropped
  greetings; the issue reads better going straight to the point.
- **Import** with `POST /contacts/imports` (multipart CSV, `column_map`, `segments`,
  **`on_conflict=skip`** so nobody who unsubscribed gets switched back on). Poll the import until
  `counts.failed == 0`. Imports do not fire `contact.created` webhooks.
- Import active people only. Drop anyone unsubscribed or bounced in the source tool, and trust each
  row's own status field over the source API's filters.
- **Language split** when the source has no language field: country TLD, diacritics in the name,
  or a first-name list for the language. Then ship the language switch (section 5b) so wrong
  guesses fix themselves. In our import 1,440 of 3,134 contacts went to Czech.
- **Test segment** with only your own plus-addresses (`you+nl@…`). The send code refuses a dry run
  if anyone else is in it. Put one unsubscribed test contact in it too.
- Your signup form adds new people to the right language segment and sets the properties.

## 4. Content queue

One database row per post: title, URL, translated URL, published date, status (Queued, Backfill,
In issue, Announced, Skipped), issue, sent date. A daily sync reads the blog's RSS feed: posts
published after go-live become Queued; older ones Announced; a one-off `backfill-queue --from=N`
turns archive posts into Backfill.

Pick rule that worked: **at most 3 posts; while archive posts remain, at most 2 new plus at least
1 archive post, oldest first.** A weekly newsletter on a blog that publishes irregularly would
otherwise go out empty, and the archive gets a second life. Never pick a post the ledger already
mailed. Skipping = set status to Skipped. When 1 post or fewer is available, refill up to 3 with
the most visited posts of the last 30 days (analytics), skipping posts mailed in the last 8 issues.

## 5. Issue template

- Take brand tokens from the site's CSS (accent colour, display font, mono font).
- Layout: neutral page background → header card (round photo, newsletter name, `☕ 2026 W47 ·
  tagline`, brand logo on the right, vertically centred) → one white rounded card per post (image,
  number badge, archive tag, reading time, a big serif title with an ↗ arrow, one-sentence teaser
  ≤ 160 characters, **bold** key words) → one offer card (a random card read live from the
  homepage, with a static fallback) → two side-by-side cards: "meet me for a coffee" and the next
  event read live from your event page (they stack on phones) → end card (language switch button,
  1–5 star rating links, forward line) → footer with the **only** unsubscribe link.
- No intro paragraph in the header. The posts carry the context, and readers skip the paragraph
  anyway. Put the intro line in the preheader (`preview_text`), where the inbox shows it.
- The posts are the highlight. Offer, meet and end cards sit on the page grey with a thin border,
  not on white and not with an accent border, so the eye goes to the posts first.
- Give it air: 32 px between cards, 36–40 px padding inside a card (about 24 px on phones), body
  text 16–17 px at line height 1.7, post titles about 30 px bold (26 px on phones). The goal is a
  newsletter people can read in two minutes.
- Change the template only with a check: regenerate the golden render fixtures, read their diff
  (only the lines you meant may change), screenshot at desktop and phone width, then send a dry run
  to the test segment and read it in a real inbox.
- Name issues by ISO week (`2026 W47`) in the header and the subject. Numbers like `#001` make a
  new newsletter look new.
- Write in first person. Arrows, not "Read more" buttons.
- Responsive with tables only: 600 px max, `@media (max-width:480px)` shrinks padding and titles.
- Always send a plain-text part; no merge tag may stay raw in either part.
- `.webp` does not show in desktop Outlook. Use PNG or JPG for the photo and logo.
- **Hard rule in code: refuse to send an issue without `{{{RESEND_UNSUBSCRIBE_URL}}}`.** Resend
  adds `List-Unsubscribe` and `List-Unsubscribe-Post: One-Click` itself (verified in raw headers).

### 5b. Language switch button

Details and code in `references/language-switch.md`. Short version: Resend does not merge
`{{{contact.id}}}` (it renders empty), so write a signed per-contact property `lang_token` =
`<contact id>.<first 24 hex of HMAC-SHA256(secret, id)>` and link to a site endpoint with it.
**GET only shows a confirm button; the POST from that button switches.** Corporate mail scanners
open every link in a mail at delivery, so a switching GET would flip people who never clicked.

## 6. Copy generation with an LLM

Per issue the LLM writes topics for the subject, one intro line (it becomes the preheader) and one
teaser per post, returned as JSON. Validate lengths, banned words, no em dash; on any failure fall back to plain copy and
still send. A newsletter that goes out with plain copy beats one that misses its day.

If you call Claude Code (`claude -p`) from a scheduled job, unset `ANTHROPIC_API_KEY` for that
subprocess when you want it to run on your subscription, not on API credits. Try the top model,
then the next one, then the fallback copy.

## 7. Send job and its safety rails

Broadcast flow: `POST /broadcasts` as a **draft** (never `send: true`) → `GET` it back and check
the segment and that this issue's first link is in the HTML → `POST /broadcasts/{id}/send` →
ledger → queue update. Rails, each with a test:

- Ledger per ISO week + language, written only after Resend accepted the send.
- A broadcast with the same name is refused, and checked again right before create (copy
  generation takes minutes, a second run could race it).
- A file lock for send, run and resend. A PAUSE file stops everything.
- A GO_LIVE flag in state. Before it, everything goes to the test segment only.
- The queue is updated only if at least one language went out; a failed language retries next run.
- Retry on 429 (Resend allows 10 requests/s per team, shared with your site's own mail); pace
  segment writes at about 4/s.
- **Never resubscribe anyone.** `POST /contacts` on an existing address is an upsert that resets
  `unsubscribed` to false. Look the contact up first and only create on 404.
- Schedule: one daily job at the send hour; the weekday decides the action (preview the day
  before, send, follow-up report two days later, retry the day after), all inside one ISO week.
- **Run it in the cloud, not on a laptop.** The reference build started as a Python job on a Mac and
  moved to a Cloudflare Worker cron so the weekly send does not depend on a laptop being awake. Cron
  triggers are UTC: schedule both possible UTC hours and let the code check the local time, so
  daylight saving can't shift the send. State (ledger, flags, lock) lives in KV. Port with a
  **golden render test**: the old and new renderer must produce byte-identical HTML for fixed inputs.
  After the port, the fixtures are the new renderer's own output, and every intended template change
  regenerates them; the fixture diff is the design review.

## 8. Numbers: Resend, Slack, PostHog

- `GET /emails/metrics?broadcast_id=<id>&dimensions=broadcast` and take the row. **Without
  `dimensions=broadcast` you get zeros**, which looks like a dead newsletter.
- `/broadcasts/{id}/recipients` is cached for up to 15 minutes.
- Slack: one channel for the newsletter. Post the preview with blockers, the send report with
  links, the follow-up with open %, click %, unsubscribes, bounce %, complaint %, star rating, and
  a warning above 2 % bounces or 0.05 % complaints. Post every failure.
- PostHog: UTM `utm_source=newsletter&utm_medium=email&utm_campaign=nl-<n>&utm_content=<slug>` on
  every link to your own site; capture `newsletter_issue_sent` (plus a project annotation, so traffic
  spikes explain themselves) and `newsletter_issue_stats`. The star rating links land on your site
  with `nl_feedback=1..5`, so ratings arrive as pageviews you can count. Mail scanners "click"
  every link at delivery, so count ratings only from JS-executed pageviews, one per person per issue,
  and drop anyone who hit 3+ star values within a minute. Report click % raw and without scanner
  bursts (3+ distinct links within 10 s of delivery).
- Resend to non-openers is a manual command with rails: at least 48 h after the send, recipient
  lists must match Resend's totals, exclude openers, clickers, unsubscribed, bounced and complained,
  skip if more than 85 % look like non-openers (open tracking is broken then, not your readers),
  fill a resend segment and check it by set equality before sending.

- **Clicks go to your CRM, per person, within a minute.** The Resend webhook already tells you who
  clicked which link. Give every button its own `utm_content` (each post, the offer card, the
  coffee card, the event, the stars, forward), or you cannot tell the clicks apart. For each real
  click (not a scanner burst, not within 10 s of delivery, not a dry run, not unsubscribe), upsert
  the person by email in the CRM and keep one entry per reader on an engagement list: stage,
  intent (high for the offer, coffee and event cards; medium for a post; low for stars and
  forward; keep the highest), last click, issue, click count, language. Post one line per click
  in a Slack channel of its own. A CRM error must never fail the webhook, or Resend retries and
  you count the click twice.

### 8b. Follow-up queues

A click is a signal, and each kind of signal gets its own follow-up. Never one email per clicked
post: that reads like tracking. The webhook sets the queue on the reader's CRM entry at click
time; a daily job on working days does the follow-up.

| Queue | Trigger | Action | Stops |
|---|---|---|---|
| Mentoring (your offer) | offer card | next working day: a personal note with one question about their situation and a link to a free intro call; no reply in 5 days → one LinkedIn note | reply, booking, 2 touches |
| Coffee | coffee card | next working day: coffee at your office or online, with your booking link; same LinkedIn second touch | reply, booking, 2 touches |
| Event | event card | 48 h later, read the event's guest list: registered → mark it, send nothing; not registered → one "your seat is waiting" mail 5 days before | registration or event date |
| Topic | 2+ clicked posts on one topic, or 3+ clicks over 2+ issues | one helpful note on that topic plus the post's own next step | one per topic per 90 days |
| B2B | a company-level post (fractional, consulting, org design) from someone at a company of 50+ | a short business note instead of the personal offer | reply, one touch |
| Advocates | 5 stars or forward | a weekly thank-you; every second one asks for a one-line quote | one per 90 days |

Rules we run with:

- One personal mail per person per 30 days, across everything you send, not only the newsletter.
  If your outreach tool applies that pause only to cold mail, check it in the job yourself.
- One touch per person per run, in a fixed priority (B2B, offer, coffee, event, topic, advocates).
- Current clients and open deals get no mail from the job, only a Slack line for you.
- Every mail goes through your normal one-to-one sender with its suppression list and duplicate
  guard, from your personal address, plain text, no tracking pixel. Log it in the CRM.
- The copy is written by an LLM per person and checked hard before sending: greeting kept, link
  present, word cap, no em dash, nothing that says "I saw you click". A copy that fails the check
  is not sent.

What surprised us in the dry runs:

- Czech copy came back without diacritics once. The check now refuses Czech text with fewer than
  three accented letters.
- Asked for a thank-you, the model promised future topics nobody had planned. Tell it in the
  prompt not to name future topics, numbers or events.
- A guest list status of "invited" does not mean registered. Count only approved, pending and
  waitlist.
- Read post tags from the live RSS feed, not from a local checkout of the site. Ours lagged the
  live site by two posts.

## 9. Warm-up and go-live

- Pick the send day from data: count the newsletters in your own inbox by weekday, and your blog
  readers by weekday and hour. Ours: Wednesday 11:18 Prague time.
- First issue to a large imported list: most engaged first, split over 3 days (700 / 1,000 / rest)
  inside the same ISO week.
- Go-live checklist, each with the owner's yes: plan paid, import done, dry run read in a real
  inbox (From, DKIM and DMARC pass, one-click header, both languages, links tagged, footer
  unsubscribe, no raw merge tags, plain-text part), job scheduled, GO_LIVE set, first follow-up
  numbers checked.

## 10. Worth adding (not built yet in the reference build)

1. Postal address in the footer.
2. Double opt-in for new signups, and a consent record (source, date, page) per contact.
3. Verify addresses before import to keep bounces under 2 %.
4. Sunset rule: stop mailing people with no open or click in the last 8 issues, after one
   re-permission mail.
5. Google Postmaster Tools for the sending domain; an inbox-placement test before issue 1.
6. Welcome automation, triggered by a custom event from the signup endpoint.
7. Built since: Resend webhooks → PostHog per person (Svix signature verified) and clicks → CRM with
   follow-up queues (sections 8 and 8b). Still open: feeding clickers into ad audiences.
8. Built since: `preview_text` (the preheader) is accepted on broadcast create though not
   documented. We fill it with the intro line.
9. Resend has no subject A/B. Alternate subject styles week to week and compare opens.

Read next: `references/resend-traps.md`, `references/language-switch.md`,
`references/safety-rails.md`.
