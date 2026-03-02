
**1- Repository Structure (proposed)**

contract-expiry-tracker/
│
├── app/                          # Main application package (backend logic)
│   ├── __init__.py               # App factory, config loading, DB & scheduler initialization
│   ├── models.py                 # Database models (Contract, NotificationLog)
│   ├── routes.py                 # HTTP routes (dashboard, lists, CRUD, upload)
│   ├── scheduler.py              # Daily job – expiry checks, status updates, notifications
│   ├── email_service.py          # Email sending logic and logging
│   ├── ai_service.py             # (Optional) AI logic – extraction, risk scoring, summaries
│   └── utils.py                  # Shared helper functions (status calculation, date utilities)
│
├── static/                       # Frontend assets
│   ├── css/
│   │   └── styles.css            # Styling of dashboard, tables, timeline, forms
│   └── js/
│       └── dashboard.js          # Interactive UI (filters, confirmations, simple AJAX)
│
├── templates/                    # Jinja2 HTML templates
│   ├── base.html                 # Base layout (nav, footer, CSS/JS includes)
│   ├── dashboard.html            # Main overview (cards, charts, timeline)
│   ├── contracts.html            # Contract list with filters & sorting
│   ├── contract_detail.html      # Contract detail view (with optional AI data)
│   ├── add_contract.html         # Create/edit contract form
│   ├── notifications.html        # Notification log view
│   └── upload_contract.html      # (Optional) Upload PDF/DOCX for AI extraction
│
├── sample_data/
│   └── contracts_sample.csv      # Sample contracts data for demo/testing
│
├── tests/                        # Automated tests
│   ├── test_models.py            # Model & status logic tests
│   ├── test_routes.py            # Route tests (responses, redirects, validation)
│   ├── test_scheduler.py         # Scheduler & notification logic tests
│   └── test_ai_service.py        # (Optional) AI service tests (with mocked LLM responses)
│
├── instance/
│   └── contracts.db              # SQLite database (ignored by Git, generated at runtime)
│
├── config.py                     # Application configuration (DB, email, AI endpoint, etc.)
├── run.py                        # Entry point – runs the Flask application
├── requirements.txt              # Python dependencies
├── .env.example                  # Example environment variables (EMAIL, AI_API_URL, etc.)
└── README.md                     # Project documentation & specification


**2-What each part of the code is supposed to do**
This is your “mental model” / spec: what you expect from each file.
run.py
Responsibility: Start the web application.

Imports create_app from app.__init__.
Creates the Flask app instance.
Runs the development server (app.run()).
No business logic here — just the entry point.


config.py
Responsibility: Central configuration.
Contains configuration classes / constants, for example:

SQLALCHEMY_DATABASE_URI – points to instance/contracts.db.
SECRET_KEY – Flask secret key.
Email settings:

MAIL_SERVER, MAIL_PORT, MAIL_USERNAME, MAIL_PASSWORD, MAIL_USE_TLS/SSL.


AI settings (optional, but useful):

AI_MODE – "none" | "local" | "api".
AI_API_URL, AI_API_KEY – if using a cloud LLM.


Debug vs production flags.

You can have:
class Config:
    SQLALCHEMY_DATABASE_URI = "sqlite:///contracts.db"
    # ...

class DevConfig(Config):
    DEBUG = True

class ProdConfig(Config):
    DEBUG = False


app/__init__.py
Responsibility: Create and wire the application.
Implements the application factory:

Creates Flask instance.
Loads configuration from config.py.
Initializes:

db = SQLAlchemy(app)
migrate = Migrate(app, db)
Scheduler (APScheduler) and attaches run_daily_check from scheduler.py.


Registers Blueprints / routes from routes.py.

This is where everything comes together:

configuration,
database,
routes,
background jobs.


