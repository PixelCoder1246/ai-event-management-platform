# Problem Analysis
## AI-Powered Event Management Platform
**n8n Capstone Project — Assignment 7 (Event Management, Domain: Events & Community Engagement)**

---

## 1. Business Context

Organizations that run recurring conferences, workshops, hackathons, and training programs — from corporate L&D teams to community meetup groups to university clubs — face a recurring operational bottleneck every time they scale beyond a handful of attendees. A single mid-sized event (50–500 participants, multiple sessions/tracks) generates a large volume of repetitive, time-sensitive administrative work: collecting registrations, confirming attendance, reminding people before their sessions, checking people in at the door, issuing proof-of-participation, and following up to learn what worked and what didn't.

Today, most of this is handled through a patchwork of manual processes — spreadsheets updated by hand, reminder emails sent individually or via a mail-merge tool, paper sign-in sheets, and certificates generated one-by-one in Canva or Word and emailed out days (or weeks) after the event. This is workable at small scale but breaks down as event volume and attendee count grow, and it consumes organizer time that would be better spent on content, logistics, and attendee experience.

This project designs and implements a modular, n8n-based automation platform that manages the full attendee lifecycle — from the moment someone registers to the moment they receive their certificate and their feedback is analyzed — with minimal manual intervention, while keeping a human decision point (the organizer) in control of key checkpoints such as "this event has concluded, issue certificates now."

---

## 2. Stakeholders

| Stakeholder | Role in the system | What they need from the platform |
|---|---|---|
| **Event Organizer / Admin** | Owns the event, configures sessions, decides when to close registration and when to issue certificates | Real-time visibility into registrations and attendance; minimal manual data entry; a reliable audit trail; the ability to trigger certificate distribution once an event concludes |
| **Participant / Attendee** | Registers, attends (or misses) sessions, receives communications, gives feedback | A frictionless registration experience; timely, relevant reminders; a fast, contactless check-in; a certificate delivered promptly after attending; an easy way to give feedback |
| **Venue / Session Staff** | Manages physical check-in at the door | A fast, low-training-required way to verify a participant is registered and mark them present (QR scan) without manual list lookups |
| **Marketing / Community team** | Uses post-event data to plan future events | Aggregated feedback sentiment, themes, and attendance analytics without manually compiling spreadsheets |
| **IT / Automation owner (this project)** | Builds and maintains the n8n workflows | A modular system where each workflow can be debugged, re-run, and extended independently; clear logging for troubleshooting |

---

## 3. Pain Points (Current, Manual Process)

1. **Registration data lives in silos.** Sign-ups arrive through a form but aren't validated, de-duplicated, or organized into a usable participant record without manual cleanup.
2. **No reliable reminder system.** Organizers either forget to send reminders or send them to everyone regardless of which session they registered for, causing irrelevant noise or missed sessions.
3. **Manual attendance tracking is slow and error-prone.** Paper sign-in sheets or manually checking names off a list at the door creates bottlenecks at entry and inaccurate attendance records.
4. **Certificates are a bottleneck.** Generating and emailing individual certificates by hand for hundreds of attendees is one of the most time-consuming post-event tasks, and it's common for certificates to go out days late — by which point attendee enthusiasm (and the value of the certificate for things like job applications) has faded.
5. **Duplicate or inconsistent records.** The same person registering twice, or a name being spelled differently between the registration form and the attendance sheet, causes downstream errors (duplicate certificates, missed reminders).
6. **Feedback is collected but rarely analyzed.** Post-event surveys are sent out, but responses usually sit in a spreadsheet unread beyond a quick skim — no structured sentiment or theme analysis is done, so recurring complaints or praise go unnoticed.
7. **No audit trail.** When something goes wrong (a participant says they never got a reminder, or a certificate, or their check-in didn't register), there's no logged record to diagnose what happened.
8. **No single source of truth.** Registration lists, attendance sheets, and feedback forms are often separate files maintained by different people, making it hard to get a unified view of a single participant's journey.

---

## 4. Objectives

The platform is designed to directly address the pain points above:

1. **Automate participant registration and record-keeping** — capture registrations, de-duplicate them, and maintain a single canonical participant record (`Participants` table) that every downstream workflow reads from and writes to.
2. **Enable fast, contactless attendance verification** — issue a unique QR code per participant at registration, and process check-ins via a webhook-driven scan flow with duplicate-check-in protection.
3. **Automate timely, targeted communications** — send session reminders only to participants who haven't yet received one, scoped to the specific session they registered for, timed relative to the session's actual start time.
4. **Automate certificate generation and delivery** — generate a personalized PDF certificate per attended participant and email it automatically, without manual design or send work, gated on verified attendance.
5. **Turn feedback into insight, automatically** — apply AI-based sentiment and theme extraction to every feedback submission as it arrives, and produce a recurring analytics summary for organizers without manual compilation.
6. **Maintain a full audit trail** — log key events (e.g. invalid QR scans, workflow errors) to a dedicated log so issues can be diagnosed after the fact.
7. **Keep the system modular and human-supervised at key checkpoints** — each of the five workflows operates independently and can be triggered, tested, and debugged on its own, while critical actions (e.g. releasing certificates) remain a deliberate, organizer-initiated step rather than a silent background process.

---

## 5. Success Criteria

- A participant can register, receive a QR code, get reminded before their session, check in at the door, and receive a certificate — all without an organizer manually touching a spreadsheet or sending an individual email.
- Duplicate registrations and duplicate check-ins are automatically detected and handled gracefully.
- Reminders and certificates are only sent once per participant (no double-sends), verified via status flags (`reminder_sent`, `certificate_sent`, `attendance`) in the shared participant record.
- Feedback is automatically tagged with sentiment and theme within the same workflow run that captures it, with zero manual review needed before the weekly analytics digest goes out.
- Every workflow can be independently re-run against test data without corrupting the shared dataset, and error/failure states (e.g. a stuck certificate generation) are logged rather than silently dropped.
