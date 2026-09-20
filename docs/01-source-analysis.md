# 01 - Analysis of the Source Documents

Five files were supplied plus the Class Booking Policy poster. Two of the Word files
(`Kennys_Briefing_Notes_for_Jessel.docx` and `..._1.docx`) are byte-identical duplicates,
so there are four distinct sources.

| # | Document | Date | What it tells us |
|---|----------|------|------------------|
| 1 | GX Coordinator - Job Description Form | 2022 | Original role definition (City Mall KK, freelance 8-12 h/week, reports to Area Manager KK) |
| 2 | Kenny's Briefing Notes for Jessel | 19 Dec 2022 | The actual operating playbook: tools, weekly/monthly cadence, policies, judgement calls |
| 3 | GX Coordinator Independent Contractor Agreement + Schedule A | 1 Jul 2026 | Current contractual scope, RM 800/month, confidentiality, 2-month notice |
| 4 | QCC_LUF.xlsx - Quality Control Checklist | undated | 20-point instructor audit rubric, pass mark 15/20 |
| 5 | Class Booking Policy poster | undated | Member-facing rules the coordinator enforces (77 h / 2 h / 1 h / class time) |

## 1. What the GX Coordinator actually does

Combining the three role documents, the job decomposes into seven duty clusters.
The "AI-fit" column is my assessment of how much of each cluster a software agent can take on.

| Cluster | Concrete deliverables (from the sources) | Cadence | AI-fit |
|---------|------------------------------------------|---------|--------|
| A. Weekly timetable | Draft schedule Wed, chase every instructor to confirm, revise until all confirmed, upload to PerfectGym (PGM), Foyer digital signage and Facebook by Thu/Fri | Weekly | **High** - this is structured, repetitive, deadline-driven coordination |
| B. Cover / substitution | On instructor absence, find a qualified replacement, coordinate with Club Manager, notify every member booked on the affected class | Ad hoc, time-critical | **High** for the broadcast and matching; **Medium** for negotiation with instructors |
| C. Monthly payroll | Compile per-instructor class counts and rates, produce payroll report, submit to HQ Admin / payroll@ before the 5th (2026) or 7th (2022), answer verification queries | Monthly | **High** for compilation; approval stays human |
| D. Performance monitoring | "Ensure GX bookings and attendance numbers are optimized"; identify underperforming time slots before adding new ones; keep Les Mills programmes above 3 classes/week or trigger the 3-month licence-drop decision | Weekly / monthly | **High** - pure analytics on booking data |
| E. Instructor quality & development | Run the 20-point QCC audit, give constructive feedback, onboard and assess new instructors, recruit mentors, monitor tuition classes | Periodic | **Low** for the in-studio observation; **Medium** for scheduling audits, scoring, tracking, feedback drafting |
| F. Policy & SOP compliance | Enforce the booking policy, payment schedule, mic-handling SOP, free-membership eligibility (2 regular classes/week, reviewed Jan/Apr/Jul/Oct) | Continuous | **High** for monitoring and reminders |
| G. Liaison, events, meetings | Be visible on the floor, collect member and instructor feedback, escalate to management meeting (weekly then fortnightly), support launches and themed events with HQ Marketing | Continuous | **Low** for presence and relationships; **Medium** for meeting prep, feedback capture and event checklists |

## 2. Hard rules and numbers encoded in the documents

These become the agent's policy table. Every one of them is currently held in someone's head or a document, not in a system.

**Scheduling**
- Draft goes out Wednesday; final published by Thursday (2026 agreement) / Friday (2022 notes). The 2026 agreement is the stricter and current one.
- Publish to three surfaces: PGM (`levelup.perfectgym.pl`), Foyer signage (`signage.levelupfitness.com`), Facebook.
- "Schedule a small number of classes and make sure each one is filled." Do not add slots on member demand; retire an underperforming slot before adding a new one.
- New instructors and new rates need sign-off from Kenny and Yen before they appear on the schedule.
- Seasonal dip: December and pre-CNY weeks run a reduced schedule, back to normal by February.

