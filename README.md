# TodoApp — Backend

Django REST Framework API for the TodoApp productivity dashboard, designed to pair with
the React frontend and a MySQL database in production.

## Tech Stack
- Django 6.0 + Django REST Framework
- MySQL 8 (default, local + production) / SQLite (optional local fallback)
- SimpleJWT (JWT authentication)
- django-cors-headers, django-filter

## Project Structure

```
todoapp-backend/
├── config/
│   ├── settings/
│   │   ├── base.py          # shared settings (DRF, JWT, CORS, MySQL config)
│   │   ├── development.py   # DEBUG=True, SQLite by default
│   │   └── production.py    # DEBUG=False, security headers, SMTP email
│   ├── urls.py               # mounts every app's urls.py under /api/
│   ├── wsgi.py / asgi.py
├── apps/
│   ├── accounts/              # custom User model, auth, profile, settings
│   ├── tasks/                 # Task, Subtask, Tag — the core CRUD
│   ├── dashboard/              # aggregated stats endpoint (no models)
│   ├── kanban/                 # Board/Column/Card + drag-drop move endpoint
│   ├── calendar_app/           # calendar events (merges with Task due dates)
│   ├── notifications/           # in-app notifications, auto-fired via signals
│   ├── analytics/                # computed stats endpoint (no models)
│   ├── team/                      # team membership + invites
│   └── activity_logs/              # audit trail, auto-logged via signals
├── core/                            # pagination, permissions, exception handler, mixins
├── requirements/                     # base / development / production
├── sql/seed_data.sql                  # optional manual seed reference
├── media/, static/, templates/
└── manage.py
```

## Getting Started (local dev — MySQL, matches production)

This project is configured to use MySQL by default in development (`USE_MYSQL=True` in
`.env`), so your local setup matches production. `mysqlclient` ships in
`requirements/base.txt`, so it installs automatically with either requirements file.

1. Create the database in MySQL 8:
   ```sql
   CREATE DATABASE todoapp_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```
2. Set up the virtualenv and install dependencies:
   ```bash
   python -m venv venv
   venv\Scripts\activate            # macOS/Linux: source venv/bin/activate
   pip install -r requirements/development.txt
   ```
   > **Windows note:** `mysqlclient` installs from a prebuilt wheel for most supported
   > Python versions — no extra setup needed. If `pip install` tries to build from source
   > (rare, usually on a brand-new Python release), install the
   > [MySQL Connector C / VS Build Tools](https://github.com/PyMySQL/mysqlclient#windows)
   > first, or install from an unofficial wheel.
3. Copy `.env.example` to `.env` and fill in your MySQL credentials:
   ```bash
   copy .env.example .env          # macOS/Linux: cp .env.example .env
   ```
   ```
   USE_MYSQL=True
   DB_NAME=todoapp_db
   DB_USER=root
   DB_PASSWORD=your_mysql_root_password
   DB_HOST=127.0.0.1
   DB_PORT=3306
   ```
4. Migrate and run:
   ```bash
   python manage.py migrate
   python manage.py createsuperuser
   python manage.py runserver
   ```

API is live at `http://127.0.0.1:8000/api/`. Admin at `http://127.0.0.1:8000/admin/`.

## Falling back to SQLite (optional, zero setup)

If you want to develop without a running MySQL instance, set `USE_MYSQL=False` in `.env`.
`development.py` then swaps in a local `db.sqlite3` file automatically — no other changes
needed.

For production, run with `DJANGO_SETTINGS_MODULE=config.settings.production` (already the
default in `wsgi.py`/`asgi.py`) and install `requirements/production.txt`, which pulls in
`requirements/base.txt` (Django, DRF, `mysqlclient`, ...) plus `gunicorn` and `whitenoise`.

## Running Tests

Because `apps/` is itself an importable package, run tests scoped to app labels
(not bare `manage.py test`, which double-discovers each app under two import paths):

```bash
python manage.py test accounts tasks dashboard kanban calendar_app notifications analytics team activity_logs
```

All 9 apps currently pass (10 tests total).

## API Reference

Base URL: `/api/`

| Endpoint | Method | Description |
|---|---|---|
| `/auth/register/` | POST | Create account, returns user + JWT tokens |
| `/auth/login/` | POST | Login, returns user + JWT tokens |
| `/auth/logout/` | POST | Blacklists the refresh token |
| `/accounts/profile/` | GET, PUT | View/update current user's profile |
| `/accounts/change-password/` | POST | Change password |
| `/tasks/` | GET, POST | List (filterable: `status`, `priority`, `category`, `due_before`, `due_after`, `search`) / create |
| `/tasks/{id}/` | GET, PUT, DELETE | Retrieve / update / delete a task |
| `/dashboard/summary/` | GET | Stats + today's tasks + weekly completion chart |
| `/kanban/board/` | GET | Current user's board (auto-creates 3 default columns) |
| `/kanban/cards/{id}/move/` | PATCH | Move a card to a new column (syncs Task.status) |
| `/calendar/events/?month=&year=` | GET | Calendar events + task due dates for a month |
| `/notifications/?unread=true` | GET | List notifications |
| `/notifications/{id}/read/` | PATCH | Mark as read |
| `/analytics/summary/` | GET | Completion rate, weekly/monthly counts, avg completion time |
| `/team/members/` | GET, POST | List members / invite a new member |
| `/activity-logs/?user=&from=&to=` | GET | Audit trail, filterable |

All endpoints except `register`/`login` require `Authorization: Bearer <access_token>`.

## Connecting the React Frontend

In the frontend's `.env`:
```
VITE_API_BASE_URL=http://127.0.0.1:8000/api
```
The frontend's `axiosInstance.js` already attaches the JWT token from `localStorage` to
every request — after login, store `response.data.access` there and swap the mock-data
calls in each `<feature>API.js` for the real ones.

## Automatic Behaviors
- Creating or completing a task auto-writes an `ActivityLog` entry (`tasks/signals.py`).
- Completing a task auto-fires a `Notification` (`notifications/signals.py`).
- First visit to `/kanban/board/` auto-creates a board with Todo / In Progress / Completed columns.
- Moving a Kanban card updates the underlying Task's `status` to match the column.
