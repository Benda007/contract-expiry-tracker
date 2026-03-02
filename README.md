# Contract Expiry Tracker (AI‑Ready)

A lightweight, extensible **contract lifecycle management** web application built with Flask.  
It helps procurement, legal, facility management, and finance teams keep track of:

- upcoming contract expiries  
- expired or critical contracts  
- notification history  
- (optional) AI‑generated summaries, risk scores, and recommendations  

The system is **AI‑ready**: an optional AI layer can extract structured data from files, summarize contracts, and assess risk — without interfering with the core application logic.

---

## 🚀 Features

### ✔️ Core Features
- Centralized contract database (SQLite by default)
- Automatic status calculation (Active / Expiring Soon / Critical / Expired)
- Daily background scheduler for expiry checks
- Email notifications at 90 / 60 / 30 / 7 days before expiry
- Full CRUD interface
- Dashboard with KPIs, filters, sorting
- Notification logs for audit/history

### 🤖 Optional AI Features
- Extract metadata from PDF/DOCX contract files
- Human‑readable summaries
- Risk scoring + recommended next steps

The AI layer is fully isolated in `app/ai_service.py`.

---

## 🧠 System Overview

### 1. Users create or upload a contract
- Manual form entry, **or**
- Upload a contract file → AI extraction pre‑fills fields

### 2. Contract stored in the database
The model stores:
- core metadata  
- dates + notice period  
- contact email  
- computed status  
- optional AI fields  

### 3. Daily scheduler processes expiries
Runs once per day (e.g., 08:00) and:
- recalculates all statuses  
- checks for 90 / 60 / 30 / 7‑day thresholds  
- sends email alerts  
- writes `NotificationLog`  

### 4. Dashboard shows the current state
Includes KPIs, expiring contracts, filters, timeline view, etc.

### 5. AI layer (optional)
Functions for:
- data extraction  
- summaries  
- risk assessment  

---

## 📂 Repository Structure

```text
contract-expiry-tracker/
│
├── app/
│   ├── __init__.py        # App factory: config, DB, scheduler
│   ├── models.py          # ORM models (Contract, NotificationLog)
│   ├── routes.py          # HTTP endpoints (dashboard, CRUD, upload)
│   ├── scheduler.py       # Daily expiry check + notifications
│   ├── email_service.py   # Email sending + logging
│   ├── ai_service.py      # (Optional) LLM logic
│   └── utils.py           # Helpers (status calculation, date utils)
│
├── static/
│   ├── css/styles.css     # UI styles
│   └── js/dashboard.js    # Filters, UI interactions
│
├── templates/
│   ├── base.html
│   ├── dashboard.html
│   ├── contracts.html
│   ├── contract_detail.html
│   ├── add_contract.html
│   ├── notifications.html
│   └── upload_contract.html
│
├── sample_data/contracts_sample.csv
│
├── tests/
│   ├── test_models.py
│   ├── test_routes.py
│   ├── test_scheduler.py
│   └── test_ai_service.py
│
├── instance/
│   └── contracts.db       # Runtime DB (gitignored)
│
├── config.py
├── run.py                 # Entry point
├── requirements.txt
└── README.md
```

---

## 🗃️ Data Model

### Contract
Core fields:
- `name`, `supplier`, `category`
- `value`, `currency`
- `start_date`, `end_date`
- `notice_period`
- `contact_email`
- `notes`

Computed / optional:
- `status` → Active / Expiring Soon / Critical / Expired  
- `risk_score`, `risk_category`
- `ai_summary`, `ai_recommendation`

Metadata:
- `created_at`, `updated_at`

---

### NotificationLog
Logged for every notification:
- `contract_id`
- `sent_at`
- `days_before`
- `recipient`
- `status` (Sent / Failed)
- `error_message` (nullable)

---

## 🔔 Status & Notification Logic

Status is computed as:

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

Notifications trigger when:

```
days_until_expiry ∈ {90, 60, 30, 7}
```

Each email generates a `NotificationLog` entry.

---

## 🤖 AI Integration (Optional)

Implemented in `app/ai_service.py`.

### `extract_contract_data(file_bytes, filename)`
Extract structured metadata from a PDF/DOCX contract.

### `generate_ai_summary(contract_text)`
Produce a business-readable summary.

### `assess_contract_risk(contract, contract_text=None)`
Return:
- risk score (1–5)
- risk category
- recommended next steps

Can use:
- local LLMs (Ollama, LM Studio)
- cloud LLMs (Azure, OpenAI)

---

## 🛠️ Running the Project

### 1. Clone the repository
```bash
git clone https://github.com/<user>/contract-expiry-tracker.git
cd contract-expiry-tracker
```

### 2. Create a virtual environment
```bash
python -m venv venv
source venv/bin/activate   # Linux/Mac
venv\Scripts\activate      # Windows
```

### 3. Install dependencies
```bash
pip install -r requirements.txt
```

### 4. Initialize the database

With Flask‑Migrate:
```bash
flask db upgrade
```

Or generate tables directly using `models.py`.

### 5. Run the application
```bash
python run.py
```

---

## 🧪 Tests

Run all tests:
```bash
pytest
```

Test coverage includes:
- model & status logic  
- API/route behavior  
- scheduler + notifications  
- AI layer with mocked LLM responses  

---

## 🔮 Future Extensions

- Power BI dashboards  
- Power Automate approval workflows  
- Role‑based access control  
- Multi‑tenant support  
- Integration with ERP / procurement platforms  