**Les Mills licence economics**
- Keep at least 3 classes/week per licensed programme (BodyPump, BodyCombat, BodyJam, BodyBalance). RPM was dropped Feb 2023.
- Dropping a programme needs 3 months' notice. Priority to retain: BodyPump, then BodyCombat / BodyJam.

**Member booking policy (poster)**
- Booking opens 77 hours before class. Latest cancel without penalty: 2 hours before. Online booking closes 1 hour before (front desk after). Class is cancelled if it has 2 or fewer bookings. More than 5 minutes late = entry may be denied, slot given to waitlist.

**Instructor policy**
- Free membership if teaching at least 2 regular classes/week; reviewed quarterly; one class/week gets day-of-class entry only.
- Pay model was planned to move to booking-based pay by July 2023 (reward instructors whose classes fill).
- Mic SOP: Shure SM31FH, cable care, sponge cover, own pouch; breakage cost may be apportioned.

**Quality audit (QCC)**
- 20 binary items across Pre-class (5), Introduction (3), During (6), Post-class (6). Pass = 15/20. Signed by instructor and assessor.

**Payroll**
- Monthly report to HQ Admin before the 5th. The coordinator's own RM 800 fee rides on the same payroll.

**Confidentiality**
- Instructor personal data, contact details and individual rates must never be shared with third parties. This constrains where an AI system may store data and which vendors can process it.

## 3. Gaps and contradictions worth flagging

1. **Deadline drift.** 2022 notes say Friday, 2026 agreement says Thursday. The agent should use Thursday and treat Wednesday draft as a hard start.
2. **Payroll date drift.** 7th (2022) vs 5th (2026). Use the 5th.
3. **Booking-based pay** was announced for July 2023 but no document confirms it was implemented. The payroll module has to support both flat-rate-per-class and booking-tiered pay until confirmed.
4. **The QCC has no scoring guidance** beyond binary tick and 15/20 pass. There is no record of how often audits run, who runs them, or what happens on a fail. Established operators run audits at least twice a year per instructor with a documented re-audit path.
5. **No written cover/substitution SOP.** The agreement says "actively find and secure replacements" but there is no qualification matrix (who may teach which programme), no notice threshold, and no member-notification template.
6. **The 2022 notes recommend against a WhatsApp instructor group** because removing an instructor later is awkward. This is actually an argument for the AI design proposed here: one-to-one channels via a bot avoid the group dynamic entirely.
7. **Credentials are written in the briefing notes** (Fitbox Virtual login). They should be rotated and never enter any AI system's prompt or storage.
8. **"Performance" is undefined.** The JD says optimise bookings and attendance but sets no target. The proposal defines a baseline KPI set (fill rate, attendance rate, no-show rate, late-cancel rate, cost per attendee) so the agent has something to optimise against.
9. **Scope exclusion.** PT-led small-group classes are out of scope (2026 agreement). The agent must be able to tag and ignore those class types in the booking system's data.
10. **Booking platform has changed since the notes were written.** The 2022 notes point to PerfectGym (`levelup.perfectgym.pl`). Kenny has confirmed the live system is now Sentinel (Sentinel Fitness by Scope Software Solutions, tenant `levelup.sentinelscope.com`, apps released July 2025). Every "upload to PGM" step in the notes now means Sentinel. Sentinel has no public API documentation, which is the single biggest constraint on the design (see `02-industry-research.md` section 7.3).

## 4. What the coordinator is not

Both the JD and the notes are explicit that the coordinator is part of management and the human face between instructors, members and the company: floor presence, conversations before and after class, coaching instructors, and sitting in the management meeting. That is the part an AI cannot replace and the proposal does not pretend it can. The realistic target is to remove 70-80% of the administrative hours (scheduling, chasing, uploading, payroll compilation, reporting) so that a much smaller human role, or the Club Manager, can carry the relationship work.
