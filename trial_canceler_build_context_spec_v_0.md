# Trial Canceler – Build Context & Spec (v0.1)

A single source of truth to generate comparable MVPs across Bubble, Base44, same.new, or a conventional code stack. Keep this file in the repo root as `CONTEXT.md`.

---

## 1) Mission
Help users avoid surprise charges by tracking free trials, reminding them before renewal, and escalating notifications until cancellation is verified or the user chooses to keep.

**Primary KPI:** prevented charges (self-reported) and on-time cancellations.

**Secondary KPIs:** time-to-first-trial-added, notification delivery rate, parsing accuracy.

---

## 2) Scope
**In-scope MVP**
- Add trials via: pasted email text, connected inbox (optional for MVP), or manual form.
- Countdown and scheduled notifications (email required; SMS/push optional).
- Escalation in last 24h with explicit user consent.
- Evidence tracking and stop conditions.
- Activity/audit log per trial.

**Out-of-scope MVP**
- Card scraping, bank connections, screen-scraping cancellations.
- Vendor-side automation beyond sending a plain email using a template (optional).

---

## 3) Personas & User Flows
**Persona:** Busy consumer who signs up for multiple trials. Wants set-and-forget reminders.

**Core flow:**
1. User adds a trial (paste email → parse → confirm fields → save).
2. System schedules notifications relative to end date.
3. User receives reminders and clicks “Cancel now” or “Keep.”
4. If cancel: user triggers cancellation or uploads proof. System watches inbox for confirmation (when connected) or accepts manual proof. Notifications stop.

---

## 4) Non-Goals (MVP)
- Automated logins to vendor portals.
- Financial account linking.
- AI-driven vendor-specific cancellation steps.

---

## 5) Data Model
```
User(id, email, phone?, notif_prefs.json, aggressive_mode:boolean, created_at)
Trial(id, user_id→User.id, service_name, plan, price_after:decimal, start_date:date, end_date:date, cancel_url, support_email, status:enum[active,cancelled,kept,expired], source:enum[manual,email,api], created_at, updated_at)
Notification(id, trial_id→Trial.id, channel:enum[email,sms,push], scheduled_at, sent_at?, status:enum[pending,sent,cancelled,failed], provider_msg_id?, meta.json)
Evidence(id, trial_id→Trial.id, type:enum[inbound_email,outbound_email,screenshot,webhook], detected_at, summary, blob_ref, verification_passed:boolean)
Event(id, trial_id→Trial.id, kind:enum[created,schedule_updated,user_keep,user_cancel,auto_email_sent,verification_passed], ts, meta.json)
```

**Indices:**
- `Trial(user_id, end_date)` for lookups.
- `Notification(trial_id, scheduled_at)`.
- `Evidence(trial_id, detected_at)`.

---

## 6) API Contract (reference)

### Auth
- Session cookie or JWT. Minimal email link sign-in allowed for MVP.

### REST Endpoints
- `POST /api/trials/ingest` → `{ raw_email?:string, manual?:{...} }` → returns parsed draft `{service_name,end_date,cancel_url,price_after}`.
- `POST /api/trials` → create trial from confirmed fields.
- `GET /api/trials` → list trials for user.
- `GET /api/trials/:id` → trial detail with upcoming notifications and evidence.
- `POST /api/trials/:id/cancel-intent` → logs user intent; optionally sends cancellation email; schedules verification check.
- `POST /api/trials/:id/keep` → sets status `kept` and next review date.
- `POST /api/notifications/dispatch` (internal, queued) → idempotent send.
- `POST /api/evidence/inbound` → email/webhook/screenshot evidence.
- `GET /api/export` → full JSON export for user.

**Idempotency:** all mutation endpoints accept `Idempotency-Key` header.

---

## 7) Notification Schedule Engine
Given `end_date`:
- Base schedule: `[-7d, -3d, -1d, -12h, -2h, -30m]`.
- If `aggressive_mode` and < 12h remaining with no user action: add `[-4h, -1h, -15m, -5m]`.
- Stop when `Trial.status in {cancelled, kept}` or any `Evidence.verification_passed = true`.

**Time safety:**
- Store all timestamps in UTC; render in user’s timezone.
- On DST changes, use absolute UTC instants to avoid duplicate sends.

---

## 8) Parsing Rules (MVP)
- Inputs: `raw_email` (RFC822 or plaintext) or manual fields.
- Extractors:
  - **Service name:** from `From:` domain, subject tokens, or first prominent vendor-like proper noun.
  - **End date:** patterns: `trial ends on`, `renews on`, dates like `Aug 30, 2025`, `30/08/2025`, relative phrases `in 7 days`.
  - **Cancel URL:** nearest `https` around tokens {cancel, manage, billing, subscription}.
  - **Price after:** currency + number near `charge`, `billed`, `after trial`.
