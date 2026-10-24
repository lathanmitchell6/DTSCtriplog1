# MTM Trip Log

A Google Apps Script web app that lets drivers log pick-ups and drop-offs from their phones. Each submission fills in a trip log tab in a Google Sheet, exports it as a PDF to a structured Google Drive folder, and sends finished logs and compliance alerts by email.

## What it does

- **Web form** (`Index.html`): a Pick-Up / Drop-Off chooser with one form for each.
- **Trip log sheet**: one template tab per route and run (`R1 AM`, `R1 PM`). Each driver gets their own copy for the day.
- **PDF filing**: after every submission the tab is exported to PDF and saved in Drive under Year > Month > Week > Day. Re-submitting replaces the previous PDF.
- **Final emails**: the finished AM and PM PDFs are emailed once per route per run to the submission recipient.
- **Alerts**: daily "logs missing / good job" checks, a morning reminder, and a weekly summary go to the alert recipient.
- **Email attachment intake**: attachments on unread emails under a route's Gmail label are saved into that route's Drive folders.

## Files

| File | Purpose |
|---|---|
| `Code.gs` | All server logic: config, form handlers, sheet and PDF work, Drive helpers, alerts, emails |
| `Index.html` | The web app front end (form UI, license-number helper, chooser) |

## How a submission flows

1. The driver opens the web app and chooses **Picking Up** or **Dropping Off**.
2. They pick the route, date and AM/PM run. The individuals list loads from column B of that route's template tab.
3. **Pick-up** writes AM/PM (column C), pick-up time (E) and pick-up departure (F) on the individual's row. Column F is the pick-up time plus a random 1 to 3 minutes. The driver's license, signature (in cursive) and printed name go in the footer rows.
4. **Drop-off** writes the drop-off time and attendant signature (columns G and H) on the individual's row, in the tab where that person was picked up.
5. The tab is exported to PDF and saved in the Drive day folder.

Tab naming: `d_<ROUTE>_<YYYY-MM-DD>_<AM|PM>_<last 4 of license>`, for example `d_R1_2026-10-09_AM_5678`.
PDF naming: `<YYYY-MM-DD>_<ROUTE>_<AM|PM>_<last 4 of license>_Trip_Log.pdf`.

Run tabs older than 7 days are deleted automatically. The PDFs in Drive are kept.

## Setup

### 1. Prepare the Google Sheet

- Create (or open) the trip log spreadsheet and copy its ID into `SHEET_ID`.
- For each route, create two template tabs named exactly `<route code> AM` and `<route code> PM`, for example `R1 AM` and `R1 PM`.
- The script finds rows by the labels in **column A**, so each template needs:
  - a row starting with `Date of Service`
  - a row starting with `Driver` and containing `License`
  - a header row starting with `MTM Trip Number`
  - optionally a row starting with `Each leg` (marks the end of the trip rows)
  - optionally `Driver ... Signature` and `Driver ... Printed Name` rows
- Trip rows sit between the header row and the `Each leg` row. Columns: **B** member's printed name, **C** AM/PM, **D** scheduled pick-up time, **E** pick-up time, **F** pick-up departure, **G** drop-off time, **H** signature.
- **To add or remove an individual, edit the names in column B of the template tab.** Nothing in the code changes.
- Templates are never modified by the script.

### 2. Prepare Drive and Gmail

- Create one root Drive folder per route and note its ID.
- Create one Gmail label per route if you want email attachments filed automatically.

### 3. Configure `Code.gs`

```js
var ROUTES = [
  {
    name: "R1",                      // route code, must match the tab names
    gmailLabel: "Route 1 Trip Logs", // Gmail label for attachment intake
    driveFolderId: "<root folder id>"
  }
  // add one object per route
];

var SUBMISSION_RECIPIENT = "...";  // receives the final AM/PM PDFs only
var ALERT_RECIPIENTS = "...";      // receives all alerts and reports
var SHEET_ID = "...";
```

Other settings:

