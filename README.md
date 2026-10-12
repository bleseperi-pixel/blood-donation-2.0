# LifeFlow: Blood Donation Platform with AI Demand Forecasting

A responsive web application for blood donor management, blood inventory tracking, and
AI-based demand forecasting. Built with Flask, NumPy and vanilla JavaScript, and configured
for deployment on Vercel.

## Features

- Donor registration, search, and eligibility checks (90-day donation gap)
- Blood compatibility finder: shows which donors can give to a given blood group
- Inventory tracking with a 42-day shelf life and FIFO stock allocation
- Hospital blood requests, fulfilled automatically from stock or left open when short
- AI demand forecasting for the next 24 hours, 7 days and 30 days (per blood group)
- Model accuracy reported against a "same as last week" baseline
- Recommendations: replenishment orders, donor requests, redistribution between
  compatible groups, and expiry alerts
- Responsive layout for mobile, tablet and desktop, with dark mode

## Project structure

```
lifeflow-vercel/
├── api/
│   └── index.py          # Vercel serverless entry point
├── lifeflow/             # Application package
│   ├── __init__.py       # create_app() factory
│   ├── config.py         # Configuration (environment-aware)
│   ├── constants.py      # Blood groups, compatibility rules
│   ├── db.py             # SQLite schema, connections, demo data
│   ├── errors.py         # JSON error handlers
│   ├── exceptions.py     # Domain exceptions
│   ├── routes.py         # Web page and REST API
│   ├── utils.py          # Shared helpers
│   ├── validators.py     # Input validation
│   └── services/
│       ├── forecasting.py  # Preprocessing, features, ridge model, forecasts
│       └── inventory.py    # Inventory engine and optimization engine
├── templates/index.html  # Single-page UI
├── static/               # CSS, JavaScript, images
├── tests/test_api.py     # API test suite
├── run.py                # Local development entry point
├── vercel.json           # Vercel build and routing configuration
├── requirements.txt      # Runtime dependencies
└── requirements-dev.txt  # Test dependencies
```

## Running locally

Requires Python 3.10 or newer.

```bash
python -m venv venv
source venv/bin/activate            # Windows: venv\Scripts\activate
pip install -r requirements-dev.txt
python run.py
```

Open http://127.0.0.1:5000. On first run, the app creates `lifeflow.db` in the project
folder and loads demo data.

Run the tests:

```bash
pytest
```

## Deploying to Vercel

1. Push this project to a GitHub repository. `.gitignore` keeps local databases and
   virtual environments out of the repository.
2. In Vercel, select **Add New → Project** and import the repository.
3. Leave the framework preset and build settings at their defaults. `vercel.json`
   already defines the Python build and routes.
4. Click **Deploy**.

No environment variables are required. The following optional variables are supported:

| Variable | Default | Purpose |
|---|---|---|
| `SEED_DEMO_DATA` | `true` | Load demo donors, stock and usage history into an empty database |
| `DB_PATH` | `/tmp/lifeflow.db` on Vercel | Location of the SQLite database file |

### Data persistence on Vercel

Vercel's serverless functions can only write to `/tmp`. This means:

- The database is recreated with demo data when a new instance starts.
- Donor and request records can disappear after a period of inactivity or a redeploy.

This is suitable for demonstrations and coursework. For real use, move the database to a
hosted service, such as Postgres (for example Neon or Supabase), and update `lifeflow/db.py`.

## API reference

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/health` | Health check |
| GET | `/api/stats` | Dashboard counters |
| GET | `/api/donors?group=&q=` | List and search donors |
| POST | `/api/donors` | Register a donor |
| DELETE | `/api/donors/<id>` | Remove a donor |
| GET | `/api/compatible/<group>` | Eligible donors who can give to a blood group |
| GET | `/api/inventory` | Stock totals and batches |
| POST | `/api/donations` | Record collected units |
| GET | `/api/requests` | Recent hospital requests |
| POST | `/api/requests` | Submit a hospital request |
| GET | `/api/forecast/<group>` | 24h, 7-day and 30-day forecast |
| GET | `/api/recommendations` | Actions for all blood groups |

Errors are returned as JSON with an `error` field and an appropriate HTTP status code
(400 for invalid input, 404 for unknown resources, 422 when there is not enough data to forecast,
500 for unexpected server errors).

## How the forecasting works

1. **Preprocessing:** usage is grouped by day per blood group, and missing days are filled with zero.
2. **Features:** the previous day, the same weekday last week, 7- and 14-day averages, the
   14-day standard deviation, and a one-hot weekday encoding.
3. **Model:** standardised ridge regression, trained separately for each blood group.
4. **Validation:** the last 14 days are held out. The model's error is reported alongside the
   "same as last week" baseline, so you can see whether the model improves on a simple rule.
5. **Forecast:** a recursive 30-day forecast, where each predicted day is used as input for the next.

## Configuration

Settings live in `lifeflow/config.py`. Key values:

- `DONATION_GAP_DAYS = 90`: minimum days between donations
- `SHELF_LIFE_DAYS = 42`: usable life of a blood unit
- `SAFETY_BUFFER = 1.10`: stock target above forecast 30-day demand
- `EXCESS_BUFFER = 1.20`: stock above this level is reported as excess
- `RIDGE_ALPHA = 1.0`: regularisation strength

## Disclaimer

This is a demonstration system. Real blood banks must follow national blood safety
regulations, and all clinical decisions must be made by qualified medical staff.
