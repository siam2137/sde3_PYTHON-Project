# CampusFlow — Python + Flask edition

CampusFlow is a responsive university-student web application for organizing courses, finding study groups, tracking deadlines, sharing course questions/resources, and managing study checklists.

## Technology

- **Backend:** Python 3.10+, Flask
- **Database layer:** Flask-SQLAlchemy / SQLAlchemy ORM
- **Database:** MySQL 8+ (PyMySQL driver) or SQLite for a quick local demo
- **Authentication:** Flask server-side sessions, Werkzeug password hashing
- **Frontend:** HTML5, custom CSS, Bootstrap 5, Bootstrap Icons, vanilla JavaScript
- **Security:** CSRF protection for all write requests, Jinja autoescaping, ORM parameterization, resource ownership checks, validated external URLs

## Project structure

```text
campusflow_python/
├── app/
│   ├── __init__.py             # App factory, extensions, CSRF and error handlers
│   ├── auth.py                 # Register, login, logout
│   ├── board.py                # Course posts, resources and replies
│   ├── config.py               # Environment-based configuration
│   ├── deadlines.py            # Deadline CRUD and monthly calendar
│   ├── extensions.py           # SQLAlchemy and Flask-Session
│   ├── main.py                 # Dashboard and student profile
│   ├── models.py               # Normalized SQLAlchemy models
│   ├── security.py             # Login guard and CSRF helpers
│   ├── seed.py                 # Demo content for local setup
│   ├── sessions.py             # Study sessions and join approvals
│   ├── tasks.py                # Course checklist CRUD
│   ├── timeutils.py            # Asia/Dhaka local time helper
│   ├── static/
│   │   ├── css/app.css
│   │   └── js/app.js
│   └── templates/
│       ├── auth/
│       ├── board/
│       ├── deadlines/
│       ├── errors/
│       ├── profile/
│       ├── sessions/
│       ├── tasks/
│       ├── base.html
│       └── dashboard.html
├── instance/                   # Local SQLite DB and server-side session files
├── database.sql                # MySQL schema and demo data
├── .env.example
├── .gitignore
├── requirements.txt
└── run.py
```

## 1. Requirements

