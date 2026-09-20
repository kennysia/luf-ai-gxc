# 04 - Implementation Notes

Technical detail behind `03-proposal.md`. This is the build sheet for phase 0 and 1.

## Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Language | Python 3.11+ | Best fit for n8n custom nodes, PerfectGym API clients, Playwright fallback |
| LLM | Claude Opus 5 (`claude-opus-5`) with adaptive thinking; Claude Haiku 4.5 (`claude-haiku-4-5`) optional for bulk message classification | Opus 5 for judgement-heavy drafting and anomaly review; Haiku for cheap high-volume parsing |
| Orchestration | n8n (self-hosted) | Cron triggers, WhatsApp and HTTP nodes, visual audit trail a non-developer can read |
| Messaging | WhatsApp Business Platform via a BSP (Twilio or 360dialog); Telegram Bot API as fallback adapter | Instructors already on WhatsApp; per-conversation pricing is trivial at this volume |
| Data | Google Sheets (phase 1) then Postgres | Kenny and Yen edit rates and roster directly; migrate when a second club joins |
| Booking system | PerfectGym (`levelup.perfectgym.pl`, member portal `members.levelupfitness.com`) | Source of truth for classes, bookings, attendance, waitlists |
| Publishing | PGM API or Playwright; Facebook Graph API page post; Foyer signage upload; timetable image rendered from an HTML template | One-click publish to all three surfaces |

## Claude API usage pattern

Every agent step is a single `messages.create` call with tools, or a short tool-runner loop. No
long-lived agent sessions are needed for phase 1; each cron job is stateless and reads its state
from PGM and the policy sheet.

Pricing reference (Anthropic first-party rates, per million tokens): Opus 5 USD 5 in / 25 out;
Haiku 4.5 USD 1 in / 5 out. A weekly timetable draft with roster, policy and last 4 weeks of KPIs
in context is roughly 15-25K input tokens and 2-4K output. At one draft plus a handful of revisions
per week, plus daily dashboards and message parsing, total model spend stays in the low tens of
USD per month.

Structured outputs (`output_config.format`) are used for every machine-consumed result so the
timetable draft, cover ranking and payroll anomaly list are schema-valid JSON before the validator
and the human see them. Prompt caching is applied to the stable prefix (policy table, roster,
instructions) so weekly runs re-use it.

Server-side fallbacks are enabled on Opus 5 calls so a safety-classifier refusal (unlikely on this
content, but possible) degrades gracefully rather than failing the cron job.

## Schema (phase 1, Google Sheets tabs)

**instructors**: id, name, phone (E.164), email, status (active/paused/offboarded), programmes (list), certifications (programme, body, expiry), availability (weekday, from, to), approved_rate (per class or tier table), free_membership (yes/no, last_reviewed), notes.

**slots** (the standing weekly template): slot_id, weekday, start, duration, studio, programme, default_instructor_id, capacity, active_from, active_to, seasonal_mode (normal/reduced).

**policy**: key, value, effective_from, note. Seeded from `01-source-analysis.md` section 2: draft_day=Wed, publish_deadline=Thu 12:00, booking_open_hours=77, cancel_penalty_hours=2, online_close_hours=1, auto_cancel_max_bookings=2, late_entry_minutes=5, lesmills_min_classes_per_week=3, lesmills_notice_months=3, free_membership_min_classes=2, eligibility_review_months=1,4,7,10, payroll_deadline_day=5, qcc_pass_score=15, cover_search_window_hours=6.

**weeks**: week_start, status (draft/approved/published), approved_by, approved_at, published_at, diff_from_prev.

**classes** (the published instance): class_id, week_start, slot_id, instructor_id, cover_instructor_id, status (scheduled/covered/cancelled), pgm_class_id, bookings, attended, no_shows, late_cancels, waitlist_max.

**qcc_audits**: audit_id, instructor_id, assessor, date, programme, items 1-20 (0/1), score, pass, strengths, improvements, acknowledged.

**messages_log**: ts, direction, instructor_id, channel, body_hash, intent, handled_by (agent/human).

## Cron jobs

| When | Job |
|------|-----|
| Mon 08:00 | KPI dashboard for the prior week to the human owner |
| Mon 10:00 | Availability ask to each instructor for the week after next |
| Tue 18:00 | Chase non-responders |
| Wed 09:00 | Draft timetable, validate, send for approval |
| Thu 12:00 | Publish deadline check; escalate anything unpublished |
| Daily 07:00 | Scan next 48 h for classes with no confirmed instructor; scan for classes at or below auto-cancel line 26 h out |
| 1st of month 06:00 | Payroll pack |
| 1 Jan/Apr/Jul/Oct | Free-membership eligibility review |
| Weekly | Les Mills programme count check |

## Validation rules run on every draft before a human sees it

- Every class references an existing slot and an active instructor qualified for that programme.
- No instructor is double-booked within a slot's duration plus 15 minutes travel/turnaround.
- No studio has overlapping classes.
- Les Mills programmes meet the minimum weekly count, or a warning is attached.
- No instructor without an approved rate appears.
- Seasonal mode matches the calendar (December and the fortnight before CNY use the reduced template unless overridden).
- Diff against the previous published week is attached.

## PerfectGym integration: what phase 0 must establish

1. Whether LUF's PerfectGym contract includes API access (PerfectGym exposes a REST API to clients on request; scope varies by plan).
2. Read endpoints needed: classes by date range, bookings and attendance per class, waitlist, class types (to exclude PT-led classes).
3. Write endpoints needed: create or update class instances (instructor, time, capacity), cancel class with member notification.
4. If write is unavailable: agent produces the CSV/table in PGM's import format and a Playwright script performs the upload under the human's login on their click.

## Security and confidentiality

- Instructor rates and phone numbers are stored only in LUF's Google Workspace / database. Prompts receive instructor IDs and first names plus the minimum fields the task needs; rates enter the prompt only for the payroll job.
- API keys and PGM credentials live in n8n's credential store or environment variables, never in a sheet or a prompt. The Fitbox Virtual credentials written in the 2022 briefing notes should be rotated.
- Every outbound message and every publish action is logged with who approved it.
- Model calls go to the Claude API under Anthropic's commercial data terms (30-day retention on current models). If LUF requires zero retention, confirm eligibility with Anthropic before phase 1.

## Open technical questions for Level Up Fitness

1. PerfectGym API availability and scope on the current plan.
2. Whether the Foyer signage system accepts an image upload via URL or API, or needs a manual upload.
3. Facebook page admin access for the Graph API app.
4. Whether HQ payroll wants a spreadsheet, a PDF, or a PGM export format.
5. Pay model in force today (flat per class vs booking-tiered).
