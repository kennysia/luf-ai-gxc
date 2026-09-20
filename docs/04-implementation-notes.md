# 04 - Implementation Notes

Technical detail behind `03-proposal.md`. This is the build sheet for phase 0 and 1.

## Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Language | Python 3.11+ | Best fit for n8n custom nodes, CSV parsing, Playwright |
| LLM | Claude Opus 5 (`claude-opus-5`) with adaptive thinking; Claude Haiku 4.5 (`claude-haiku-4-5`) optional for bulk message classification | Opus 5 for judgement-heavy drafting and anomaly review; Haiku for cheap high-volume parsing |
| Orchestration | n8n (self-hosted) | Cron triggers, WhatsApp and HTTP nodes, visual audit trail a non-developer can read |
| Messaging | WhatsApp Cloud API (direct, or via a Malaysian BSP such as Wati or SleekFlow); Telegram Bot API as the alternative adapter | Instructors already on WhatsApp; Meta per-message fees are trivial, the BSP fee is not; Telegram is free and has polls and buttons |
| Data | Google Sheets (phase 1) then Postgres | Kenny and Yen edit rates and roster directly; migrate when a second club joins |
| Booking system | Sentinel Fitness by Scope Software Solutions (tenant `levelup.sentinelscope.com`, apps `scope.levelup` and `scope.leveluptrainers`); replaced PerfectGym mid-2025 | Source of truth for classes, bookings, waitlists, attendance, members; no public API |
| Publishing | Sentinel change list applied by the human (portal or Level Up Trainers app) or by Playwright under their login; Facebook Graph API page post; Foyer signage upload; timetable image rendered from an HTML template | One-click publish to Facebook and Foyer; Sentinel write stays human-triggered |

## Claude API usage pattern

Every agent step is a single `messages.create` call with tools, or a short tool-runner loop. No
long-lived agent sessions are needed for phase 1; each cron job is stateless and reads its state
from the Sentinel adapter and the policy sheet.

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

**instructors**: id, name, phone (E.164), email, status (active/paused/offboarded), programmes (list), certifications (programme, body, expiry), availability (weekday, from, to), grade (Uncertified/Rookie/Established/Star/Superstar/Veteran, per the July 2023 scheme), approved_rate (per class at current grade, revised quarterly), free_access (yes/no, period, last_reviewed), notes.

**slots** (the standing weekly template): slot_id, weekday, start, duration, studio, programme, default_instructor_id, capacity, active_from, active_to, seasonal_mode (normal/reduced).

**policy**: key, value, effective_from, source_thread, note. Seeded from `01-source-analysis.md` section 2 and the email register in `05-agreed-gx-policies.md`: draft_send_day=Mon, instructor_change_cutoff=Wed 18:00, publish_deadline=Thu 12:00, booking_open_hours=77, cancel_penalty_hours=2, frontdesk_cancel_hours=1, online_close_hours=1, class_go_min_bookings=3, class_go_check_hours_before=2, cancelled_class_instructor_pay=0, late_entry_grace_minutes=5, member_strikes=3, member_strike_window_weeks=2, member_suspension_days=14, absence_record_retention_weeks=2, grey_capacity_non_equipment=4, grey_capacity_equipment=1, booking_rate_target_pct=70, lesmills_min_classes_per_week=3, lesmills_notice_months=3, trainee_clearance_months=6, free_access_min_classes=2, eligibility_review_months=1,4,7,10, rate_review_months=1,4,7,10, payroll_deadline_day=5, qcc_pass_score=15, mic_company_replace_after_months=12, incentive_budget_per_quarter_rm=300, cover_search_window_hours=6.

**weeks**: week_start, status (draft/approved/published), approved_by, approved_at, published_at, diff_from_prev.

**classes** (the published instance): class_id, week_start, slot_id, instructor_id, cover_instructor_id, status (scheduled/covered/cancelled), sentinel_class_id, bookings, attended, no_shows, late_cancels, waitlist_max.

**publish_changes**: week_start, change_id, action (add/remove/reassign/retime), slot_id, from_value, to_value, applied_by, applied_at. This is the change list the human applies in Sentinel; the agent verifies it by re-reading the next export or portal view.

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

## Sentinel integration: what is known and what phase 0 must establish

Known (vendor feature list and app descriptions, snippet-level; see research doc 7.3):

- Sentinel holds the GX booking calendar, instructor database, member bookings, waiting lists (managed from the Level Up Trainers app), class attendance, class efficiency reports, scheduled and ad hoc reports, and a "data extract for commission purposes".
- The vendor mentions an API and has integration partners (Keepme, FitnessKPI), but publishes no developer documentation. No Zapier, Make or n8n connector exists.
- Notifications available inside Sentinel: email and SMS campaigns and alerts.

The Sentinel adapter is written against an internal interface (`list_classes(week)`, `list_bookings(class_id)`, `list_attendance(class_id)`, `list_instructors()`) with three interchangeable back-ends:

| Back-end | Requires | Freshness | Effort |
|----------|----------|-----------|--------|
| A. Partner API | Scope grants credentials and a spec | Real time | 1 week once spec is in hand |
| B. Scheduled report exports | Sentinel scheduled reports emailed nightly as CSV or Excel to an agent mailbox; the agent parses them | Nightly (enough for W1, W3, W4, W5, W6; W2 cover routing uses the roster sheet plus the latest export) | 1 week, no vendor dependency |
| C. Browser session | Staff login; Playwright reads the portal calendar and booking lists | On demand | 1-2 weeks, brittle to UI changes |

Phase 0 must establish:

1. Scope's answer to a written API or export request (ask specifically for: class list with instructor and capacity, bookings and waitlist per class, attendance per class, instructor list, and whether class creation or instructor assignment is exposed).
2. Which scheduled reports Sentinel can already email, in what format, and whether they carry instructor and attended count per class. If yes, back-end B is the pilot path regardless of the API answer.
3. Whether the Level Up Trainers app or the portal is where the Club Manager prefers to apply the weekly change list.
4. Whether the human owner is comfortable with a Playwright script applying the change list under their login, or prefers to apply it by hand (about 5-10 minutes a week).
5. Whether `levelup.perfectgym.pl` still holds historical GX attendance (2015 to mid-2025). If so, a one-off export gives the KPI dashboard a multi-year baseline from day one.

## Security and confidentiality

- Instructor rates and phone numbers are stored only in LUF's Google Workspace / database. Prompts receive instructor IDs and first names plus the minimum fields the task needs; rates enter the prompt only for the payroll job.
- API keys and Sentinel credentials live in n8n's credential store or environment variables, never in a sheet or a prompt. The Fitbox Virtual credentials written in the 2022 briefing notes should be rotated.
- Every outbound message and every publish action is logged with who approved it.
- Model calls go to the Claude API under Anthropic's commercial data terms (30-day retention on current models). If LUF requires zero retention, confirm eligibility with Anthropic before phase 1.

## Open technical questions for Level Up Fitness

1. Scope's response to the API or export request, and a sample of any Sentinel report LUF already receives.
2. Whether the Foyer signage system accepts an image upload via URL or API, or needs a manual upload.
3. Facebook page admin access for the Graph API app.
4. Whether HQ payroll wants a spreadsheet, a PDF, or a Sentinel export format.
5. Pay model in force today (flat per class vs booking-tiered).
6. WhatsApp: does LUF already have a verified Meta Business account and a spare number? If not, Telegram is the faster pilot channel.