- Python **3.10 or newer**
- For MySQL mode: MySQL 8.0.16+ (or XAMPP's MySQL/MariaDB-compatible server)
- A modern browser

## 2. Install and run (quick SQLite setup)

SQLite is the easiest way to start: no database server is required.

### Windows

Open Command Prompt or PowerShell inside the extracted project folder:

```powershell
py -m venv .venv
.venv\Scripts\activate
py -m pip install -r requirements.txt
copy .env.example .env
```

Open `.env` and **comment out the MySQL `DATABASE_URL` line** so the application uses its built-in SQLite default. You can also set it explicitly:

```dotenv
DATABASE_URL=sqlite:///instance/campusflow.db
SECRET_KEY=replace-with-a-long-random-secret
APP_TIMEZONE=Asia/Dhaka
COOKIE_SECURE=false
```

Create the database tables and insert the sample content:

```powershell
py -m flask --app run.py init-db
py -m flask --app run.py seed-demo
```

Start the server:

```powershell
py run.py
```

Open **http://127.0.0.1:5000** in your browser.

### macOS / Linux

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Set `DATABASE_URL=sqlite:///instance/campusflow.db` in `.env`, then run:

```bash
flask --app run.py init-db
flask --app run.py seed-demo
python run.py
```

## 3. Run with MySQL / XAMPP

1. Start **MySQL** from the XAMPP Control Panel.
2. Open phpMyAdmin at `http://localhost/phpmyadmin`.
3. Import the project's `database.sql` file. It creates the `campusflow` database, all nine tables, indexes, foreign keys, constraints, and demo data. **The script drops and recreates the CampusFlow tables**, so use a fresh development database.
4. Copy `.env.example` to `.env` and set the database connection. The example below assumes the default XAMPP `root` account has no password:

   ```dotenv
   SECRET_KEY=replace-with-a-long-random-secret
   DATABASE_URL=mysql+pymysql://root:@127.0.0.1:3306/campusflow?charset=utf8mb4
   APP_TIMEZONE=Asia/Dhaka
   COOKIE_SECURE=false
   ```

   For a MySQL account with a password, put that password after `root:` and URL-encode special characters.

5. Install packages and run the app:

   ```bash
   python -m venv .venv
   # Activate the virtual environment, then:
   pip install -r requirements.txt
   python run.py
   ```

6. Open **http://127.0.0.1:5000**.

If you prefer SQLAlchemy to create an empty MySQL schema instead of importing the SQL file, create an empty `campusflow` database first, set `DATABASE_URL`, then run `flask --app run.py init-db` and `flask --app run.py seed-demo`.

**Time zone:** CampusFlow interprets date/time inputs in `APP_TIMEZONE` (default `Asia/Dhaka`). Set your MySQL server's time zone to the same local zone when using the SQL seed file so the sample deadline times line up with the application clock.

## 4. Demo accounts

The seeded accounts all use this password:

**Password:** `Campus123!`

| Email | Name |
|---|---|
| `siam@campusflow.test` | Siam Ahmed |
| `nadia@campusflow.test` | Nadia Rahman |
| `arif@campusflow.test` | Arif Hasan |

Use Nadia or Arif to inspect sessions they created and the creator-only request controls. Use Siam to test a join request and the personal dashboard.

## 5. Features and route map

| Feature | URL | What it does |
|---|---|---|
| Dashboard | `/` or `/dashboard` | Active courses, 48-hour reminders, upcoming deadlines, study groups and task summary |
| Register / login | `/register`, `/login` | Password-hashed student accounts and session-based authentication |
| Profile | `/profile` | Edit department/semester and select active courses |
| Study groups | `/sessions/` | Search by title/topic/location/course code, create sessions and request to join |
| Deadlines | `/deadlines/` | Add, complete and delete events; monthly calendar with type colors |
| Course board | `/board/` | Questions, validated resource links, course filters and replies |
| Checklist | `/tasks/` | Add, complete and delete per-course tasks |

All create/update/delete operations use POST forms protected by CSRF tokens. Jinja autoescapes text rendered in templates. ORM queries use bound parameters. Each personal action checks the authenticated user's ownership. Session creators alone can accept or decline requests.

### Group capacity handling

`max_members` includes the creator. For example, a limit of four means the creator plus up to three accepted students. Approvals lock the study-session row inside a database transaction (`SELECT ... FOR UPDATE`) before counting accepted requests and updating the request/session state. MySQL enforces this row lock; SQLite is suitable for local demos but does not provide the same row-level concurrency behavior.

### Contextual permissions instead of global user roles

All registered accounts are students. A user acts as a **session creator** only for sessions they created, and as a **post author** only for posts they wrote. Those permissions are checked against the relevant record's `creator_id` or `user_id`; users do not receive a permanent creator/author role across the whole application.

## 6. Database tables

The schema contains nine normalized tables:

- `users`
- `courses`
- `user_courses` (composite primary key)
- `study_sessions`
- `session_requests` (unique `(session_id, requester_id)`)
- `deadlines`
- `course_posts`
- `course_replies`
- `course_tasks`

Foreign keys, uniqueness constraints, status/type checks, and indexes are defined in `database.sql` and mirrored by the SQLAlchemy models.

## 7. Security and deployment notes

- Change `SECRET_KEY` to a long random value before using the project beyond local development.
- For HTTPS deployment, set `COOKIE_SECURE=true` and configure HTTPS at the reverse proxy/web server.
- Do not commit `.env`, database files, or session files.
- Use a dedicated MySQL application account rather than the XAMPP root account outside local development.
- The app validates user-submitted resource links to HTTP/HTTPS URLs and opens them with `rel="noopener noreferrer"`.
- This is a learning-project starter, not a production-hardened campus identity system. Add email verification, password reset, rate limiting, automated tests, logging, backups and deployment-specific hardening before a public launch.

## 8. Troubleshooting

**`ModuleNotFoundError` after starting:** Activate the virtual environment and run `pip install -r requirements.txt` again.

**Can't connect to MySQL:** Confirm MySQL is running, the database exists, the username/password are correct, and `DATABASE_URL` in `.env` matches your local setup.

**MySQL authentication/driver error:** Confirm `PyMySQL` is installed and the URL starts with `mysql+pymysql://`.

**CSRF / expired form message:** Refresh the page and submit again. The app requires a valid session token on every write request.

**No courses in the forms:** Select courses from your profile or use a seeded demo account.
