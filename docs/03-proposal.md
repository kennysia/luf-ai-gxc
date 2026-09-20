# 03 - Proposal: AI Virtual GX Coordinator ("Coach Ops") for Level Up Fitness

## 0. The honest premise check

Before proposing anything, the idea itself needs stress-testing. "An AI that performs the job of a GX
Coordinator" is only half true, and building as if it were fully true is how these projects fail.

What the documents say the job is, split by whether software can own it:

| Software can own it | Software can assist, a human must own it |
|---------------------|------------------------------------------|
| Collect availability, draft the weekly timetable, chase confirmations, publish to PGM / Foyer / Facebook | Deciding to drop an instructor, a programme or a Les Mills licence |
| Detect an absence, rank qualified covers, broadcast the ask, notify booked members | Persuading a reluctant instructor to cover at short notice |
| Compile payroll from attendance and rate tables, flag anomalies, produce the report by the 5th | Approving payroll and signing off rate changes (Kenny and Yen) |
| Track fill rate, attendance, no-shows, late cancels, Les Mills minimums; recommend slot changes | Sitting in the management meeting and owning the P&L decision |
| Schedule QCC audits, digitise scores, track pass/fail history, draft feedback | Standing in the studio and observing a class (item 9-14 of the QCC cannot be judged from a booking system) |
| Remind on policy: mic SOP, free-membership eligibility review, booking rules | Floor presence, member conversations, coaching instructors, resolving conflict |

So the correct framing is: **an AI agent that removes the administrative 70-80% of the role, with a
named human (Club Manager or a reduced-hours coordinator) retaining judgement, presence and approval.**
The RM 800/month contractor fee is modest; the value case is not headcount saving but consistency
(the timetable never slips), speed (cover found in minutes not hours), and data (decisions on slots
and instructors backed by numbers instead of impressions). If the goal were purely to eliminate the
RM 800, the project would not pay for its own build. If the goal is a reliable, scalable GX
operation across City Mall KK and future outlets, it does.

## 1. Design principles

1. **PerfectGym is the source of truth.** Classes, bookings, attendance, waitlists and members already live there. The agent reads from it and writes the published timetable to it. No parallel spreadsheet of bookings.
2. **One-to-one messaging, never a group.** Kenny's 2022 note against a WhatsApp group is right. The agent talks to each instructor individually via WhatsApp (or Telegram as a cheaper fallback). Adding or removing an instructor is invisible to the others.
3. **Human-in-the-loop on every irreversible action.** Publishing a timetable, confirming a cover, sending a member broadcast, submitting payroll: each is a one-tap approval by the human owner. The agent prepares, the human releases.
4. **Policy as data, not prose.** Every rule in `01-source-analysis.md` section 2 lives in a versioned policy file the agent reasons over. Change the rule, not the code.
5. **Confidentiality by construction.** Instructor rates and contact details are stored in LUF-controlled systems (Google Workspace or a small database) and only the minimum needed for a task enters a model prompt. No credentials in prompts. No data to third-party vendors beyond the model API and the messaging provider, both under data-processing terms.
6. **Start narrow, prove the loop, then widen.** Weekly timetable first, because it is the highest-frequency pain and the easiest to measure.

## 2. Target architecture

```
                      +---------------------------+
   Instructors  <---> |  Messaging layer          | <---> Human owner (Club Manager / Kenny)
   (WhatsApp 1:1)     |  WhatsApp Business API    |       (WhatsApp + web approval page)
                      |  or Telegram Bot          |
                      +------------+--------------+
                                   |
                      +------------v--------------+
                      |  Orchestrator             |   scheduled jobs (cron):
                      |  (n8n or a small Python   |   Mon availability ask, Wed draft,
                      |   service) running        |   Thu publish, daily KPI, monthly payroll
                      |   Claude agent workflows  |
                      +---+----------+---------+--+
                          |          |         |
        +-----------------v-+  +-----v------+  +v-------------------+
        | PerfectGym API    |  | Policy +   |  | Publishing adapters |
        | classes, bookings |  | instructor |  | PGM timetable       |
        | attendance,       |  | roster DB  |  | Foyer signage       |
        | waitlist, members |  | (Sheets or |  | Facebook page post  |
        +-------------------+  |  Postgres) |  | Canva timetable img |
                               +------------+  +---------------------+
                                     |
                        +------------v-------------+
                        | Reporting                 |
                        | Weekly GX dashboard,      |
                        | monthly payroll pack,     |
                        | QCC audit log             |
                        +---------------------------+
```

