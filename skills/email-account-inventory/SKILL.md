---
name: email-account-inventory
description: Inventory every service that uses a given email address as a login by sweeping a Gmail mailbox in five passes, then deliver a sorted migration tracker (markdown + xlsx). Use when retiring, migrating, or auditing an email address.
---

# Email account inventory (Gmail)

Build one deduplicated table of every service, vendor, or platform that has a target email address on file as a login or primary contact, so the user can migrate or close each account before retiring the address.

**Inputs to confirm up front (ask only if not given):** the target address (`TARGET`), the user's own company domain(s) to exclude from "services" (`OWN_DOMAIN`), and the Gmail account to search (the connected Gmail connector is assumed to be that mailbox).

## Ground rules (apply to every pass)

- **One running table, never restarted.** Keep it in a working file (`account-migration-table.md`) and extend it each pass. Columns:
  `# | Service | Sender domain | Type of evidence | Most recent email date | Likely has login`
- **One row per company, not per sender address.** Fold every Apple sender into "Apple"; Workspace/GCP/Play/Firebase into "Google"; AWS/signin.aws/sns into "AWS"; multiple Slack workspaces into "Slack". Put the sub-products and account identifiers (org names, usernames, account IDs, project names) in the Service cell — they are what the user will need when logging in.
- **Likely has login values:** `Yes` (reset, code, signup, or invoice seen), `Probably` (automated notifications, receipts, or product emails only), `Newsletter only` (marketing list only). Never downgrade a row once it is Yes.
- **Type of evidence** lists the strongest evidence first and keeps dates for notable events (password reset 2025-06-30, email changed, charge failed).
- **CRITICAL flag:** prefix the Type cell with `**CRITICAL — <reason>**` for domain registrars, DNS hosts, certificate authorities / cert-issuing accounts (ACM, Apple Developer certs), email-sending DNS (SES/DKIM), and app-store accounts (Apple Developer / App Store Connect, Google Play Console). Losing these breaks the business.
- **Never reproduce secrets.** API keys, temp passwords, recovery codes, and one-time codes appear in snippets. Note that a key exists ("receipts include API key") but never copy the value into the table, file, or reply.
- **Flag anomalies** as you go: the account under a different name, a Relay/masked address, a recent "email changed" notice, repeated failed charges, a security alert — these change what the user does first.
- Use `mcp__Gmail__search_threads` (load via `ToolSearch select:mcp__Gmail__search_threads`). `pageSize` max is 50; page with `pageToken`. Results return only the 5 oldest messages per thread, so do not assert how a thread ended. When a query returns mostly one high-volume sender, re-run it with `-from:<that domain>` to see what is underneath. If a result is truncated (>25k tokens), read the saved tool-result file to see the tail and say a service may have slipped through.
- Every query is scoped `to:TARGET` and starts at the beginning of the mailbox; use `before:`/`after:` windows to page through years when a query is noisy.

## Pass 1 — signup / account creation

`to:TARGET subject:(welcome OR "confirm your email" OR "verify your email" OR "your account has been created" OR "thanks for signing up" OR "activate your" OR "account created" OR "get started")`

Also search body text for the same phrases without `subject:`. Add each service; Yes if the email is a genuine signup/verification, Probably if it is a welcome to a trial or incomplete enrollment.

## Pass 2 — password reset / sign-in security / 2FA

`to:TARGET ("reset your password" OR "password reset" OR "new sign-in" OR "new login" OR "security alert" OR "one-time code" OR 2FA OR "unusual activity" OR "was this you" OR "verification code" OR "two-factor" OR "sign-in code" OR "login code")`

Anything that sent a reset or a code is **Yes**. Record the most recent reset date in Type of evidence.

## Pass 3 — billing

`to:TARGET (invoice OR subscription OR "payment received" OR "payment failed" OR receipt OR renewal OR "trial ending" OR "trial is ending" OR "card expiring" OR "card is expiring" OR "payment method")`