- Confidence score; if < threshold, flag for manual confirmation.

---

## 9) Verification Heuristics
- Inbound email subjects matching regex set: `cancel(l)?ed|subscription (ended|cancelled)|confirmation|we're sorry to see you go` within ±24h of cancel intent.
- Outbound copy archived if we triggered email.
- Manual override: user upload; reviewer can mark `verification_passed`.

---

## 10) UX Surfaces (Minimal)
- **Trial List:** filter chips `Active | Cancels Soon | Cancelled | Kept`, search by vendor.
- **Add Trial:** tabs `Paste Email | Connect Inbox | Manual` with preview diff of parsed fields.
- **Trial Detail:** countdown, next reminders, actions `Cancel now | Keep`, evidence timeline.
- **Settings:** channels toggles, aggressive mode opt-in with clear frequency text, export/delete account.
- **Activity Log:** per trial chronological events.

Accessibility: keyboard nav, ARIA for live countdowns, color-contrast AA.

---

## 11) Acceptance Criteria
- Add a manual trial in ≤ 60 seconds.
- Schedule generates all baseline notifications at correct UTC instants for a sample of 20 trials across time zones.
- Parsing test set ≥ 85% exact end_date, ≥ 90% service_name, ≥ 75% cancel_url.
- Evidence upload stops future notifications within 60 seconds.
- Export returns a complete, valid JSON document of the user’s data.

---

## 12) Test Fixtures
Include in `/fixtures/emails/` and `/fixtures/expected.csv`.
- 20 samples with variations: US/EU dates, no explicit cancel URL, obfuscated `canceI` (capital i), different currencies, marketing fluff.
- `expected.csv` columns: `fixture,service_name,end_date_iso,cancel_url,price_after_minor_units`.

---

## 13) Observability & Logging
- Correlation id per request; include in Notification metadata.
- Structured logs for sends and schedule changes.
- Minimal admin view: search trial by email, view delivery events and evidence.

---

## 14) Security & Privacy
- Data at rest encryption (managed by platform where applicable).
- No storing email inbox contents beyond parsed fields and minimal evidence; redact PII in logs.
- Consent text for aggressive mode; one-click disable all notifications.
- GDPR-style export/delete flows.

---

## 15) Platform Notes
**Bubble:** backend workflows for schedules; store returned schedule ids; use Mailparser/Resend inbound for `POST /api/evidence/inbound` equivalent. OAuth for Gmail/Graph is a plus.

**Base44 / same.new / codegen:** generate: schema, REST endpoints above, queue worker for notifications, and a minimal React/PWA. Provide a Dockerfile for portability.

**Conventional stack:**
- API: FastAPI/Express.
- Queue: Postgres + cron worker or lightweight queue.
- Mail: Resend/SES.
- SMS/Push: Twilio/OneSignal.
- DB: Postgres with Prisma/Drizzle.

---

## 16) Project Structure (reference)
```
/fixtures
  /emails
  expected.csv
/packages (optional monorepo)
  /app (web/PWA)
  /api (REST + workers)
/docs
  CONTEXT.md (this file)
  RUBRIC.md
/scripts
  seed-fixtures.ts
  run-parsing-tests.ts

.env.example
Dockerfile
Makefile
```

---

## 17) Env Vars (example)
```
APP_BASE_URL=
JWT_SECRET=
RESEND_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
ONESIGNAL_APP_ID=
ONESIGNAL_API_KEY=
GMAIL_OAUTH_CLIENT_ID=
GMAIL_OAUTH_CLIENT_SECRET=
GRAPH_CLIENT_ID=
GRAPH_CLIENT_SECRET=
```

---

## 18) Contribution Workflow
- Branch naming: `feat/…`, `fix/…`, `chore/…`.
- Conventional commits for readable history.
- PR template must include: scope touched, tests added/updated, screenshots of schedule table for a sample trial.
- CI checks: parse tests pass on `/fixtures`, lint, typecheck.

---

## 19) Bake-off Rubric Summary (mirror of README)
- Build speed (20)
- Parsing accuracy (15)
- Notification reliability (15)
- UX clarity (10)
- Integrations (10)
- Observability (10)
- Cost & limits (10)
- Lock-in & portability (10)

Scoring sheet in `/docs/RUBRIC.md`.

---

## 20) Monetization Notes
- Free: 5 active trials, email only.
- Pro: unlimited + SMS/push + inbox + export.
- Family/Team: shared dashboard and delegated cancel.

---

## 21) Roadmap After MVP
- Calendar integration (T-1d event).
- Vendor-specific cancel playbooks.
- Shared household view.
- Import from mailbox search for historical trials.

---

## 22) License & Attribution
- MIT for app code; sample emails are synthetic.

---

**End of spec.** Keep this document terse and exact. If you change anything that affects data shape or endpoints, bump the version and update `/docs/RUBRIC.md`. 

