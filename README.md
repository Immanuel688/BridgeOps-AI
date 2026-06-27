# Bridge Ops

An automated IT incident bridge call orchestrator. When a critical incident hits, Bridge Ops pages the right teams, drafts a Google Calendar invite with a Meet link, gets human approval, sends it — then calls the on-call CIM via an AI voice agent to confirm their ETA.

---

## How It Works

```
Service Desk Agent
       │
       ▼
  [Screen 1] Enter incident details (ticket ID, priority, subject, context)
       │
       ▼
  LangGraph Pipeline
  ├── Node 1: Collect input
  ├── Node 2: Retrieval lookup — fetch teams & on-call CIM from Google Sheets, LLM routes to matched teams
  ├── Node 3: Draft calendar invite
  ├── Node 4: Human approval (interrupt — agent reviews/edits draft)
  └── Node 5: Send Google Calendar invite with Meet link
       │
       ▼
  [Screen 2] Agent reviews and approves invite
       │
       ▼
  [Screen 3] Meet link shown immediately
  + Background: Twilio outbound call → OpenAI Realtime voice agent → captures CIM ETA
```

---

## Features

- **LangGraph-powered workflow** with a human-in-the-loop approval step backed by a PostgreSQL checkpointer
- **Google Sheets as live config** — teams/DLs, on-call CIM schedule (primary + backup with time windows), and fixed attendees are all pulled at runtime
- **LLM-based team routing** — GPT-4o-mini matches the incident subject/context to the right team distribution lists
- **Google Calendar integration** — creates an event with a Meet link and emails all attendees automatically
- **AI voice call** — Twilio + OpenAI Realtime bridges a phone call to the CIM, follows a strict script, and extracts their ETA
- **ETA polling** — the results page polls `/thread-eta/{thread_id}` until the voice agent confirms the CIM's ETA
- **Containerised for Cloud Run** — single Gunicorn/UvicornWorker, secrets via Secret Manager

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web framework | FastAPI + Jinja2 |
| Agent orchestration | LangGraph |
| LLM | OpenAI GPT-4o-mini |
| Voice agent | OpenAI Realtime API + Twilio |
| Calendar & Sheets | Google APIs (service account + OAuth) |
| Persistence | PostgreSQL (via psycopg3 + async pool) |
| Deployment | Docker → Google Cloud Run |

---

## Project Structure

```
.
├── main.py            # FastAPI app, lifespan (DB pool + LangGraph setup)
├── graph.py           # LangGraph nodes and graph assembly
├── state.py           # IncidentState TypedDict
├── routes_ui.py       # UI routes: input form, approval, results
├── routes_call.py     # Call routes: Twilio webhook, OpenAI Realtime WS bridge, ETA polling
├── call_utils.py      # Background task helper to trigger the CIM call
├── sheets.py          # Google Sheets fetchers (teams, CIM schedule, fixed members)
├── calendar_tool.py   # Google Calendar invite creator
├── db.py              # Async psycopg connection pool
├── app_state.py       # Module-level graph reference shared across routes
├── requirements.txt
├── Dockerfile
├── templates/         # Jinja2 HTML templates
└── static/            # Static assets
```

---

## Environment Variables

Create a `.env` file (never commit it):

```env
# OpenAI
OPENAI_API_KEY=

# Twilio
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# OpenAI Realtime model
OPENAI_REALTIME_MODEL=gpt-realtime-mini-2025-12-15

# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Google Sheets IDs
GOOGLE_SHEET_ID_TECH=
GOOGLE_SHEET_ID_CIM=
GOOGLE_SHEET_ID_FIXED=

# Google service account (local path or Secret Manager path)
GOOGLE_SERVICE_ACCOUNT_FILE=path/to/service_account.json

# Google Calendar OAuth token
CALENDAR_TOKEN_PATH=path/to/token.pickle

# Public URL (ngrok locally, Cloud Run URL in prod)
PUBLIC_BASE_URL=https://your-service.run.app

# Fallbacks if Sheets are unreachable
FALLBACK_CIM_NAME=On-Call CIM
FALLBACK_CIM_PHONE=+00000000000
FALLBACK_CIM_EMAIL=
FALLBACK_STAKEHOLDER_DL=
```

---

## Google Sheets Schema

**Sheet 1 — Teams** (`GOOGLE_SHEET_ID_TECH`, range `A2:D`)

| display_name | dl_email | keywords_hint |
|---|---|---|
| Network | network-dl@company.com | network, connectivity, switch, wifi |

**Sheet 2 — OnCall CIM** (`GOOGLE_SHEET_ID_CIM`, range `A10:I`)

| name | email | phone | start_date | end_date | is_backup | backup_start | backup_end | timezone |
|---|---|---|---|---|---|---|---|---|
| Alice | alice@company.com | +1... | 01/07/2025 | 07/07/2025 | | | | |
| Bob | bob@company.com | +1... | 01/07/2025 | 07/07/2025 | yes | 20:00 | 8:00 | EST |

Dates in `dd/mm/yyyy`. Backup rows activate only within their time window.

**Sheet 3 — Fixed Members** (`GOOGLE_SHEET_ID_FIXED`, range `A2:C`)

| role | email | attendance_type |
|---|---|---|
| Stakeholder | stakeholder-dl@company.com | required |
| IT Manager | itmanager@company.com | optional |

---

## Database

LangGraph manages its own checkpoint tables via `checkpointer.setup()` on startup. You also need one application table:

```sql
CREATE TABLE incident_calls (
    thread_id   TEXT PRIMARY KEY,
    call_sid    TEXT,
    cim_name    TEXT,
    ticket_id   TEXT,
    priority    TEXT,
    description TEXT,
    eta         TEXT
);
```

---

## Running Locally

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Set up .env (see above)

# 3. Start the server
uvicorn main:app --reload --port 8000

# 4. For Twilio webhooks, expose locally with ngrok
ngrok http 8000
# Then set PUBLIC_BASE_URL=https://<your-ngrok-id>.ngrok.io in .env
```

---

## Deploying to Google Cloud Run

```bash
# Build and push the image
gcloud builds submit --tag gcr.io/YOUR_PROJECT/bridge-ops

# Deploy
gcloud run deploy bridge-ops \
  --image gcr.io/YOUR_PROJECT/bridge-ops \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-secrets="DATABASE_URL=DATABASE_URL:latest,OPENAI_API_KEY=OPENAI_API_KEY:latest,..."
```

Secrets (`.env` values, `token.pickle`, `service_account.json`) should be stored in **Google Secret Manager** and mounted at the paths the app expects:
- Service account JSON → `/secrets/sa/bridge-assistant-gsheet.json`
- Calendar token → `/secrets/cal/token.pickle`

> **Note on Calendar auth:** This project uses OAuth 2.0 (`token.pickle`) for the Calendar API, which authenticates as a personal Google account. This is intentional — the project is not tied to a Google Workspace organisation, so a service account (which would require domain-wide delegation to send invites on behalf of users) is not applicable here. For a production Workspace deployment, replace the OAuth token with a service account that has domain-wide delegation enabled.

---

## Key Design Notes

- **`autocommit=True`** on the DB pool is required — LangGraph's checkpointer manages its own transactions.
- **One worker** is used on Cloud Run (`--workers 1`). Scale via Cloud Run replicas, not in-process workers, because the async LangGraph graph is shared module-level state.
- The voice agent follows a **strict 5-step script** and writes `ETA_CONFIRMED:<eta>` as a sentinel in the transcript to signal a clean hang-up.
- The **Stakeholder DL is protected** — it is re-inserted automatically if an agent accidentally removes it during the approval edit.

---

## License

MIT