Billing means the address is on the account: mark **Yes** even with no signup seen. Note card last-4 only (never full numbers), billing cadence, and any failed/expiring-card notices. A Stripe-sent invoice from an unidentified merchant becomes a row "Unknown Stripe merchant (acct\_…)" marked Probably.

## Pass 4 — developer & infrastructure

Run as several smaller queries (one big OR is too noisy):

- `to:TARGET ("API key" OR quota OR "usage limit" OR "rate limit")`
- `to:TARGET (domain OR DNS OR SSL OR certificate OR registrar OR "zone" OR DKIM)`
- `to:TARGET (deploy OR "build failed" OR "build succeeded" OR pipeline)`
- `to:TARGET (GitHub OR GitLab OR Bitbucket)`
- `to:TARGET ("Apple Developer" OR "App Store Connect" OR TestFlight OR "Google Play" OR "Play Console")`
- `to:TARGET (AWS OR "Amazon Web Services" OR Cloudflare OR DigitalOcean OR Vercel OR Netlify OR Heroku OR Firebase OR "Google Cloud")`
- `to:TARGET (Stripe OR Twilio OR Shopify OR SendGrid OR Mailgun OR MongoDB OR Datadog)`

Apply the **CRITICAL** flag here. Capture account IDs, org names, project names, domains hosted, and certificate expiry dates in the Service / Type cells.

## Pass 5 — sender-domain sweep

Goal: catch services none of the keyword passes hit. Sample the mailbox in 3–5 date windows (e.g. one query per ~12–18 months, `to:TARGET -from:OWN_DOMAIN -from:<domains already in table…>`, 50 results each, `THREAD_VIEW_METADATA_ONLY`), tally sender domains, and list every domain seen more than twice that is not OWN_DOMAIN and not already in the table.

For each, one line: **account-like** (noreply@, notifications@, account@, no-reply@, automated subjects) or **correspondence** (a named person writing). Add account-like senders to the table as **Probably**; leave correspondence out but list it in the reply. Say plainly that this pass is sampled, not exhaustive. While tallying, note any newer "most recent" dates for services already in the table and apply them.

## Final step — sort, count, stale list

1. Sort: **CRITICAL** first, then **Yes**, then **Probably**, then **Newsletter only**; most recent email first within each group.
2. Dedupe once more by company; remove rows that are individual people rather than services (a human emailing through a service's platform is still that service).
3. Give the count per group and the grand total.
4. **Stale list:** every row whose last email is more than two years before today. Call these likely dead / skippable, but single out any stale row that still matters: a registrar "about to lose your domains" notice, an HR/payroll provider (records), anything that issued an API key that may still be in code.
5. State the number of active accounts to migrate (total minus stale).

Deliver the final table in the reply (text) and keep the markdown file current.

## Spreadsheet deliverable (when asked, or offer it)

If a Sheets artifact type is available, use it; otherwise build `.xlsx` with openpyxl per the `xlsx` skill (Arial, formulas not hardcoded values, run `recalc.py`):

- **Accounts** sheet: `# | Group | Service | Sender domain | Type of evidence | Most recent email (real date cell) | Likely has login | Stale (>2 yrs) | Status | New email set? | Notes`. Stale is a formula against an editable cutoff cell on Summary. Status is a dropdown (To do / In progress / Migrated / Closed / Skip (dead)); New email set? is Yes/No. Group column color-coded; table with filters; freeze panes.
- **Summary** sheet: title, source line (mailbox span, passes run), editable stale-cutoff cell (blue text, yellow fill), per-group `COUNTIF`/`COUNTIFS` table (Total / Stale / Active / Migrated / Remaining) with a Total row, and a legend explaining CRITICAL / Yes / Probably / Newsletter only / Stale plus the stale rows worth a second look.
- Verify Summary counts match the text table before sending.

## Reply style

Lead with the deliverable (table or file), then counts, then anomalies and the "do these first" list. Keep caveats to one or two lines (sampled pass 5, truncated results, 5-oldest-messages limit).
