# Campus Autopilot - College Notice Automation

An [n8n](https://n8n.io) workflow that turns a faculty-submitted college notice into a targeted email broadcast and, when the notice has a date and time, a Google Calendar event. Faculty fill in a simple web form; the workflow uses AI to extract the details, finds the right students in a Google Sheet, checks for calendar clashes, and sends the notice only when every safety check passes.

## Features

- **Simple faculty form**: pick Branch, Year, Division and paste the notice text.
- **AI extraction (Claude)**: pulls out event type, title, venue, date, time, deadline, links and a short summary. Anything not stated in the notice is left `null`; the model is told never to invent details.
- **Exact targeting**: recipients come only from the Branch + Year + Division chosen in the form, never from what the AI guesses.
- **Recipient cleaning**: validates email format, removes duplicates and drops any row that does not match the selected group.
- **Calendar conflict check**: looks for overlapping events in the target time slot.
- **Automatic calendar invites**: creates an event with all matching students as attendees, only when the slot is free.
- **Duplicate protection**: the same notice sent to the same group twice is blocked.
- **Safety gates**: sending is blocked when something looks wrong (see below).
- **Execution log**: every run returns a status report and is recorded in workflow static data.

## How It Works

```
Faculty form
   -> Validate fields
   -> AI extracts notice details
   -> Look up students in Google Sheet
   -> Validate and deduplicate recipients
   -> Has date + time?
        yes -> Check calendar conflicts
        no  -> skip
   -> Create calendar event (only if no conflict)
   -> Build email + preview
   -> Live send allowed?
        yes -> Send one email per student
        no  -> preview only
   -> Record execution status
```

### Workflow stages

| Stage | What it does |
|---|---|
| **Receive notice** | Takes the form submission and rejects incomplete or invalid input. |
| **Identify recipients** | AI extracts notice fields, then finds students matching branch, year and division with valid, unique emails. |
| **Check schedule** | If the notice has a start date and time, searches Google Calendar for overlapping events. |
| **Draft notification** | Creates the calendar event only when it is safe, then writes the email and a preview. |
| **Deliver and record** | Sends emails only when live mode is authorized, then logs the outcome. |

### Send safety checks

Emails are sent only if **all** of the following are true:

- Test mode is off and live sending is authorized
- The submission is not a duplicate
- At least one matching student was found, and no more than **300** (safety cap)
- AI extraction succeeded and returned valid JSON
- There is no calendar conflict
- If the notice had a schedule, the calendar event was actually created

Otherwise the workflow returns a `preview_only` result listing the reasons the send was blocked.

### Possible output statuses

`sent`, `sent_with_failures`, `preview_only`, `duplicate_blocked`, `no_matching_students`, `rejected_invalid_form`

## Requirements

- An n8n instance (cloud or self-hosted) with support for the Form Trigger node (v2.6+)
- **Anthropic API** credentials (the workflow uses `claude-haiku-5-5`)
- **Google Sheets** OAuth2 credentials
- **Google Calendar** OAuth2 credentials
- **Gmail** OAuth2 credentials

## Setup

### 1. Prepare the student directory

Create a Google Sheet with a tab named **`Students`** and these columns:

| Roll_No | Name | Email | Branch | Year | Division |
|---|---|---|---|---|---|
| 101 | Example Student | student@example.com | CSE | 3 | A |

Notes:
- `Branch` must match one of: `CSE`, `CSE-DS`, `CSE-AIML`, `Civil`, `Mechanical`, `ENTC`
- `Year` is stored as a number: `2` = SY, `3` = TY, `4` = Final Year
- `Division` is a letter from `A` to `I`

### 2. Import the workflow

1. In n8n, go to **Workflows -> Import from file**.
2. Select `Campus Autopilot - College Notice Automation.json`.

### 3. Connect your own accounts

The exported file contains values from the original author's setup. Replace them with your own:

- **Get Row(s) in sheet**: select your own Google Sheet.
- **Check Calendar Conflicts** and **Create Calendar Event**: select the calendar you want to use.
- **AI Extract Notice Fields**: attach your Anthropic credentials.
- **Send Notification Email**: attach your Gmail account.

### 4. Test before going live

In the **Validate Form Fields** node, set:

```js
test_mode: true,
live_send_authorized: false,
```

With these settings the workflow does everything except send emails, so you can inspect the preview output. When you are happy, switch back to `test_mode: false` and `live_send_authorized: true`, then activate the workflow.

### 5. Use it

Open the form's production URL (path: `/campus-autopilot-notice`), choose the target group, paste the notice and submit.

## Configuration

| Setting | Where | Default |
|---|---|---|
| Branch / Year / Division options | Form Trigger node | See form fields above |
| Test mode and live send flag | Validate Form Fields node | `test_mode: false`, `live_send_authorized: true` |
| Recipient safety cap | Generate Notification and Preview node | 300 |
| Default event length when no end time is given | Evaluate Calendar Result node | 1 hour |
| Time zone | Calendar and date logic | Asia/Kolkata (IST) |
| AI model | AI Extract Notice Fields node | `claude-haiku-5-5` |

## Notes and Limitations

- Duplicate detection and the run log use n8n workflow static data, which only persists in **active** (production) executions, not manual test runs.
- If a calendar conflict is found, no calendar event is created and the email is held back so the clash can be resolved first.
- Calendar invites and emails are sent from the connected Google accounts, so check your sending limits for large groups.
- The AI is instructed not to guess missing details, but faculty should still review the preview for important notices.

## Security

Before publishing this repository, make sure your exported workflow does **not** include personal data such as email addresses, Google Sheet IDs, calendar IDs or credential IDs. Never commit API keys or OAuth secrets.

## License

Add a license of your choice (for example MIT) before publishing.
