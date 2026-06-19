# Water Bill Spike Alert

Watches your Gmail inbox for the monthly City of Palm Bay FL utility bill reminder,
extracts the balance due using OpenAI, and sends a push notification if the bill
jumps more than 25% compared to last month.

---

## How It Works

```
New email arrives in Gmail
  → IMAP integration fires imap_content event
    → Automation checks sender + subject (filter happens here, not in IMAP)
      → Matches? → ai_task.generate_data extracts "Balance Due" dollar amount
        → Amount > 125% of last month? → Push notification to Pixel 8
        → Always: rotate current → previous, store new amount
```

**Trigger is email-driven** — fires the moment the reminder email lands, not on a
fixed schedule.

---

## Components

### Helpers

| Entity | Purpose |
|---|---|
| `input_number.water_bill_current_month` | Most recent bill amount (USD) |
| `input_number.water_bill_previous_month` | Previous month's bill for comparison |

Both created via Settings → Helpers with range 0–1000, step 0.01, box mode.

### Automation

**Entity:** `automation.water_bill_update_spike_alert`

**Trigger:** `event_type: imap_content` — fires for every new unread email via the
IMAP integration (`johnrbarton27@gmail.com`, entry `01KVEV6YJ1299XK6FJJWP1RB37`).

**Condition (email filter):**
```
sender contains  → no-reply@invoicecloud.net
subject contains → City of Palm Bay FL
subject contains → Reminder
```
The IMAP integration is set to watch all unread inbox mail (no server-side filter),
so this condition gates the rest of the automation. Non-matching emails cost nothing.

**Actions (in order):**

1. `ai_task.generate_data` — passes the full email body (may be raw HTML) to
   `ai_task.openai_ai_task` asking it to extract the Balance Due dollar amount.
   Result stored in `ai_result`; the extracted amount is `ai_result.data.amount`.

2. `choose` — if `old_amount > 0` and new amount exceeds `old_amount × 1.25`,
   sends a push notification to `notify.mobile_app_pixel_8` with the % change and
   both dollar amounts. Template condition is required here because both thresholds
   are dynamic (response variable vs. helper state).

3. `input_number.set_value` — writes `old_amount` into `water_bill_previous_month`.

4. `input_number.set_value` — writes `ai_result.data.amount` into
   `water_bill_current_month`.

**Mode:** `single` — prevents duplicate runs if IMAP reconnects and replays the
same event.

---

## Setup

### Prerequisites

| Requirement | Notes |
|---|---|
| IMAP integration | Settings → Add Integration → IMAP. Server: `imap.gmail.com`, port 993. Needs a Gmail App Password. Set `event_message_data` to include `text` and `headers`. |
| AI Task integration | OpenAI integration with at least one `ai_task` entity (`ai_task.openai_ai_task`). |
| HA Companion app | For push notifications to `notify.mobile_app_pixel_8`. |

### First-Time Seeding

The helpers start at 0. The spike condition requires `old_amount > 0`, so the first
real bill won't trigger an alert — but set the current bill manually so the *next*
month compares correctly:

1. Go to **Developer Tools → States** (or Settings → Helpers).
2. Find `input_number.water_bill_current_month`.
3. Set its value to your most recent bill (e.g., `147.43`).

---

## Notification Format

```
Title:   Water Bill Spike!
Message: Palm Bay water bill jumped 31.2% — was $112.45, now $147.43.
```

---

## Utility Details

| Field | Value |
|---|---|
| Utility | City of Palm Bay FL |
| Billing platform | InvoiceCloud |
| Sender | `no-reply@invoicecloud.net` |
| Subject pattern | `City of Palm Bay FL Invoice# ... Reminder` |
| Amount field | "Balance Due" in the email sidebar |
| Email format | HTML only (no plain text part) |
| Billing cycle | Monthly, AutoPay via ACH |
| Typical reminder timing | ~7 days before AutoPay date (mid-month) |

---

## Troubleshooting

**Automation never fires**
- Check that the IMAP integration is loaded (Settings → Devices & Services → IMAP).
- In Developer Tools → Events, listen for `imap_content` and send a test email to
  yourself to confirm the event fires.
- Confirm the event data includes `sender`, `subject`, and `text` fields (set by
  `event_message_data: ["text", "headers"]` in the IMAP options).

**Amount extracted incorrectly**
- The email body is HTML. OpenAI parses it reliably, but you can verify what was
  extracted by checking `input_number.water_bill_current_month` after the run.
- In the automation trace (Settings → Automations → ··· → Traces), inspect the
  `ai_result` response variable to see the raw extracted value.

**Spike alert not firing despite a large increase**
- Check that `water_bill_previous_month` is non-zero. If it's 0, the `old_amount > 0`
  guard suppresses the alert — seed it manually (see First-Time Seeding above).
- Confirm the automation `mode: single` isn't blocking a second run. Check the trace.

**Wrong bill captured (e.g., Payment Confirmation instead of Reminder)**
- InvoiceCloud sends two emails per cycle: a Reminder (~7 days before) and a Payment
  Confirmation (after charge). The subject condition requires `Reminder`, so the
  confirmation is ignored. If both fire in the same month, `mode: single` prevents
  a second update on that same run — but a new unread email after HA restarts could
  retrigger. Safe because both emails carry the same balance due amount.

---

## Extending This

**Add email notification on spike**
Add a second action in the spike `sequence` using `notify.smtp` (if configured) or
a `rest_command` to a notification service of your choice.

**Track history over time**
Create a `statistics` helper sourced from `input_number.water_bill_current_month`
to surface trends in a dashboard card.

**Add to an energy/utility dashboard**
Both `input_number` helpers can be added directly to an Entities card or a
Statistics card for a month-over-month view.

**Adjust the spike threshold**
The 25% threshold is hardcoded in the automation condition
(`old_amount * 1.25`). To make it user-adjustable, create an
`input_number.water_bill_spike_threshold` helper (e.g., range 5–100, default 25)
and change the condition to `ai_result.data.amount | float(0) > old_amount * (1 + states('input_number.water_bill_spike_threshold') | float(25) / 100)`.
