# Wings Detailing AI Receptionist (VAPI + n8n)

n8n workflows behind **Mira**, a voice AI receptionist for a car detailing business. VAPI handles the
conversation. n8n is the backend: an MCP server exposes seven tools that read and write a Google Sheet
(CRM) and Google Calendar (scheduling).

## What it does

- Recognizes a caller as a new or returning client by email
- Creates a CRM entry for new clients
- Checks calendar availability, books, reschedules and cancels one-hour appointments
- Logs every call (summary and outcome) after it ends

## Architecture

```
VAPI agent ──(MCP, SSE)──► Vapi MCP Server (MCP Server Trigger)
                              ├─ Client Lookup
                              ├─ New Client CRM
                              ├─ Check Availability
                              ├─ Book Event
                              ├─ Lookup Appointment
                              ├─ Update Appointment
                              └─ Delete Appointment

VAPI end-of-call report ──(POST webhook)──► Wings Receptionist EOC Report ──► Call Log sheet
```

| File | Workflow | Purpose |
|---|---|---|
| `01-vapi-mcp-server.json` | Vapi MCP Server | Router. Exposes the 7 sub-workflows as MCP tools |
| `02-client-lookup.json` | Client Lookup | Finds the email in `Clients`, then checks `Appointment Log` for history |
| `03-new-client-crm.json` | New Client CRM | Appends a row to `Clients` |
| `04-check-availability.json` | Check Availability | Lists calendar events in a time window. Empty means the day is free |
| `05-book-event.json` | Book Event | Creates the calendar event and logs it in `Appointment Log` |
| `06-lookup-appointment.json` | Lookup Appointment | Returns events in a window so the agent can get the event ID |
| `07-update-appointment.json` | Update Appointment | Moves the event and sets Notes to `Moved to <time>` |
| `08-delete-appointment.json` | Delete Appointment | Deletes the event and sets Notes to `Canceled` |
| `09-end-of-call-report.json` | EOC Report | Webhook from VAPI. Appends summary and outcome to `Call Log` |

## Google Sheet layout

Create one spreadsheet with three tabs. The tab names must match exactly.

| Tab | Columns |
|---|---|
| `Clients` | Email, Name, Phone |
| `Appointment Log` | ID, Email, Appointment Type, Date, Notes |
| `Call Log` | Date, Summary, Outcome |

## Setup

No credentials, IDs or URLs are included in these files. Everything account-specific is a placeholder.

1. **Import** each JSON file into n8n (Workflows → Import from file).
2. **Credentials.** Open each Google Sheets and Google Calendar node and select your own credential.
3. **Placeholders.** Replace these values in the nodes that use them:
   - `YOUR_GOOGLE_SHEET_ID` — the ID of your spreadsheet (the long string in its URL)
   - `YOUR_GOOGLE_CALENDAR_ID` — usually the Google account email that owns the calendar
4. **Link the tools.** Open `Vapi MCP Server`. In each `Call '...'` tool node, pick the matching
   sub-workflow from the list. Imported copies have new IDs, so this step is required.
5. **Paths.** Change `your-mcp-path` on the MCP Server Trigger and `your-eoc-webhook-path` on the EOC
   webhook to your own values. Consider adding authentication to both.
6. **Activate** all nine workflows.

## VAPI configuration

1. Tools → create an **MCP Server** tool.
2. Server URL: the **Production URL** of the MCP Server Trigger.
3. Header: `Authorization: Bearer <your n8n credential>` (note the space after `Bearer`).
4. Communication mode: **Server-Sent Events (SSE)**.
5. Attach the tool to your assistant and map the seven tool names.
6. Set the assistant's **Server URL** to the EOC webhook Production URL so end-of-call reports are logged.
7. In the assistant's prompt, use filler phrases ("give me one sec") before each tool call to avoid dead air.

## Notes

- Appointments are one hour. Times are passed to the calendar as ISO strings from the agent.
- Set the timezone in the assistant prompt to match your calendar's timezone.
- Sample data in any docs is fictional.
