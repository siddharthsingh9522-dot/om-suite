# OM Suite

This repo has **two independent Flask apps**, each deployed as its own
Render web service:

```
om-suite/
├── main_app/                      → unified login + 4 tools (see below)
├── gr_auto_modification_system/   → untouched, its own login/DB
└── render.yaml                    → Render Blueprint for both services
```

## 1. `main_app/` — unified app

One login system (with admin approval) in front of four tools:

| Tool | URL | What it does |
|---|---|---|
| TBB Party Mail | `/tbb` | Branch-wise GR verification mail |
| Bill Generated Mail | `/bill` | Branch-wise Auto Generated Freight Bill mail |
| Branch Detail Finder | `/branch` | Branch code lookup (single + bulk Excel) |
| Customer Search | `/customer` | Customer search (manual + bulk Excel) |
| OM Automation | `/om` | CN / GST / Party Code / Master lookup |

**Login & admin (already built-in, unchanged behaviour):**
- Anyone can sign up at `/signup`.
- The **first person to sign up automatically becomes admin**.
- Everyone after that starts as **`pending`** — they cannot log in until
  an admin approves them at `/admin/users`.
- Once approved (`active`), a user can log in and use **all four tools**
  above — Branch, Customer, OM Automation, and TBB/Bill Mail all sit
  behind the same login + approval gate.
- Admin can approve, disable, re-enable, or promote/demote any user from
  `/admin/users` — full control stays with the admin, exactly as asked.

This *is* the "admin must approve before the app can be used" flow —
it now covers every tool in the suite, not just the mail app it
originally shipped with.

### Running locally

```bash
cd main_app
pip install -r requirements.txt
cp .env.example .env      # fill in what you have; safe to leave blanks
python app.py
```

Open `http://localhost:5000`, sign up (you'll become admin), and go.

### Important notes before deploying

- **User database persistence:** `main_app` stores its users/login data
  in a local SQLite file (`data/app.db`). Render's **free** plan has an
  *ephemeral* filesystem — every redeploy (and periodically on free
  instances) wipes this file, which means your users/admin approvals
  would reset. For anything beyond quick testing, either:
  - add a Render **persistent disk** mounted at `main_app/data` (paid
    plan), or
  - migrate `db.py` to a hosted Postgres (e.g. Neon, same as
    `gr_auto_modification_system` already does) — ask if you'd like
    this done.
- **GST Query (inside OM Automation) needs a real Chrome browser** —
  it drives services.gst.gov.in with Selenium because that portal
  requires a human-solved captcha. A plain Render Python web service
  does **not** have Chrome installed, so this one screen will show
  "not available" until you either:
  - switch this service to a **Docker** deploy on Render with Chrome +
    chromedriver installed in the image, or
  - leave GST Query for local/desktop use only and use CN / Party /
    Master (which don't need a browser) on Render.
  Every other tool in the suite works normally on a plain Render web
  service — this only affects the GST-captcha screen.
- Mail sending (TBB/Bill modules) needs your real `SMTP_*` / `IMAP_*`
  values — add them in the Render dashboard's Environment tab after
  first deploy (they're intentionally left blank in `render.yaml`).

## 2. `gr_auto_modification_system/` — untouched

No changes were made here, as asked — it already had its own complete
login system. It already ships its own `render.yaml`; the top-level
`render.yaml` in this repo just points Render at it as a second
service. It needs a Postgres `DATABASE_URL` (e.g. from Neon) — see its
own `README.md` for details.

## Deploying both to Render

1. Push this whole `om-suite/` folder to a GitHub repo.
2. In Render: **New → Blueprint**, point it at the repo. Render reads
   the top-level `render.yaml` and creates **both** services in one go.
3. Render will prompt you for the values marked `sync: false`
   (`DATABASE_URL` for gr_auto, SMTP/IMAP/AI keys for main_app) — fill
   in what you have, or leave blank and add later from each service's
   Environment tab.
4. Each service gets its own URL — link between them from wherever you
   normally share these tools (e.g. an internal wiki page, or add a
   plain link in `main_app`'s dashboard, already included).

If you'd rather deploy them one at a time instead of as a Blueprint,
each folder also works as a standalone Render "Web Service" — just set
its **Root Directory** to `main_app` or `gr_auto_modification_system`
when creating it.
