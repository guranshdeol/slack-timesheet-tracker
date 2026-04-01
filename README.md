# Kantata Timesheet Compliance Tracker
![Status](https://img.shields.io/badge/status-active-2ea44f)
![Platform](https://img.shields.io/badge/platform-Google%20Apps%20Script-1f6feb)
![Visitor Count](https://visitor-badge.laobi.icu/badge?page_id=guranshdeol.slack-timesheet-tracker)

An automated weekly timesheet accountability system built on **Slack Workflow Builder**, **Google Sheets**, and **Apps Script** — no third-party tools, no code to host, fully within your organisation's existing stack.

Every Friday, a Slack message goes out asking the team to confirm they've filled their Kantata time entries. Anyone who doesn't respond gets a reminder the same night. Anyone still missing by Sunday morning gets escalated to the manager — automatically, with a tag. A Google Sheet tracks every week of the year, flags consistent defaulters, and builds a monthly heat map so patterns are visible at a glance.

---

## How it works

```
Friday 5 PM IST   →  Slack Workflow Builder posts message to channel
                      Team clicks "Yes, I've filled it ✅" button
                      Workflow logs @handle + timestamp → Google Sheet (Responses tab)

Friday 10 PM IST  →  Apps Script checks who hasn't responded
                      Posts reminder in channel tagging non-responders

Sunday 9 AM IST   →  Apps Script runs final check
                      Tags manager with list of still-missing members
                      Updates WeeklyLog (DONE/MISSED per person)
                      Updates DefaulterTracker (risk bands, streaks, miss rate)
                      Updates MonthlySnapshot (heat map)
```

---

## Stack

| Layer | Tool | Purpose |
|---|---|---|
| Messaging | Slack Workflow Builder | Friday trigger, button message, logging clicks |
| Storage | Google Sheets | Responses log, weekly log, defaulter tracking |
| Logic | Google Apps Script | Diff check, alerts, sheet updates |
| Notifications | Slack Incoming Webhook | Posting alerts back to channel |

No Zapier. No Make. No external servers. Everything runs inside tools your org already has.

---

## Folder structure

```
kantata-timesheet-tracker/
├── README.md
├── KantataTracker_Final.gs        ← Apps Script (paste into Google Apps Script)
└── Kantata_Timesheet_Tracker_2026.xlsx  ← Upload to Google Drive, open as Sheets
```

---

## Google Sheet structure

The workbook has 5 tabs:

| Tab | Who writes it | Purpose |
|---|---|---|
| **MasterList** | You (once) | Team @handles, Slack Member IDs, active status |
| **Responses** | Workflow Builder (auto) | Raw log of every button click with UTC timestamp |
| **WeeklyLog** | Apps Script (auto) | DONE/MISSED per person per week — 52 weeks pre-populated |
| **DefaulterTracker** | Apps Script (auto) | Risk band, miss rate, streak, worst month per person |
| **MonthlySnapshot** | Apps Script (auto) | Heat map — misses per person per month |
| **Config** | You (once) | Webhook URL, manager ID, thresholds |

### Defaulter risk bands

| Band | Trigger |
|---|---|
| 🟢 GREEN | 0–1 misses, no bad months |
| 🟡 AMBER | Streak of 2 consecutive missed weeks |
| 🔴 RED | **2+ misses in any single month** (primary rule), or streak of 3+, or 5+ total year misses |
| ⚫ CRITICAL | Miss rate above 20% of elapsed weeks |

---

## Setup guide

### Prerequisites

- Slack workspace on a paid plan (Workflow Builder required)
- Google account with Drive and Sheets access
- Slack admin access to create an incoming webhook

---

### Step 1 — Google Sheet

1. Download `Kantata_Timesheet_Tracker_2026.xlsx`
2. Upload to Google Drive → right-click → **Open with Google Sheets**
3. Go to the **MasterList** tab and fill in your team:
   - Column A: `@handle` exactly as Slack Workflow Builder writes it (test by clicking the button once and checking what appears in the Responses sheet)
   - Column B: `@handle` for alert messages (usually the same as col A)
   - Column C: Slack Member ID — get this from Slack by clicking a person's profile → `⋮` → **Copy member ID** (format: `U0XXXXXXXX`)
   - Column D: `Yes` for active members, `No` for managers/exempt
4. Select columns B and C in the **Responses** tab → Format → Number → **Date time** (prevents serial number display issue)

---

### Step 2 — Slack incoming webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → From scratch
2. Name it `Kantata Logger` (or anything)
3. Left sidebar → **Incoming Webhooks** → toggle on → **Add New Webhook to Workspace**
4. Select your team channel → **Allow**
5. Copy the webhook URL (starts with `https://hooks.slack.com/services/...`)
6. Paste it into the **Config** tab, cell B4 of your Google Sheet

---

### Step 3 — Apps Script

1. In your Google Sheet → **Extensions → Apps Script**
2. Delete all default code
3. Paste the entire contents of `KantataTracker_Final.gs`
4. Click **Save**
5. Go to **Project Settings** (gear icon) → scroll to **Time zone** → set to `(UTC+00:00) UTC`

> **Important:** The `atHour()` in Apps Script triggers uses your Google account's timezone, not the script timezone setting. If your team is in IST (UTC+5:30), set trigger hours in IST directly (see Step 4).

---

### Step 4 — Install triggers

**Option A — Automatic (recommended)**

In Apps Script, select `installTriggers` from the function dropdown → click **Run** → grant permissions.

This installs:
- `firstAlertCheck` — Friday 10 PM IST (non-responder nudge)
- `escalationCheck` — Sunday 9 AM IST (manager escalation + sheet update)

Then go to the Triggers page (clock icon in sidebar) and verify both appear.

**Option B — Manual**

Triggers → Add Trigger → configure each one:

| Function | Event source | Type | Day | Time |
|---|---|---|---|---|
| `firstAlertCheck` | Time-driven | Week timer | Every Friday | 10pm–11pm |
| `escalationCheck` | Time-driven | Week timer | Every Sunday | 9am–10am |

---

### Step 5 — Slack Workflow Builder

1. In Slack → **Tools** (or Automations) → **Workflow Builder** → **New** → **Build Workflow**
2. **Trigger:** On a schedule → Every week → Friday → set to your 5 PM local time
3. **Step 1:** Send a message to your channel:

```
Hey team 👋 Have you filled your Kantata time entries for this week?

Please click the button below to confirm.
```

Add a button: label `Yes, I've filled it ✅`

4. **Add a Branch:** If `User clicked button: Yes, I've filled it ✅`
5. **Inside the branch → Add Step → Google Sheets → Add row:**
   - Sheet: `Responses`
   - Column A (`Slack Display Name`): `{Person who clicked the button}`
   - Column B (`Acknowledged At`): `{Time when the button was clicked}`
   - Column C (`Week Start`): `{Time when workflow started}`
6. **Publish** the workflow

---

### Step 6 — Test before go-live

1. Run `firstAlertCheck` manually in Apps Script → check your Slack channel
2. Click the button in Slack yourself → verify a row appears in the Responses sheet
3. Run `escalationCheck` manually → verify WeeklyLog, DefaulterTracker, MonthlySnapshot all update
4. Run `resetTestData` (below) to wipe the test data before the first real Friday

```javascript
function resetTestData() {
  const SS = SpreadsheetApp.getActiveSpreadsheet();
  const wl = SS.getSheetByName("WeeklyLog");
  if (wl.getLastRow() >= 4) wl.getRange("D4:E" + wl.getLastRow()).clearContent();
  const dt = SS.getSheetByName("DefaulterTracker");
  if (dt.getLastRow() >= 5) dt.getRange("B5:I" + dt.getLastRow()).clearContent();
  const ms = SS.getSheetByName("MonthlySnapshot");
  if (ms.getLastRow() >= 4) ms.getRange("B4:M" + ms.getLastRow()).setValue(0);
  Logger.log("Reset complete.");
}
```

---

## Timezone note

All Slack timestamps are written in **UTC**. The Apps Script logic uses `getUTCDay()`, `getUTCDate()`, and `setUTCHours()` throughout so timezone conversion is never needed. The weekly cutoff is calculated as last Friday at 11:30 AM UTC (= 5:00 PM IST), which is when the Slack workflow fires.

If your team is in a different timezone, adjust the Workflow Builder schedule and the trigger hours accordingly. No changes needed in the script logic itself.

---

## Adapting for your team

| What to change | Where |
|---|---|
| Team members | MasterList tab — add/remove rows, set Active = No to exempt |
| Defaulter threshold (misses/month) | Config tab B13 — default is 2 |
| Alert message text | `firstAlertCheck` and `escalationCheck` functions in the script |
| Channel | Config tab B6 (informational) + Workflow Builder step |
| Manager to tag | Config tab B5 — Slack Member ID |
| Year / week dates | WeeklyLog pre-populated for 2026 — for 2027 regenerate using the Python script in the repo |

---

## Why no third-party tools?

Most timesheet reminder setups use Zapier or Make to bridge the "who hasn't responded" gap. The issue in enterprise Slack is that third-party connectors need IT approval and often aren't available. This setup routes everything through Google Sheets as the state store — Workflow Builder writes to it, Apps Script reads from it and posts back to Slack via a webhook. The only external call is the webhook POST, which is a standard Slack-native feature available on all paid plans.

---

## Contributing

PRs welcome. Likely useful extensions:
- Support for multiple channels / teams
- Carry-forward of partial Kantata completion (not just binary yes/no)
- Monthly summary report posted to Slack automatically
- Google Looker Studio dashboard connected to the Sheets data

---

## License

MIT
