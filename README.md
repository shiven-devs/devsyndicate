# Dev Syndicate — website

The public website and lead pipeline for Dev Syndicate, a hackathon partner for colleges.
Next.js 16 (App Router, Turbopack), React 19, Tailwind CSS 4, TypeScript.

```bash
npm install
cp .env.example .env.local  # optional in development — see below
npm run dev
```

| Script | |
| --- | --- |
| `npm run dev` / `build` / `start` | Next.js |
| `npm run lint` · `npm run typecheck` | ESLint · TypeScript |
| `npm test` | Unit tests (Node's built-in runner, Node 22.6+) |
| `BASE_URL=… npm run smoke` | Smoke-test a running server: pages, canonicals, sitemap, robots, API guards. Never submits a lead |
| `npm run db:setup` | Create or upgrade the lead, admin, activity and site-content tables in `DATABASE_URL` (idempotent; run before deploying an upgrade) |
| `npm run admin:hash -- "password"` | Generate `ADMIN_PASSWORD_HASH` |

## Structure

```text
app/
  (site)/            public pages: /, about, how-it-works, services, contact, privacy, terms
  admin/             protected lead dashboard: /admin, /admin/leads, /admin/leads/[id]
  api/leads/         contact form endpoint (JSON from the form, or a plain form post before JavaScript loads)
  api/health/        uptime check: is the form able to accept leads?
  error.tsx, global-error.tsx, not-found.tsx
  sitemap.ts, robots.ts, manifest.ts, opengraph-image.tsx
components/          hero, sections (incl. team), process, contact, analytics, admin, layout, navigation, footer, ui
lib/
  site.ts            company details (single source of truth)
  content.ts         shared marketing copy
  validation/lead.ts form schema — shared by the form and the API
  contact/submit.ts  submitContactForm(): the form's only link to the backend
  analytics.ts       track(): provider-agnostic conversion events (no script loaded)
  attribution.ts     landing page + referrer host, saved with each lead
  seo.ts             page metadata + JSON-LD
  leads/             lead store: Postgres, or a JSON file in development
  email/             team notification, enquirer acknowledgement, provider (SMTP / console)
  auth/              admin session (jose JWT in an httpOnly cookie)
proxy.ts             optimistic /admin gate (pages, actions and route handlers re-check)
db/schema.sql        leads (incl. pipeline fields), notes, admins, activity log, login failures, site content
tests/, scripts/smoke.mjs, .github/workflows/ci.yml   unit tests, smoke test, CI
vercel.json          runs server functions in Washington (iad1), next to the Neon database (us-east-1)
```

## How a request flows

Contact form → client validation → `POST /api/leads` → same-origin check, rate limit (per IP, in memory and
in the database), honeypot and minimum-fill-time checks → server validation → **stored** → **email
notification** to `CONTACT_EMAIL` (defaults to dsdevsyndicate@gmail.com), with Reply-To set to the requester →
**acknowledgement email** to the enquirer.

- The fill-time check uses `fill_ms`, measured in the browser from page load to submit, so a visitor's wrong
  clock can't cause a rejection. The honeypot field is `hp_x7k`, a name browser autofill doesn't recognise.
- If the lead is stored, both emails are sent after the response. The result of the team notification is saved
  on the lead (`notify_status`), and failures show up in the admin. If storage isn't configured, the
  notification is the record and must succeed, or the visitor sees an error with direct contact details.
- The acknowledgement repeats nothing the visitor typed (only the enquiry type, next steps and contact details),
  so the form can't be used to send arbitrary text to other people.
- Before JavaScript loads, the form still works as a normal form post. The API answers with a 303 redirect to
  `/contact/sent`, `/contact/invalid` or `/contact/error`. These are prerendered pages that show the
  outcome without JavaScript, and no personal data ever goes into a URL.
- Each lead records where it came from in `source`: the form page, the landing page (with only `utm_*`, `ref`,
  `type` and `format` query parameters kept) and the referring site's hostname. It is shown in the admin and
  the notification email.

| | Development | Production |
| --- | --- | --- |
| Storage | `.data/leads.json` unless `DATABASE_URL` is set | Postgres via `DATABASE_URL` |
| Email | printed to the server log unless `SMTP_HOST` is set | SMTP via `SMTP_*` |

Configure **both** `DATABASE_URL` and `SMTP_*` in production. With storage alone, leads arrive silently and
nobody is notified. `GET /api/health` reports `{status, store, mailer, database}` and returns 503 when the form
can't accept leads, so point an uptime monitor at it.

The domain already uses Zoho Mail, so send from a domain mailbox (for example `website@devsyndicate.co.in`
via `smtp.zoho.in:465`) and set `SMTP_FROM`. Add a DMARC record for the domain as well.

## Contact form

`components/contact/lead-form.tsx` talks to the backend only through `submitContactForm()` in
`lib/contact/submit.ts`, which posts to `POST /api/leads` and returns a typed result. To move to another
provider, change that one function; the form doesn't need to change.

- Required: organisation type (college or university, company, or other organisation), full name, the
  organisation's name (its label follows the type: "College or university", "Company name" or "Organisation
  name"), work or institutional email, phone, enquiry type
  (Campus Hackathon, National Hackathon, Industry Challenge, Sponsorship or Partnership, General Enquiry).
  Designation, city, participants, format, timeline and additional requirements are optional.
- The form only shows success after the server confirms the enquiry was stored or emailed. Otherwise it keeps the
  visitor's details and offers the phone number and email instead.
- `?type=<enquiry type>` or `?format=<format code>` on `/contact` pre-selects the enquiry type.
  `?type=sponsorship` also pre-selects "Company" as the organisation type.
- The schema still accepts the older optional fields (state, website, theme) and earlier enquiry types, so
  leads that are already stored keep displaying correctly in the admin.

## Analytics

No analytics script is loaded. `track()` in `lib/analytics.ts` is the integration boundary. A single delegated
listener (`components/analytics/click-tracker.tsx`) sends:

| Event | When |
| --- | --- |
| `contact_cta_click` | a link to `/contact` from another page |
| `phone_click` / `email_click` / `linkedin_click` | `tel:`, `mailto:` and LinkedIn links |
| `contact_form_start` | first interaction with the form |
| `contact_form_submit_success` | the server confirmed the enquiry, not on button click |
| `contact_form_submit_error` | validation, server or network failure (reason only) |

Events carry the page path and where the click happened. They never include names, emails, phone numbers or form
contents. To turn analytics on, add the provider's snippet (GA4 `gtag`, Google Tag Manager `dataLayer`, or Plausible)
to `app/layout.tsx`, and `track()` forwards to it. Add the provider's domains to the Content-Security-Policy in
`next.config.ts` (`script-src`, `connect-src`), or the browser will block it. Before you do, update the
"Cookies and tracking" section of `/privacy`, and add a consent banner if the provider sets cookies.

## Search engines

- `/sitemap.xml` and `/robots.txt` are generated from `NEXT_PUBLIC_SITE_URL`, which defaults to
  `https://www.devsyndicate.co.in`. Set it in production if the canonical host differs, and make the other host
  (with or without `www`) redirect to it.
- Each page sets a unique title, description, canonical URL and social tags through `pageMetadata()` in
  `lib/seo.ts`. JSON-LD covers Organization (with founders), WebSite, WebPage/AboutPage/ContactPage and
  BreadcrumbList, plus FAQPage on `/` and Service on `/services`.
- Google Search Console: add a URL-prefix property for the canonical URL, choose "HTML tag", put the token in
  `GOOGLE_SITE_VERIFICATION`, deploy, verify, then submit `https://www.devsyndicate.co.in/sitemap.xml`.
- Bing Webmaster Tools: use "Import from Google Search Console", or put the meta-tag token in
  `BING_SITE_VERIFICATION`, deploy, verify, then submit the sitemap.
- The old site's URLs (`/faq`, `/blog`, `/hackathons`, `/workshops`, `/coding-events`, `/features`, `/tech-stack`,
  `/success-stories`, `/resources`, `/profile`, `/event/:id`, `/cookies`, `/cancellations`) redirect
  permanently to their nearest new page. See `redirects()` in `next.config.ts`.
