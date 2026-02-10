# Serene Scheduler

Serene Scheduler is a role-based timetable web app:
- Frontend: React + Vite
- Backend: Flask
- Storage (prototype): JSON files

This guide explains **Render deployment using `render.yaml` (Blueprint)**.

## Project Layout

```text
boka/
  render.yaml
  README.md
  backend/
    app.py
    requirements.txt
    users.json
    pending_registrations.json
    published_timetable.json
    reschedule_requests.json
    activity_log.json
  frontend/
    package.json
    vite.config.js
    src/
```

## Important Notes Before Deploy

- `render.yaml` is already configured for:
  - Backend Web Service (`backend`)
  - Frontend Static Site (`frontend`)
  - Persistent disk mounted at `/var/data`
- Frontend publish folder is `build` (correct for your Vite config).
- Do **not** commit real secrets in `backend/.env`.

## 1) Push Code to GitHub

1. Commit latest code including:
   - `render.yaml`
   - `README.md`
2. Push to your GitHub repo.

## 2) Deploy with Blueprint (YAML)

1. Open Render Dashboard: `https://dashboard.render.com`
2. Click `New +`
3. Click `Blueprint`
4. Select your GitHub repo
5. Choose branch (usually `main`)
6. Click `Apply`

Render will read `render.yaml` and auto-create:
- `serene-scheduler-backend` (Web Service)
- `serene-scheduler-frontend` (Static Site)

## 3) What `render.yaml` Creates

### Backend service
- Root: `backend`
- Build: `pip install -r requirements.txt`
- Start: `gunicorn app:app`
- Disk: `scheduler-data` at `/var/data`
- Defaults:
  - `DATA_DIR=/var/data`
  - `SMTP_HOST=smtp.gmail.com`
  - `SMTP_PORT=587`
  - `SHOW_DEV_VERIFICATION_CODE=0`
  - `FLASK_SECRET_KEY` auto-generated

### Frontend service
- Root: `frontend`
- Build: `npm install && npm run build`
- Publish: `build`
- SPA rewrite: `/* -> /index.html`

## 4) Required Environment Variables After Blueprint

In `render.yaml`, variables marked `sync: false` must be entered manually in Render UI.

### Backend (`serene-scheduler-backend`) required
- `FRONTEND_ORIGIN`
- `SMTP_USER`
- `SMTP_PASS`
- `SMTP_FROM_EMAIL`

Optional (already set):
- `SMTP_FROM_NAME=Serene Scheduler`

### Frontend (`serene-scheduler-frontend`) required
- `VITE_API_BASE_URL`

## 5) Correct Order to Set Env Vars

Use this order to avoid CORS/fetch issues:

1. Let backend deploy once and copy backend URL.
   - Example: `https://serene-scheduler-backend.onrender.com`
2. Open frontend service -> Environment:
   - Set `VITE_API_BASE_URL=https://serene-scheduler-backend.onrender.com`
3. Deploy frontend and copy frontend URL.
   - Example: `https://serene-scheduler-frontend.onrender.com`
4. Open backend service -> Environment:
   - Set `FRONTEND_ORIGIN=https://serene-scheduler-frontend.onrender.com`
   - Set SMTP vars (`SMTP_USER`, `SMTP_PASS`, `SMTP_FROM_EMAIL`)
5. Redeploy backend.

## 6) SMTP Setup (Gmail OTP)

Use Gmail App Password (not your normal password).

Steps:
1. Google Account -> Security
2. Enable 2-Step Verification
3. Open `App passwords`
4. Create app password
5. Put values in Render backend env:

```env
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_16_char_app_password
SMTP_FROM_EMAIL=your_email@gmail.com
SMTP_FROM_NAME=Serene Scheduler
```

## 7) Persistent Storage Behavior

Because `DATA_DIR=/var/data` and disk is mounted:
- `users.json`
- `pending_registrations.json`
- `published_timetable.json`
- `reschedule_requests.json`
- `activity_log.json`

will persist across restart/redeploy.

Without disk, JSON data resets.

## 8) Post-Deploy Verification Checklist

1. Open frontend URL.
2. Login as admin.
3. Register one student -> verify OTP.
4. Register one teacher -> approve from admin.
5. Generate/publish timetable.
6. Login as student/teacher and verify timetable.
7. Restart backend service from Render.
8. Confirm users and timetable still exist.

## 9) Common Errors and Fixes

### `Failed to fetch`
Cause:
- `VITE_API_BASE_URL` wrong, or backend not running.
Fix:
- Correct frontend env var, redeploy frontend.

### CORS blocked
Cause:
- `FRONTEND_ORIGIN` not exact frontend URL.
Fix:
- Set exact URL and redeploy backend.

### OTP mail not received
Cause:
- Wrong SMTP values / bad app password.
Fix:
- Recreate app password and update env vars.

### Data disappears
Cause:
- No disk or wrong `DATA_DIR`.
Fix:
- Ensure disk exists and `DATA_DIR=/var/data`.

## 10) Update Deployment After Future Code Changes

After pushing new commits:
- Render auto-deploys by default.
- If env changes are needed, update in Render UI.
- If you edit `render.yaml`, re-apply via Blueprint (or reconnect and sync).

## 11) Security Checklist

- Never store real secrets in repo `.env`.
- Rotate leaked credentials immediately.
- Keep `SHOW_DEV_VERIFICATION_CODE=0` on production.
- Use strong, unique `FLASK_SECRET_KEY`.

## 12) Local Run (Quick)

Backend:

```powershell
cd backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

Frontend:

```powershell
cd frontend
npm install
npm run dev
```

---

If you want, I can also add a short **"Render Deployment Quick Card"** at the top (10 lines only) for faster reuse.