| Setting | Meaning |
|---|---|
| `LOGS_PER_DAY` | Expected runs per day (2: AM and PM) |
| `PM_EXPECTED_AFTER_HOUR` | Checks before this hour (24h, script time zone) expect only the AM run; later checks expect AM and PM |
| `DAY_TAB_KEEP_DAYS` | How long run tabs stay in the sheet (default 7) |
| `SIGNATURE_FONT`, `SIGNATURE_FONT_SIZE` | Cursive font used for signatures |
| `DEPARTURE_MIN_MINUTES`, `DEPARTURE_MAX_MINUTES` | Random offset range for the pick-up departure time |

The route list label shown in the form is the Gmail label with a trailing "Trip Logs" removed.

### 4. Deploy the web app

1. In the Apps Script editor choose **Deploy > New deployment > Web app**.
2. Execute as the account that owns the sheet, Drive folders and Gmail, and set access to whoever should be able to use the form.
3. Share the web app URL with drivers.
4. Authorize the requested scopes (Sheets, Drive, Gmail, external requests for PDF export) on first run.

### 5. Add triggers

Use **Triggers > Add Trigger** in the editor. All of these skip weekends themselves.

| Function | Schedule | What it does |
|---|---|---|
| `saveLabelAttachmentsToDrive` | Time-driven (for example every 15 min) | Saves attachments from unread labeled emails to Drive, then marks them read |
| `sendFinalAmLogs` | Day timer, 11am to 12pm | Emails the day's finished AM PDFs, one email per route |
| `sendFinalPmLogs` | Day timer, 6pm to 7pm | Emails the day's finished PM PDFs, one email per route |
| `checkDailyTripLogs` | Day timer, 12pm to 1pm | Midday check (AM run expected) |
| `checkDailyTripLogs` | Day timer, 7pm to 8pm | End-of-day check (AM and PM expected) |
| `resendMorningTripLogReport` | Day timer, 6am to 7am | Resends the previous evening's report as a reminder |
| `sendWeeklyTripLogReport` | Week timer, Friday 8pm to 9pm | One summary email, each route shown as total / 10 |

## Behavior to know about

- **Multiple drivers on one run**: each driver (identified by license) gets their own tab and PDF. The run still counts as one run for alerts.
- **Correcting a mistake**: submitting again for the same individual and type overwrites that row.
- **Drop-off before pick-up**: creates a tab with no driver. The next pick-up claims it.
- **Drivers should keep the page open**: the license number is kept in memory on the page only. Closing or refreshing means typing it again.
- **License format**: 1 letter followed by 8 numbers (9 characters). Spaces and dashes are stripped, and the form enforces the pattern on both client and server.
- **Concurrent submissions** are serialized with a script lock (30 second wait).
- **Email attachments** cannot be tied to AM or PM, so each one fills one missing slot in the daily and weekly counts.
- **Final emails only send PDFs already in Drive**, so supervisors never see in-progress versions.

## Troubleshooting

| Problem | Likely cause |
|---|---|
| "No 'R1 AM' tab found" | Template tab missing or named differently from `<route code> <AM/PM>` |
| "Could not find the Date / License / header rows" | A required label is missing from column A of the tab |
| "'Name' is not listed on this route's tab" | The name in column B changed after the tab was created |
| "No free trip row on the tab" | More people than rows between the header and the `Each leg` row |
| No individuals appear in the dropdown | Column B of the template is empty |
| No alert or final email | Trigger missing, weekend, or no PDFs for that run yet. Check **Executions** in the editor |
| PDF export fails | The script account lacks access to the sheet, or authorization was not granted |

Logs for every function are in the Apps Script **Executions** view.

## Known issue in the supplied `Code.gs`

The `ROUTES` array in the pasted configuration is missing the closing `}` for the route object, so it should read:

```js
var ROUTES = [
  {
    name: "EX",
    gmailLabel: "Example",
    driveFolderId: "..."
  }
];
```

Also, the variable `isPastNoon` in `checkDailyTripLogsForRoute_` actually tests against `PM_EXPECTED_AFTER_HOUR` (3pm), so the name is misleading but the logic is correct.