**Model.** Claude via the Claude API. Claude Opus 5 (`claude-opus-5`) for the reasoning steps:
schedule drafting, cover ranking, payroll anomaly review, dashboard commentary. Claude Haiku 4.5
(`claude-haiku-4-5`) as an optional cheaper worker for high-volume, low-stakes message parsing
(classifying an instructor's "YES" or "swap me to 7pm"). Tool use against the PerfectGym API and
the policy store, with structured outputs so the timetable draft is always valid JSON that the
validator can check. See `docs/04-implementation-notes.md` for model IDs, pricing and integration
detail.

**Orchestration.** n8n (self-hosted, low cost, visual, easy for a non-developer to inspect) or a
small Python service. Recommendation: n8n for phase 1 to get running fast, with the option to move
the scheduling brain into a dedicated service if logic outgrows it.

**Messaging.** WhatsApp Business Platform is what instructors already use in Malaysia. Cost is
per-conversation and small at this scale (roughly 15-30 instructors, a few conversations each per
week). Telegram is free and simpler to build against but forces instructors onto a second app.
Recommendation: WhatsApp via a BSP such as Twilio or 360dialog; keep the adapter thin so Telegram
can be swapped in.

**Data store.** Google Sheets is acceptable for phase 1 (instructor roster, rates, availability,
qualifications, policy table) because Kenny and Yen can edit it directly. Move to Postgres when a
second outlet comes on.

**Publishing.** PGM timetable upload is the critical adapter. If the PerfectGym API supports class
creation it is a direct write. If it does not, the fallback is a generated CSV plus a one-click
browser automation (Playwright) that the human triggers, or a manual upload from the agent's
prepared file. Foyer signage and Facebook take a generated timetable image; the agent renders it
from a template (Canva API or HTML-to-image) so branding stays consistent.

## 3. The workflows, one by one

### W1. Weekly timetable (the core loop)

| Day | Agent action | Human action |
|-----|--------------|--------------|
| Mon 10:00 | Message each instructor 1:1: "Here is your usual slot(s) for week of DD/MM. Reply YES, or tell me changes." Pre-fills from the standing roster. | none |
| Tue 18:00 | Chase non-responders once. Parse free-text replies (swaps, unavailability, requests). | none |
| Wed 09:00 | Draft the timetable: standing roster + confirmed changes, checked against policy (Les Mills minimums, no clashes, studio capacity, seasonal mode, no new instructor without approved rate). Attach a delta list ("2 changes from last week") and any flags ("BodyBalance drops to 2 classes; licence minimum is 3"). | Review draft on a web page or in WhatsApp; approve, or edit and approve |
| Wed-Thu | Send each instructor their confirmed slots; collect final confirmations. | none |
| Thu 12:00 | Publish: write to PGM, render timetable image, post to Facebook, push to Foyer. Confirm back to the human with links. | One-tap "publish" if any late change came in after approval |
| Thu 12:30 | If any class is unconfirmed by an instructor, escalate to the human with cover candidates already ranked. | Decide |

Measured by: on-time publish rate (target 100%), confirmation latency, human minutes per week.

### W2. Cover and substitution

Trigger: instructor messages "can't make Thursday 7pm", or the human forwards a message, or a class
in PGM has no instructor assigned within 48 h.

1. Agent identifies the class, programme and time.
2. Ranks candidate covers from the roster: qualified for that programme (certification on file), not already teaching in that slot, historically reliable, within approved rate.
3. Broadcasts a 1:1 ask to the top 3 candidates in parallel, first to accept wins; escalates to the next 3 after 30 minutes.
4. On acceptance: updates PGM instructor field, tells the original instructor, tells the Club Manager, and drafts the member notification ("Your BodyPump Thu 7pm will be taught by X") for release.
5. If no cover within the policy window (say 6 hours before class): recommends cancel-and-notify with the booked-member list, and the human decides.

Measured by: time-to-cover, classes cancelled for lack of instructor, member notifications sent before class.

### W3. Monthly payroll

1. On the 1st: pull all classes taught in the prior month from PGM with instructor, programme, headcount, attended count.
2. Apply the rate table (flat per class, or booking-tiered if that model is live). Add the coordinator's own fee line.
3. Anomaly flags: class taught but not on published timetable, instructor paid for a class with zero attendance, rate differs from approved rate, cover taught but original instructor still listed.
4. Produce the payroll pack (spreadsheet + summary) by the 3rd; human reviews and submits to HQ Admin by the 5th.
5. Handle verification questions from HQ by pointing at the underlying PGM records.

Measured by: on-time submission, number of corrections after submission.

### W4. Performance monitoring and slot optimisation

Weekly dashboard, sent Monday morning before the availability ask so the human can act on it:

- Per class and per instructor: bookings, attendance, fill rate (attended / capacity), no-show rate, late-cancel rate, waitlist depth, trailing 4-week trend.
- Per programme: classes per week vs licence minimum (Les Mills), cost per attendee.
- Watchlist: any slot below the "2 or fewer bookings" auto-cancel line twice in 4 weeks; any slot consistently waitlisted (a case for a second class); any instructor whose fill rate is falling.
- Recommendations phrased as decisions for the human: "Retire Tue 6am Zumba (avg 2.1 attendees, 4 weeks) and reuse the slot for a second Thu BodyPump (waitlist avg 6)."

This is the piece that directly serves the JD's "optimise bookings and attendance" and Kenny's
rule "retire an underperforming slot before adding a new one".

### W5. Instructor quality (QCC) support

- Maintain the audit calendar: every active instructor audited at least twice a year, new instructors at weeks 2 and 8.
- Digitise the 20-point QCC as a mobile form the assessor fills in the studio; the agent stores it, computes the score, tracks history, and drafts the feedback message in the constructive tone the JD asks for.
- On a fail (below 15), schedule the re-audit and a mentor pairing automatically.
- Items 1-5 and 15-20 (pre and post class logistics) can be partly evidenced from data: class started on time can be checked against PGM check-ins; the rest still needs the assessor.

The observation itself stays human. The paperwork, scheduling and follow-through become automatic.

### W6. Policy compliance and reminders

- Quarterly (Jan/Apr/Jul/Oct): compute each instructor's regular classes per week and flag free-membership eligibility changes for the human to action in PGM.
- Weekly: check Les Mills programme counts against the 3-class minimum; warn 3 months ahead of any licence renewal decision.
- Onboarding: send the mic SOP, booking policy and studio SOP to every new instructor and record acknowledgement.
- Seasonal mode: switch to the reduced December / pre-CNY template on a configurable date and post the "temporary schedule" notice.

### W7. Liaison and meeting prep

- Collect instructor feedback on studio, equipment and members via a monthly 1:1 prompt; classify and summarise into the management meeting pack.
- Prepare the weekly management meeting agenda from the dashboard, cover incidents, and open issues.
- Event support: for a launch or themed class, produce the checklist (timetable slot, PGM class creation, Facebook post, signage, instructor briefing) and track it.

## 4. Phased delivery

| Phase | Weeks | Scope | Exit criterion |
|-------|-------|-------|----------------|
| 0. Foundations | 1-2 | PerfectGym API access confirmed; instructor roster, rates, qualifications and policy table loaded; WhatsApp Business number provisioned; human owner named | Agent can read this week's PGM timetable and message one test instructor |
| 1. Timetable loop | 3-6 | W1 end to end, with human approval. Publishing to PGM automated or semi-automated; Facebook and Foyer via generated image | 4 consecutive weeks published by Thursday with under 30 min of human time per week |
| 2. Cover + dashboard | 7-10 | W2 and W4 | Median time-to-cover under 2 h; dashboard used in the management meeting |
| 3. Payroll | 11-13 | W3 | Two consecutive months submitted by the 5th with zero post-submission corrections |
| 4. Quality + compliance | 14-18 | W5, W6, W7 | Every active instructor has a QCC record; quarterly eligibility review runs automatically |
| 5. Second outlet | later | Multi-club support, per-club policy | Same agent runs two clubs with one human owner |

## 5. Costs (order of magnitude)

| Item | Estimate |
|------|----------|
| Claude API usage | Low tens of USD per month at this message volume |
| WhatsApp Business Platform via a BSP | Under USD 30/month for this conversation volume, plus number setup |
| n8n self-hosted on a small VPS | USD 5-15/month |
| Build effort | Phase 0-1 roughly 3-4 weeks of one developer; phases 2-4 another 6-8 weeks |
| Ongoing human owner | 2-4 h/week for approvals, cover decisions and audits (down from 8-12 h) |

The biggest cost driver is unknown until phase 0: whether PerfectGym exposes a usable API for
class creation and booking reads on LUF's plan. If it does not, budget an extra 1-2 weeks for a
browser-automation adapter and accept that publishing stays semi-manual.

## 6. Risks and how the design handles them

| Risk | Mitigation |
|------|------------|
| PerfectGym API unavailable or read-only | Phase 0 check before any other build; CSV + Playwright fallback |
| Instructors ignore a bot | Messages come from a named LUF number with a human name on it; the human owner steps in after two ignored chases; instructors learn the bot is the fastest way to get paid correctly |
| Agent publishes a wrong timetable | Nothing publishes without human approval; every publish has a diff against last week; one-click rollback to the previous PGM state |
| Confidentiality breach | Rates and phone numbers never leave LUF systems except to the model API under Anthropic's data terms; no rate is ever shown to another instructor; audit log of every message sent |
| Over-automation erodes instructor relationship | The bot handles logistics only; feedback, coaching and difficult conversations are routed to the human with a drafted message, never sent automatically |
| Hallucinated schedule content | Draft generation is constrained: the model may only place instructors and programmes that exist in the roster, into slots that exist in the template, and every output is validated against the policy rules before the human sees it |
| Vendor lock-in | Thin adapters for messaging, publishing and model; policy and data stay in LUF-owned stores |

## 7. What is needed from Level Up Fitness to start

1. Confirmation of PerfectGym API access (or willingness to ask PerfectGym support for it).
2. Current instructor roster with programmes, certifications, availability and approved rates, in a sheet the agent can read.
3. A named human owner for approvals (Club Manager at City Mall KK, or Kenny for the pilot).
4. A WhatsApp Business number for the agent (or a decision to use Telegram).
5. Confirmation of the pay model in force (flat per class vs booking-tiered) and the current rate table.
6. Which class types in PGM are PT-led and therefore out of scope.
7. Decision on whether the pilot runs alongside the current coordinator for the first 4 weeks (recommended) or replaces the role from day one.