app/models.py
Responsibility: Define database schema / ORM models.
Contract model
Fields (roughly according to your earlier spec): 
class Contract(db.Model):
    id = db.Column(db.Integer, primary_key=True)

    name = db.Column(db.String(255), nullable=False)
    supplier = db.Column(db.String(255), nullable=False)
    category = db.Column(db.String(50), nullable=False)  # CAPEX / OPEX / Services / ...

    value = db.Column(db.Float)
    currency = db.Column(db.String(3), default="EUR")

    start_date = db.Column(db.Date)
    end_date = db.Column(db.Date)         # key field for expiry
    notice_period = db.Column(db.Integer, default=90)

    contact_email = db.Column(db.String(255), nullable=False)
    notes = db.Column(db.Text)

    status = db.Column(db.String(20))     # Active / Expiring Soon / Critical / Expired

    # AI-related, optional but future-proof:
    risk_score = db.Column(db.Integer)        # 1–5
    risk_category = db.Column(db.String(20))  # Low / Medium / High
    ai_summary = db.Column(db.Text)           # business-readable contract summary
    ai_recommendation = db.Column(db.Text)    # “next steps” for procurement

    created_at = db.Column(db.DateTime)
    updated_at = db.Column(db.DateTime)

NotificationLog model 

class NotificationLog(db.Model):
    id = db.Column(db.Integer, primary_key=True)

    contract_id = db.Column(db.Integer, db.ForeignKey("contract.id"))
    sent_at = db.Column(db.DateTime)
    days_before = db.Column(db.Integer)       # 90 / 60 / 30 / 7, etc.
    recipient = db.Column(db.String(255))
    status = db.Column(db.String(20))         # Sent / Failed
    error_message = db.Column(db.Text)        # optional, for debugging

This is the single source of truth for what the system stores. 

This is the single source of truth for what the system stores.
if end_date < today:
    return "Expired"
elif end_date <= today + 30 days:
    return "Critical"
elif end_date <= today + 90 days:
    return "Expiring Soon"
else:
    return "Active"


days_until(date, today).


Simple validation/conversion helpers.


Anything that is used both by routes.py and scheduler.py.

app/email_service.py
Responsibility: Email sending and logging.
Typical functions:

def send_expiry_notification(contract, days_before):
    """
    Build and send an email about a contract that is expiring in `days_before` days.
    Use contract.contact_email as recipient.
    Log the result into NotificationLog.
    """

Composes subject and body text.
Uses Flask-Mail, smtplib, or any other email library.
Writes a NotificationLog entry (success/failure).

Later you can integrate AI here as well (e.g., include ai_summary and ai_recommendation in the email body).

app/scheduler.py
Responsibility: Daily expiry check and notifications.
Implements something like run_daily_check():

Load all contracts from DB.
For each contract:

Calculate status via utils.compute_status.
Update contract.status.
Compute days until end_date.
If days until end_date is in [90, 60, 30, 7], call send_expiry_notification(contract, days_before).


Optionally, trigger ai_service.assess_contract_risk() for certain contracts (e.g., high value, soon expiring).
Commit DB changes.

This function is scheduled once per day (e.g., at 08:00) via APScheduler configured in __init__.py.

app/ai_service.py (AI-ready, but can start as stubs)
Responsibility: All AI / LLM related logic.
Suggested functions:
def extract_contract_data(file_bytes, filename):
    """
    Input: raw file bytes (PDF/DOCX) and filename.
    Output: dict with structured contract data:
    { name, supplier, category, value, currency, start_date, end_date, notice_period, ... }
    """

def generate_ai_summary(contract_text):
    """
    Input: plain text of the contract.
    Output: short, business-focused summary.
    """

def assess_contract_risk(contract, contract_text=None):
    """
    Input: Contract object (+ optional raw text).
    Output: (risk_score, risk_category, ai_recommendation)
    """

For now you can implement these as:

“dummy” functions returning placeholder data, or
wrap a local LLM (Ollama) or a cloud API (OpenAI/Azure) — depending on what you want and what’s free.

The key idea: all AI complexity is isolated here, so the rest of the app does not depend on LLM implementation details.

app/routes.py
Responsibility: HTTP routes and views.
Typical routes:


GET / or /dashboard – show the main dashboard:

total contracts, active, expiring, expired, critical;
list of upcoming expiries;
maybe a timeline or chart.



GET /contracts – list contracts (filters, sorting).


GET /contracts/<id> – show a contract detail:

base information,
status,
optional AI info (summary, risk, recommendation),
notification history.



GET /contracts/add + POST /contracts/add – create a new contract.


GET /contracts/<id>/edit + POST /contracts/<id>/edit – edit contract.


POST /contracts/<id>/delete – delete contract (with confirmation).


