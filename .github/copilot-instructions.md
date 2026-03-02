# AI Coding Agent Instructions for Contract Expiry Tracker

## Project Overview
A Flask web application for tracking contract expiry dates and sending automated email notifications. **AI-ready architecture**: all LLM logic isolated in `app/ai_service.py` to support contract data extraction, summarization, and risk scoring without coupling to the core app.

## Architecture & Data Flow

### Core Components
- **App Factory** (`app/__init__.py`): Initializes Flask, SQLAlchemy, Flask-Migrate, and APScheduler daily job
- **Models** (`app/models.py`): `Contract` (metadata + AI fields) and `NotificationLog` (audit trail)
- **Routes** (`app/routes.py`): Dashboard, CRUD operations, file uploads
- **Scheduler** (`app/scheduler.py`): Daily `run_daily_check()` recalculates status, triggers notifications
- **Email** (`app/email_service.py`): Composes and sends expiry notifications; logs results
- **AI Service** (`app/ai_service.py`): Stub functions for extract, summarize, risk assessment (plugable with any LLM)

### Status Calculation Logic (Critical)
```python
# Used in scheduler and route filters
if end_date < today: status = "Expired"
elif end_date <= today + 30 days: status = "Critical"
elif end_date <= today + 90 days: status = "Expiring Soon"
else: status = "Active"
```
This logic is the "heartbeat"—scheduler recalculates daily; notifications trigger at [90, 60, 30, 7] days.

### Data Model Key Fields
- **Contract**: `id, name, supplier, category, value, currency, start_date, end_date, notice_period, contact_email, status, risk_score, risk_category, ai_summary, ai_recommendation, created_at, updated_at`
- **NotificationLog**: `id, contract_id, sent_at, days_before, recipient, status, error_message` (each alert logged for audit)

## Developer Workflows

### Local Setup
```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
flask db upgrade          # Initialize DB with migrations
python run.py             # Start dev server
```

### Database
- SQLAlchemy ORM, Flask-Migrate for schema versioning
- Database URI and secrets in `config.py` (loaded from `.env` via `python-dotenv`)
- `instance/contracts.db` (SQLite) is gitignored; created at runtime

### Tests
```bash
pytest
```
- `test_models.py`: Status calculation, date logic, validation
- `test_routes.py`: Form submissions, redirects, error handling
- `test_scheduler.py`: Daily check logic, notification triggering
- `test_ai_service.py`: AI functions with mocked LLM responses

## Key Conventions

### Status & Notification Flow
- Scheduler runs daily (configured in `__init__.py`); updates all contract statuses in one pass
- Notifications are **idempotent**: checking `NotificationLog` prevents duplicate alerts on same day
- `notice_period` field controls when first notification triggers (default 90 days)

### AI Integration Points (Design Pattern)
All AI is in `app/ai_service.py`; core routes/scheduler call it optionally:
- **Extract** (`extract_contract_data`): File upload → pre-fill contract form
- **Summarize** (`generate_ai_summary`): Contract text → business-readable summary in `Contract.ai_summary`
- **Risk** (`assess_contract_risk`): Contract object → score, category, recommendation

AI service can call local LLM (Ollama) or cloud API (OpenAI/Azure). Routes decide when to invoke; AI service remains provider-agnostic.

## Project-Specific Patterns

### Configuration Management
- `config.py` defines `Config` class (base, extendable for DevConfig, ProdConfig)
- Environment variables: `SECRET_KEY`, `DATABASE_URL`, `MAIL_*`, `AI_MODE`, `AI_API_*`
- `.env.example` documents required variables

### File Structure Rationale
- `templates/` uses Jinja2; `base.html` is parent template
- `static/css/styles.css` and `static/js/dashboard.js` for UI/interactivity
- `sample_data/contracts_sample.csv` for quick bootstrapping and demos

### External Dependencies
- **Flask**: Web framework, routing, templating
- **Flask-SQLAlchemy**: ORM
- **Flask-Migrate**: Database schema versioning
- **APScheduler**: Background task scheduling (daily cron-like jobs)
- **openpyxl**: Excel file parsing (contract upload feature)
- **python-dotenv**: Environment variable loading

## Common Tasks

### Adding a New Contract Field
1. Update `Contract` model in `app/models.py`
2. Create migration: `flask db migrate -m "Add field_name"`
3. Update `add_contract.html` form and `routes.py` POST handler
4. Add test cases in `test_models.py`

### Modifying Notification Logic
- Core logic: `scheduler.py` `run_daily_check()` + status calculation in `utils.py`
- Email composition: `email_service.py` (include AI summaries if available)
- Always check `NotificationLog` to prevent duplicates

### Implementing AI Feature
1. Define stub function in `ai_service.py` with clear input/output contracts
2. Integrate call site in `routes.py` (e.g., upload handler) or `scheduler.py` (batch processing)
3. Test with mocked responses in `test_ai_service.py`
4. Document fallback behavior if AI service is disabled/unavailable

## Notes for AI Agents
- **Isolation principle**: Keep business logic (scheduling, notifications, status) separate from AI implementation; changes to LLM provider should not affect core app flow
- **Idempotency**: Scheduler runs daily; ensure notifications don't duplicate and status updates are safe to re-run
- **Testing**: Mock external calls (email, AI APIs) in tests; use `fixtures` for sample data
- **Configuration**: Use `config.py` for deployment-wide settings; `.env` for secrets
