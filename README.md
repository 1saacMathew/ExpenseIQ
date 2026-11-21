# ExpenseIQ — *Know your money. Own your future.*

A personal finance web app: record income and expenses, set budgets, explore analytics, export reports.
Flask + SQLite on the backend, vanilla JavaScript + Chart.js on the front end. Single-user, local-first.

> Screenshots: add yours to `docs/` (dashboard, transactions, budgets, analytics) and link them here.

## Features
- **Dashboard**: income, expenses, net balance, savings rate with comparison to the previous equivalent period; four live charts; recent transactions; budget alerts. Period filter: this month, last month, last 3 months, this year, custom range.
- **Transactions**: add / edit / delete (with confirmation), search, filter by category/type/date range, sort, pagination.
- **Categories**: 14 defaults plus custom ones (name, type, colour, icon). Deleting a category that has transactions requires choosing another category to move them to.
- **Budgets**: monthly limit per expense category, progress bars (green < 80%, amber 80–100%, red over), month navigation.
- **Analytics**: 7-day / month / year summaries, average per day, top category, largest expenses, rule-based insights (no AI API).
- **Reports**: CSV export (range or everything), PDF report (monthly or custom range) via ReportLab.
- **Settings**: currency (INR/USD/EUR, display only), light/dark theme, date format, CSV import, demo data (load/reset/remove), clear-all with typed confirmation.

## Tech stack
Python 3.10+, Flask, Flask-SQLAlchemy, Flask-WTF (CSRF), SQLite, ReportLab, Gunicorn (Linux hosting), Chart.js and Lucide (via CDN), pytest.

## Run locally on Windows (PowerShell)
```powershell
cd expenseiq
python -m venv venv
venv\Scripts\Activate.ps1          # if blocked: Set-ExecutionPolicy -Scope Process Bypass
pip install -r requirements.txt
copy .env.example .env              # then edit SECRET_KEY
python run.py
```
Open http://127.0.0.1:5000. The database (`instance/expenseiq.db`) is created automatically on first start, so there is no separate init step. For sample data go to **Settings → Load demo data**.
macOS/Linux: `python3 -m venv venv && source venv/bin/activate`, the rest is the same.

Charts and icons load Chart.js, Lucide and Inter from CDNs, so the browser needs internet for those.

## Configuration (`.env`)
| Variable | Purpose |
|---|---|
| `EXPENSEIQ_ENV` | `development` (default) or `production` |
| `SECRET_KEY` | signs sessions/CSRF tokens; required in production |
| `APP_USERNAME` / `APP_PASSWORD` | HTTP Basic login. **Required in production**: the app refuses to start without it |
| `DATABASE_URL` | optional SQLite URL, e.g. `sqlite:////var/data/expenseiq.db` |

Generate a key: `python -c "import secrets; print(secrets.token_hex(32))"`

## Tests
```
pytest
```
62 tests cover CRUD, filtering, categories, budgets, totals, savings rate, empty DB, CSV/PDF, import, date ranges, CSRF, auth, headers and invalid input.

## Project structure
```
config.py  run.py  requirements.txt  .env.example
app/__init__.py            app factory: config, CSRF, auth gate, security headers, error pages
app/models.py              Category, Transaction, Budget, Preference
app/routes/                one blueprint per page (dashboard, transactions, budgets, analytics, reports, settings)
app/services/finance.py    ALL metric definitions, periods, budgets, chart data, insights
app/services/money.py      exact money parsing/formatting
app/services/reporting.py  CSV + PDF
app/services/seed.py       default categories + fictional demo data
app/templates/ app/static/ Jinja templates, CSS design system, app.js, charts.js
tests/
```

## Architecture decisions
- **Money is integer minor units** (paise/cents). Text is parsed with a strict regex into integers (`parse_amount`); floats are only used to hand numbers to Chart.js for display.
- **Savings rate** = (income − expenses) / income × 100, shown as "—" when income is zero (`finance.savings_rate`).
- **Previous equivalent period** (`finance.resolve_period`): this month compares with the same days of last month, this year with the same dates last year, custom ranges with the immediately preceding range of equal length. Percent changes show "No earlier data" instead of dividing by zero.
- **Budgets** are one recurring limit per category, evaluated against the selected month's real transactions.
- **Demo data** is flagged (`is_demo`) so it can be removed without touching real entries.
- **Security**: CSRF on every POST, SQLAlchemy parameterised queries, Jinja auto-escaping, CSP and other headers, same-site-only redirects, CSV formula-injection guard on export, production refuses to start without a password.
- Importing/exporting amounts never converts currency.

## Deploying
Never publish real financial data in a public repo (`.gitignore` excludes `.env`, `instance/`, `*.db`).

**GitHub**: `git init`, `git add .`, `git commit -m "ExpenseIQ"`, create an empty repo on github.com, then `git remote add origin <url>` and `git push -u origin main`.

**Hosting** (I have not verified current free-tier terms; check each provider's docs before relying on this):
- *PythonAnywhere (free tier)*: Flask-friendly, file storage normally persists, so SQLite works. Set env vars in the WSGI file, and set `APP_PASSWORD`.
- *Render / Railway / Fly.io style hosts*: web service filesystems are typically **ephemeral** unless you attach a paid persistent disk/volume, so SQLite data would be lost on redeploy. Start command: `gunicorn -w 1 run:app`.
- Without a persistent disk you would need a hosted database such as PostgreSQL. That needs a driver and a code change (the models are portable, but the app is only tested on SQLite).

**SQLite limits**: one writer at a time, so use a single gunicorn worker; back up `instance/expenseiq.db` regularly.

## Known limitations / ideas
- Single user; auth is one shared HTTP Basic login, not multi-user accounts.
- PDFs show `Rs.` instead of ₹ because built-in PDF fonts lack the glyph.
- Ideas: recurring transactions, per-month budgets, multi-currency with exchange rates, PostgreSQL support.