GET /contracts/upload + POST /contracts/upload:

upload a contract file,
call ai_service.extract_contract_data() to pre-fill fields,
show preview form before saving.



Everything here is pure HTTP/web logic – it calls into models, utils, ai_service, email_service as needed.

templates/
Responsibility: UI layer (Jinja2).

base.html – main layout, nav, messages.
dashboard.html – overview cards + tables + optional charts.
contracts.html – table of contracts.
contract_detail.html – detail info; optionally show AI summary & recommendation.
add_contract.html – form for create/update.
notifications.html – list of NotificationLog entries.
upload_contract.html – upload form for contract files.


static/css/styles.css and static/js/dashboard.js

Styles and JS for a decent UX.
JS might handle:

filter toggles,
confirmation modals (delete),
simple AJAX refresh for parts of the dashboard (optional).




sample_data/contracts_sample.csv

A small CSV with fake data so you can quickly bootstrap a demo DB.
Useful for tests and for showing the project to others.


tests/
Write tests at least for:

Models & status logic: making sure compute_status and date computations behave correctly.
Routes: that important routes respond, protect against invalid data, etc.
Scheduler logic: that notifications are sent on the right days.
AI service: with mocked LLM responses so tests are deterministic.

**3-README.md draft (in English, spec-style)**
to be modified.
# Contract Expiry Tracker (AI-ready)

A web application for managing contracts, tracking expiry dates, and sending automated email alerts.

Designed for procurement / legal / facilities teams that need a clear overview of:

- which contracts are expiring soon,
- which contracts are already expired,
- what the priority and risk of each contract is,
- which actions should be taken (renew, renegotiate, terminate).

The system is **AI-ready**: it is structured so that you can easily plug in
LLM-based services later for:

- extracting structured data from PDF/DOCX contracts,
- generating human-readable summaries,
- scoring risk and suggesting next actions.

---

## 🧠 High-level logic

1. **User creates or uploads a contract**

   - Either by manually filling in a web form, or  
   - by uploading a contract file (PDF/DOCX) that can be processed by an AI service
     (`app/ai_service.py`) to pre-fill key fields.

2. **Contract is stored in the database**

   The `Contract` model stores all important metadata:

   - Name, supplier, category, value, currency
   - Start and end date
   - Notice period (how many days before end date to start getting alerts)
   - Contact email (who should receive notifications)
   - Status (Active / Expiring Soon / Critical / Expired)
   - (Optional) AI fields:
     - `risk_score`, `risk_category`
     - `ai_summary`, `ai_recommendation`

3. **Daily scheduler checks for upcoming expiries**

   A scheduled job (APScheduler) runs once per day (e.g. 08:00) and:

   - Recalculates the **status** of each contract based on its `end_date`.
   - Finds contracts that are exactly 90 / 60 / 30 / 7 days before expiry.
   - Sends email notifications to the assigned contacts.
   - Logs each notification in `NotificationLog`.

4. **Dashboard shows current situation**

   The dashboard view provides:

   - KPIs (total contracts, active, expiring, critical, expired)
   - A table of upcoming expiries
   - Filters by supplier, category, status
   - Optional timeline / Gantt-style view
   - Access to contract detail pages and notification history

5. **AI layer (optional)**

   If configured, the `ai_service` can:

   - Extract structured data from uploaded contract files,
   - Generate summaries of contracts,
   - Compute a risk score and suggest next steps.

---

## 📂 Project structure

