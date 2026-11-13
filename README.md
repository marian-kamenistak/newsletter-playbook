# Newsletter playbook: your own weekly newsletter on top of your blog, built with AI

I had a blog with 100+ posts and an audience of 3,000+ people sitting in another tool. What I didn't
have was a newsletter that sends itself every Wednesday, in the reader's language, and tells me
in Slack what happened.

So I built one with Claude Code on Resend, Astro, Cloudflare Workers and PostHog. It took me 3
hours to get it running on top of my existing blog posts. This repo is the playbook the agent and
I ended with: what to decide first, what to build, and every trap that cost us time.

By Marian, [marian.coach](https://www.marian.coach/). The newsletter it runs is "Grab a coffee with
Marian".

## What you get when you follow it

- A weekly issue built from your blog's RSS feed: at most 3 posts, new ones mixed with archive
  posts, so the issue is never empty and old posts get a second life.
- Two languages, guessed at import, with a **"Prefer English? Next issue in English →"** button.
  One click on a signed link moves the reader. No login, no preference centre.
- Numbers where you already look: open %, click %, bounces, complaints and a 1–5 star rating in
  Slack two days after each send, and every click as a tagged visit in PostHog.
- Rails, each one a test: dry runs go only to your own test addresses, nobody gets resubscribed,
  no issue goes out without an unsubscribe link, one send per week per language.

## What it costs

| | Per month |
|---|---|
| Resend Marketing Pro, up to 5,000 contacts, unlimited sends ([pricing](https://resend.com/pricing)) | $40 |
| Astro site on Cloudflare Workers, PostHog, Slack | what my blog already used |
| Claude Code for the build and the weekly copy | my existing subscription |

The full comparison with Mailchimp, Kit, beehiiv, Substack, Ecomail and a build on Cloudflare's
own email sending, and why I picked this one, is in the blog post (link below).

## Install

```bash
git clone https://github.com/marian-kamenistak/newsletter-playbook ~/.claude/skills/newsletter-playbook
```

Then tell your agent: *"Set up a weekly newsletter for my blog with the newsletter playbook."* It
starts with six questions for you (section 0), because the answers decide the vendor and the
code. Works with any coding agent that reads Markdown; the file is written as a Claude skill.

## What's inside

| File | What it carries |
|---|---|
| `SKILL.md` | The playbook: vendor and unsubscribe isolation, DNS, contacts and language split, content queue, issue template, send job, analytics, warm-up and go-live |
| `references/resend-traps.md` | 12 Resend API behaviours checked live, plus one Cloudflare trap. Some fail silently, like the metrics call that returns zeros |
| `references/language-switch.md` | The signed-token language button, Python + TypeScript, and why GET may never switch anything |
| `references/safety-rails.md` | The rails that make an unattended weekly send safe for an AI agent to change |

## The one thing to decide before any code

Which unsubscribe button kills what. The one-click button Gmail and Apple Mail show unsubscribes a
reader from **every broadcast of a Resend team**, and from every list of an Ecomail account.
If this newsletter must not take your other lists down with it, it gets its own vendor or its own
team. Section 1 has the table.

## Read the story

How I built it, what it costs compared to the hosted tools, and the mistakes:
[marian.coach/blog](https://www.marian.coach/blog/).

MIT licensed. If it saved you a weekend, star it. Stars are how the next person with a blog and
no newsletter finds it.