- In Vercel → Domains, make `devsyndicate.co.in` redirect to `www` with a permanent (308) redirect.

## Admin

The admin lives at `/admin` (sign in at `/admin/login`). Admin pages are `noindex` and disallowed in robots.txt.

### Signing in

| Variable | |
| --- | --- |
| `SESSION_SECRET` | 32+ random characters (`openssl rand -base64 48`). Signs the session cookie. Changing it signs everyone out |
| `ADMIN_USERNAME` | The primary admin's username (case-insensitive) |
| `ADMIN_PASSWORD_HASH` | The primary admin's password, hashed with `npm run admin:hash -- "a long password"` |

- **Primary admin:** `ADMIN_USERNAME` + the password behind `ADMIN_PASSWORD_HASH`. It can always sign in once all
  three variables are set; it isn't a bootstrap mode that switches off later. Its password is changed on the server
  (a new hash), not in the admin. `ADMIN_PASSWORD` (plain text) still works as a fallback for the hash, but don't use
  it in production. `ADMIN_ALLOW_OWNER_LOGIN` is no longer read and can be removed.
- **Additional admins:** named accounts created under Team sign in with their **email** in the Username field and
  their own password. They get a temporary password once and change it under Account.
- Sign-in shows one generic error ("That username and password don't match.") whatever was wrong, and keeps the
  typed username. After 10 failed attempts from one IP in 15 minutes, sign-in is refused for the rest of that window
  (counted per server instance and in the database); a successful sign-in clears the count.