```text
contract-expiry-tracker/
├── app/
│   ├── __init__.py        # App factory: config, DB, scheduler
│   ├── models.py          # Contract + NotificationLog models
│   ├── routes.py          # HTTP routes and views
│   ├── scheduler.py       # Daily expiry check and notifications
│   ├── email_service.py   # Email sending logic
│   ├── ai_service.py      # (Optional) AI integration
│   └── utils.py           # Shared helpers (status, dates, etc.)
├── static/
│   ├── css/styles.css
│   └── js/dashboard.js
├── templates/
│   ├── base.html
│   ├── dashboard.html
│   ├── contracts.html
│   ├── contract_detail.html
│   ├── add_contract.html
│   ├── notifications.html
│   └── upload_contract.html
├── sample_data/contracts_sample.csv
├── tests/
│   ├── test_models.py
│   ├── test_routes.py
│   ├── test_scheduler.py
│   └── test_ai_service.py
├── instance/contracts.db
├── config.py
├── run.py
├── requirements.txt
└── README.md

🗃️ Data model
Contract
Core contract metadata:

id – primary key
name – contract name (e.g. “Leasing CNC Machines 2024–2027”)
supplier – supplier name
category – CAPEX / OPEX / Services / Maintenance / IT / Other
value – monetary value
currency – currency code (CZK / EUR / USD / …)
start_date – contract start date
end_date – contract end date (key field for expiry logic)
notice_period – how many days before end_date to start alerting (default: 90)
contact_email – who should receive notifications
notes – free text notes

Status and AI-related fields:

status – Active / Expiring Soon / Critical / Expired (auto-calculated)
risk_score – integer, e.g. 1–5 (optional)
risk_category – Low / Medium / High (optional)
ai_summary – concise summary generated by AI (optional)
ai_recommendation – recommended action / comments from AI (optional)

Meta fields:

created_at, updated_at – timestamps

NotificationLog
Every time an email alert is sent (or fails), a log entry is stored:

id – primary key
contract_id – foreign key referencing Contract
sent_at – timestamp when email was sent
days_before – how many days before expiry this notification was triggered (90 / 60 / 30 / 7)
recipient – email address that received the notification
status – Sent / Failed
error_message – optional error details if sending failed


Status and notification logic
The status of a contract is calculated based on end_date:
if end_date < today:
    status = "Expired"
elif end_date <= today + 30 days:
    status = "Critical"
elif end_date <= today + 90 days:
    status = "Expiring Soon"
else:
    status = "Active"

Daily scheduler (run_daily_check)

Fetch all contracts from the database.
For each contract:

Recalculate status using the rule above.
Compute days_until_end = (end_date - today).days.
If days_until_end is exactly one of [90, 60, 30, 7]:

Call send_expiry_notification(contract, days_before=days_until_end).




Save all changes and notification logs.

This ensures users receive multiple reminders: early (90 days), then 60, 30, and 7 days before expiry.

🤖 AI integration (optional but supported by design)
All AI-related logic lives in app/ai_service.py.
The intention is to keep the core app independent from any specific LLM provider.
Suggested functions:


extract_contract_data(file_bytes, filename)


Input: raw file contents (PDF/DOCX) and filename.


Output: Python dict with keys matching Contract fields:

{
  "name": "...",
  "supplier": "...",
  "category": "...",
  "value": 123456,
  "currency": "EUR",
  "start_date": "2024-03-01",
  "end_date": "2027-02-28",
  "notice_period": 90,
  "contact_email": "someone@example.com"
}

generate_ai_summary(contract_text)

Input: plain text contract.
Output: short summary focusing on key business terms, obligations and risks.



assess_contract_risk(contract, contract_text=None)

Input: Contract object + optional raw text.
Output: (risk_score, risk_category, ai_recommendation).



These functions may:

Call a local open-source model (e.g. via an HTTP endpoint), or
Call a cloud LLM (OpenAI/Azure/…).

Routes can call ai_service:

On upload (/contracts/upload) to pre-fill fields.
In scheduler for high-value contracts.
On contract detail page to refresh summaries/recommendations.


🚀 How to run the project (basic flow)


Clone the repository
git clone https://github.com/<user>/contract-expiry-tracker.git
cd contract-expiry-tracker
``
Create virtual environment
python -m venv venv
source venv/bin/activate      # Linux/Mac
# or
venv\Scripts\activate         # Windows

Install dependencies
pip install -r requirements.txt

Initialize the database
If using Flask-Migrate:
flask db upgrade


Or run a small script to create tables from models.py.


Run the app
python run.py

Tests
Run tests via:
pytest

Tests cover:

Data model and status logic (test_models.py)
Route behavior (test_routes.py)
Scheduler and notification logic (test_scheduler.py)
AI functions with mocked responses (test_ai_service.py)


🔮 Possible extensions

Power BI integration / export of aggregated contract data
Power Platform integration (Power Automate flows for further approvals)
Role-based access control (different views for Legal / Procurement / Finance)
Multi-tenant support (multiple companies in one instance)


