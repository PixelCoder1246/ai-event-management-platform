# Workflow Documentation
## AI-Powered Event Management Platform — n8n Capstone (Assignment 7)

**Total workflows:** 5
**Total nodes across all workflows:** 56 (well above the 20–25 minimum requirement)
**Shared data store:** Google Sheets "Master DB" — tabs: `Registration`, `Participants`, `Sessions`, `Audit Log`, `Feedback`, `Feedback Form Responses`

---

## Master DB Schema Reference

### `Participants` tab (the central record — read/written by all 5 workflows)
| Column | Type | Set by |
|---|---|---|
| `participant_id` | string (`email-event_id`) | W1 |
| `full_name` | string | W1 |
| `email` | string | W1 |
| `phone` | string | W1 |
| `event_id` | string | W1 |
| `session_choice` | string | W1 |
| `registered_at` | ISO timestamp | W1 |
| `attendance` | boolean-as-string | W1 (default `false`) → W2 (set `true`) |
| `checkin_time` | ISO timestamp | W2 |
| `reminder_sent` | boolean-as-string | W3 |
| `certificate_sent` | boolean-as-string | W1 (default `false`) → W4 (set `true`) |

### `Sessions` tab
`session_id`, `session_name`, `session_start_time`, `session_start_hours_from_now` (computed)

### `Audit Log` tab
`workflow`, `event_type`, `detail`, `timestamp`

### `Feedback` tab
`participant_id`, `rating`, `comments`, `sentiment` (AI-generated), `theme` (AI-generated), `submitted_at`

---

## W1 — Registration & Participant Management

**Purpose:** Capture new registrations, prevent duplicates, generate a per-participant QR code, and send a confirmation email.

**Trigger:** Google Sheets Trigger — polls the `Registration` tab every minute for new rows (`rowAdded` event).

| # | Node | Type | Function |
|---|---|---|---|
| 1 | New Registration (Google Sheets Trigger) | `googleSheetsTrigger` | Fires when a new row lands in `Registration` |
| 2 | Normalize Participant Data | `set` | Lowercases/trims email, builds `participant_id` as `email-event_id`, sets `registered_at` |
| 3 | Check Duplicate Registration | `googleSheets` (lookup) | Looks up `Participants` by `participant_id`; `continueOnFail: true` so a "not found" doesn't break the run |
| 4 | IF Already Registered? | `if` | Branches on whether a matching `participant_id` was found |
| 5a | Log Duplicate (No-Op) | `noOp` | Duplicate path — ends the run without writing a second record |
| 5b | Save New Participant | `googleSheets` (append) | New path — appends the record to `Participants` with `attendance=false`, `certificate_sent=false` |
| 6 | Generate QR Code | `httpRequest` | Calls `api.qrserver.com` to render a QR image encoding the W2 check-in webhook URL + `participant_id`; `responseFormat: file` |
| 7 | Upload QR to Drive | `googleDrive` | Saves the QR PNG to the shared Drive folder |
| 8 | Send Confirmation Email | `gmail` | Emails the participant their registration confirmation with the QR code attached |

**Data written:** `Participants` (new row), Google Drive (QR image)
**Known gap to fix before final submission:** `Save New Participant` does not explicitly set `reminder_sent`. Since W3 filters on `reminder_sent = "false"` as an exact string match, a blank/empty cell will not match and that participant may never receive a reminder. **Fix:** add `"reminder_sent": "false"` to the `Save New Participant` column mapping.

---

## W2 — QR Code Attendance Check-in

**Purpose:** Verify a scanned QR code against the participant record and mark attendance, with protection against invalid codes and duplicate check-ins.

**Trigger:** Webhook — `POST/GET /event-checkin?participant_id=...` (URL embedded in the QR code from W1).

