# AI Coding Agent Instructions — Contract Expiry Tracker

## Project Overview
Contract Expiry Tracker is a lightweight Flask app for managing contract lifecycle dates and automated
alerts. Key objectives:

- centralized contract database (SQLite default)
- status calculation and dashboard
- daily expiry scheduler with email reminders
- notification audit/log
- optional AI features (extract, summarize, risk)

All AI/LLM logic lives in `app/ai_service.py` so the core application remains unaffected by provider changes.

## Critical rules
- **Status calculation** (single source of truth):
  ```python
  if end_date < today:
      status = "Expired"
  elif end_date <= today + 30 days:
      status = "Critical"
  elif end_date <= today + 90 days:
      status = "Expiring Soon"
  else:
      status = "Active"
  ```
- **Notification cadence**: send reminders at exactly 90, 60, 30, and 7 days before expiry.
- **Idempotency**: check `NotificationLog` for `(contract_id, days_before, sent_date)` before sending.

## Architecture & Data Flow
- **App factory** (`app/__init__.py`): config, DB (SQLAlchemy), migrations, APScheduler.
- **Models** (`app/models.py`): `Contract` (metadata + AI fields) and `NotificationLog` (audit trail).
- **Routes** (`app/routes.py`): dashboard, CRUD, file upload (possible AI call).
- **Scheduler** (`app/scheduler.py`): `run_daily_check` updates statuses and triggers notifications.
- **Email** (`app/email_service.py`): compose/send emails, write `NotificationLog` entries.
- **AI service** (`app/ai_service.py`): extract, summarize, risk; interface is provider-agnostic.

## Developer workflows
### Setup & run
```bash
python -m venv venv
# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate
pip install -r requirements.txt
flask db upgrade   # if migrations exist
python run.py
```
Run tests: `pytest`.

### Adding features
- **New contract field**: update model, create migration (`flask db migrate`), adjust form/template, add tests.
- **Modify notifications**: edit scheduler logic, update `NotificationLog` checks, add scheduler tests.
- **AI functionality**: implement in `app/ai_service.py`, call from routes or scheduler, test with mocked LLM.

## Testing guidance
- Mock SMTP/AI API calls.
- Test status boundaries and scheduler triggers ([90,60,30,7] days). 
- Ensure no duplicate log entries.

## Configuration
`config.py` loads `.env` using python-dotenv. Key vars: `SECRET_KEY`, `DATABASE_URL`, `MAIL_SERVER`, `MAIL_PORT`, `MAIL_USERNAME`, `MAIL_PASSWORD`, `AI_MODE`, `AI_API_KEY`.

## Agent behaviour
- Follow core rules unless changes are explicitly requested.
- Prefer small, focused commits with tests and migrations.
- Keep AI interactions behind config flags.


*Feel free to ask for example PR prompts or template snippets; adapt this file as the project evolves.*