- Sessions last 8 hours. Every admin page, server action and route handler re-checks the session on the server
  (`requireAdmin()`), not only `proxy.ts`. A session ends on its next request when its account is disabled or removed,
  when its password is changed or reset (other sessions of that account; changing your own keeps the current one),
  or, for the primary admin, when `ADMIN_USERNAME` or `ADMIN_PASSWORD_HASH` changes.
- Without a primary admin configured, the last active named account can't be disabled, so there is always a way in.

### Leads

Leads can be searched, filtered (status, enquiry and organisation type, dates, organisation, failed notifications,
open, follow-ups due) and sorted, moved through the pipeline
(New → Contacted → Meeting Scheduled → Proposal Sent → Negotiation → Won / Lost) with a next follow-up date, last
contacted date, meeting time, proposal details and loss reason, and annotated with notes. Follow-up and contact dates
are days in Asia/Kolkata.

- **Privacy requests:** each lead's contact details can be edited (correction) or the lead deleted permanently,
  together with its notes. The activity log keeps who changed what, with the institution's name, after a deletion.
- **Failed notifications:** leads whose team email failed show an "Email failed" badge, and the dashboard warns
  about them. It also warns when no email provider is configured.
- **Deep links:** the "Open in admin" link in a notification email opens that lead after you sign in, because
  login returns you to the page you asked for (otherwise it opens `/admin`).

### Deploying an upgrade

Run `npm run db:setup` against the production database **before** deploying a version that adds columns or tables,
for example with `DATABASE_URL` set in your shell for that one command. It only adds what's missing
(`create … if not exists`, `add column if not exists`) and never drops data, so it's safe to re-run. This version adds
the lead pipeline columns (`next_follow_up`, `last_contacted`, `meeting_at`, `proposal_details`, `loss_reason`), an
activity index, and the `site_content` and `site_content_revisions` tables; earlier versions added `notify_status`,
`org_type`, and the `admins`, `admin_activity` and `login_failures` tables. If the code is deployed first, new leads
are still stored, but admin pages that use the new columns fail until the command has run.

Then set `ADMIN_USERNAME` next to the existing `ADMIN_PASSWORD_HASH` and `SESSION_SECRET` in the host's environment
(the primary admin can't sign in without it) and redeploy. Sessions from before this version are signed out once.

## Editing content

- Company details: `lib/site.ts`, including `site.primaryContact` (the person calls are routed to) and `team`
  (founders' names, roles and LinkedIn URLs). Add real social profiles to `site.social` and the footer shows them.
- Copy: `lib/content.ts`.
- **Outcome metrics** (`outcomeMetrics`) have `value: null` until there are verified, cumulative numbers.
  Set a value and the homepage animates it in. Don't publish estimates.
- There is no testimonials, partner-logo or events section by design. Add them only with real, permissioned content.
- The team section is typography-led because no photographs have been supplied. Add photos only if the founders
  provide and approve them.

## Motion

- **Ambient loops:** the hero schematic and the ecosystem path are wrapped in `[data-ambient]`.
  `components/layout/reveal-observer.tsx` pauses them while they are off screen or the tab is hidden.
- **Pause control:** "Pause animations" (`components/ui/motion-toggle.tsx`), in the hero caption bar and the
  footer, stops them site-wide. It also stops the experience slideshow. The choice is saved in `localStorage`
  (`ds-motion`) and applied before first paint through `html[data-motion="paused"]`.
- **Experience slideshow:** it auto-advances, pauses on hover, and stays paused once a visitor picks a tab. It has
  its own play/pause button.
- **Lifecycle spotlight:** inactive stages are dimmed by colour, not opacity, so every stage stays readable
  (4.5:1 or better).
- **Reduced motion:** with `prefers-reduced-motion`, everything is static.
- **Printing:** print styles show all content, including sections that haven't been revealed by scrolling.