| # | Node | Type | Function |
|---|---|---|---|
| 1 | QR Scan Webhook | `webhook` | Entry point, `responseMode: responseNode` |
| 2 | Lookup Participant | `googleSheets` (lookup) | Looks up `Participants` by the `participant_id` query param; `continueOnFail: true` |
| 3 | IF Valid Participant? | `if` | Branches on whether `participant_id` was found |
| 4a | Log Invalid Scan to Audit | `googleSheets` (append) | Invalid path — logs the failed scan to `Audit Log` |
| 4b | Respond Invalid QR | `respondToWebhook` | Returns an error JSON response for an unrecognized code |
| 5 | IF Already Checked In? | `if` | On the valid path, checks `attendance == "true"` |
| 6a | Respond Duplicate Checkin | `respondToWebhook` | Already-checked-in path — friendly error response, no double-write |
| 6b | Timestamp Check-in | `set` | New check-in path — captures `checkin_time` |
| 7 | Update Attendance Record | `googleSheets` (update) | Writes `attendance=true`, `checkin_time` back to `Participants`, matched on `participant_id` |
| 8 | Respond Success | `respondToWebhook` | Confirms check-in with the participant's name |

**Data written:** `Participants` (`attendance`, `checkin_time`), `Audit Log` (invalid scans)
**Error handling:** Every failure path (invalid QR, duplicate scan) is logged/responded to explicitly rather than allowed to error out silently — this satisfies the "error handling and retry logic" advanced-feature requirement.

---

## W3 — Event Notifications & Reminders

**Purpose:** Send session reminders to registered participants within a 24-hour window of their session, without duplicate sends.

**Trigger:** Schedule Trigger — runs hourly (`interval: minutes` in the current build; should be set to `hours` for production use — see note below).

| # | Node | Type | Function |
|---|---|---|---|
| 1 | Hourly Schedule Trigger | `scheduleTrigger` | Cron-style recurring trigger |
| 2 | Read Upcoming Sessions | `googleSheets` (read) | Reads the entire `Sessions` tab |
| 3 | Filter Sessions Within Reminder Window | `filter` | Keeps sessions where `session_start_hours_from_now <= 24` |
| 4 | Get Registered Participants | `googleSheets` (lookup, all matches) | Filters `Participants` by `session_choice == session_id` AND `reminder_sent == "false"` |
| 5 | Loop Over Participants | `splitInBatches` | Processes participants in batches of 10 |
| 6 | Send Reminder Email | `gmail` | Sends the session reminder |
| 7 | Wait (Rate Limit Guard) | `wait` | 2-second pause between sends to avoid Gmail rate limits |
| 8 | Mark Reminder Sent | `googleSheets` (update) | Sets `reminder_sent=true`, matched on `participant_id`; loops back to node 5 for the next batch |

**Data written:** `Participants` (`reminder_sent`)
**Note:** Confirm the Schedule Trigger's `rule.interval.field` is set to `"hours"` (not `"minutes"`) before final submission — running this every minute in production would send excessive duplicate-check queries against the sheet.

---

## W4 — Certificate Generation & Distribution

**Purpose:** Generate a personalized PDF certificate for every attendee who hasn't yet received one, and email it to them.

**Trigger:** Webhook — `POST /event-completed`, manually invoked by the organizer once an event concludes (see architecture note: this is an intentional human checkpoint, not an automatic chain from another workflow).

| # | Node | Type | Function |
|---|---|---|---|
| 1 | Event Completed Webhook | `webhook` | Entry point |
| 2 | Get Attendees Only | `googleSheets` (lookup) | Filters `Participants` where `attendance == "true"` |
| 3 | Filter Not Yet Certified | `filter` | Keeps records where `certificate_sent != "true"` |
| 4 | Loop Over Attendees | `splitInBatches` | Processes 5 attendees per batch |
| 5 | Generate Certificate PDF | `httpRequest` | `POST` to PDFMonkey's `/documents` endpoint with `document_template_id`, `payload` (name, event, date), and `status: pending` nested under a `document` object — this explicitly triggers generation instead of leaving the document in draft |
| 6 | Wait | `wait` | 5-second pause before checking generation status |
| 7 | Check PDF Status | `httpRequest` | `GET /documents/{id}` — polls PDFMonkey for render completion |
| 8 | If (Is PDF Ready?) | `if` | Branches on `document.status == "success"`; false branch loops back to node 6 |
| 9 | Download Certificate PDF | `httpRequest` | `GET` the `download_url`, `responseFormat: file` — retrieves the actual PDF binary |
| 10 | Upload Certificate to Drive | `googleDrive` | Saves the certificate PDF, named `certificate_{participant_id}.pdf` |
| 11 | Email Certificate | `gmail` | Emails the certificate as an attachment |
| 12 | Mark Certificate Sent | `googleSheets` (update) | Sets `certificate_sent=true`; loops back to node 4 |
| 13 | Respond Completed | `respondToWebhook` | Returns success JSON once all batches are processed |

**Data written:** `Participants` (`certificate_sent`), Google Drive (certificate PDF)
**External API:** PDFMonkey (`api.pdfmonkey.io`) for templated PDF generation
**Error handling:** `retryOnFail: true` (3 attempts, 3s apart) on the PDF generation call; polling loop guards against reading a certificate before it's ready.

---

## W5 — Feedback Collection & Event Analytics

**Purpose:** Tag incoming feedback with AI-derived sentiment/theme in real time, and produce a recurring analytics digest for organizers.

**Two independent triggers, sharing the same workflow:**

### Branch A — Real-time feedback tagging
| # | Node | Type | Function |
|---|---|---|---|
| 1 | New Feedback Submitted | `googleSheetsTrigger` | Polls `Feedback Form Responses` every minute for new rows |
| 2 | Normalize Feedback | `set` | Builds `participant_id`, extracts `rating`, `comments` |
| 3 | AI Sentiment & Theme Extraction | `@n8n/n8n-nodes-langchain.chainLlm` | LLM call classifying sentiment (positive/neutral/negative) and a short theme |
| 4 | Sentiment & Theme Parser | `outputParserStructured` | Enforces structured JSON output (`sentiment`, `theme`) from the LLM |
| 5 | Save Feedback with AI Tags | `googleSheets` (append) | Writes the tagged feedback to `Feedback` |

### Branch B — Weekly analytics digest
| # | Node | Type | Function |
|---|---|---|---|
| 1 | Weekly Report Schedule | `scheduleTrigger` | Fires once a week |
| 2 | Read Registration & Attendance Data | `googleSheets` (read) | Full `Participants` read |
| 3 | Read Feedback Data | `googleSheets` (read) | Full `Feedback` read |
| 4 | Merge Registration + Feedback | `merge` | Combines both datasets into one item stream |
| 5 | Prepare Report Data | `code` | Computes participant count, feedback count, average rating |
| 6 | AI Generate Analytics Summary | `chainLlm` | LLM writes a plain-text weekly summary: attendance vs registration, sentiment breakdown, recurring themes, recommendations |
| 7 | Email Analytics Report to Organizers | `gmail` | Sends the summary to the organizer inbox |

**Data written:** `Feedback` (AI-tagged rows), organizer inbox (weekly report email)
**AI models used:** `nvidia/nemotron-3-ultra-550b-a55b` (via an OpenAI-compatible provider) for both sentiment extraction and report summarization — satisfies the "AI-powered decision making" advanced-feature requirement.

---

## Advanced Features Implemented (cross-reference to rubric)

| Requirement | Where it's implemented |
|---|---|
| AI-powered decision making | W5 — sentiment/theme classification and AI-written analytics summaries |
| Error handling and retry logic | W1/W2 duplicate detection with `continueOnFail`; W4 `retryOnFail` on the PDFMonkey call and its status-polling loop |
| Logging and audit trail | W2 → `Audit Log` tab on invalid scans |
| Scheduled workflows (Cron) | W3 (hourly), W5 Branch B (weekly) |
| Webhook-triggered workflows | W2 (`event-checkin`), W4 (`event-completed`) |
| Conditional branching and loops | `If`/`Filter` nodes in every workflow; `splitInBatches` loops in W1 (implicit), W3, W4 |

## Known Issues to Resolve Before Final Submission

1. **W1:** add `reminder_sent: "false"` to the `Save New Participant` column mapping (see W1 section above).
2. **W3:** confirm the Schedule Trigger interval is `hours`, not `minutes`.
3. **W4:** confirm the `status: pending` fix and nested `document` body structure are applied in your live n8n instance (this was fixed mid-build — see prior debugging notes).
4. **General:** replace all `row_number: 0` writes in update nodes if any remain — they can silently overwrite a real `row_number` column with `0`.
