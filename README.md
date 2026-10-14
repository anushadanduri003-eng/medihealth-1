# MediMate AI — Personal Health Copilot

MediMate keeps the existing React interface and branding, with Netlify hosting the static frontend and a separate API-only Flask service backed by MySQL. Supabase Auth provides email/password accounts and persistent browser sessions; Flask verifies Supabase JWTs and continues to store MediMate profiles and health records in MySQL. The Flask server is not run by Netlify. The React app sends `/api/*` requests through a Netlify Function proxy to the separately hosted backend; local Vite development proxies the same API to Flask. The incomplete legacy Flask-rendered screens are not registered; use the React frontend.

## Features

- Supabase Auth accounts with MySQL-backed profiles, documents, observations, timeline entries, audit events, chat sessions and messages.
- Authenticated, user-scoped access to private uploaded files and health records.
- Backend OCR, structured result extraction, editable/verified observations, English/Telugu summaries, and reference-range highlighting.
- Report comparison, timeline intelligence, source-linked assistant responses, Telugu term explanations, and a doctor visit brief.
- Chat history and evidence citations persist in MySQL. AI endpoints return explicit errors if the backend Gemini key is absent or the provider fails; no canned clinical responses are used.

## Local development

Requirements: Node.js, Python 3.10+, MySQL, and the Tesseract executable for image OCR.

1. Create the MySQL database and apply `medimate_ai/schema.sql`. For an existing database, apply `medimate_ai/migrations/20261014_add_telugu_summary_fields.sql` and `medimate_ai/migrations/20261014_add_supabase_user_id.sql` once before deploying the updated backend. These migrations add the Telugu summary fields and Supabase identity mapping without deleting user or health records.
2. Copy `.env.example` to the repository root as `.env`. Set `VITE_SUPABASE_URL` to the project origin (for example, `https://your-project.supabase.co`, without `/rest/v1/`) and `VITE_SUPABASE_PUBLISHABLE_KEY` to the Supabase publishable key. Set `SUPABASE_URL` for Flask token verification, and `SUPABASE_PUBLISHABLE_KEY` so Flask can verify email ownership when linking an existing MySQL account. Also set a random `SECRET_KEY`, a real MySQL `DATABASE_URL`, and the backend-only `GEMINI_API_KEY`. Keep `.env` untracked.
3. Install and start the Flask backend from the repository root:

   ```powershell
   python -m pip install -r medimate_ai\requirements.txt
   python -m medimate_ai.run
   ```

4. In a second terminal, install and start the React/Vite frontend:

   ```powershell
   npm install
   npm run dev
   ```

   Open the Vite URL printed in the terminal. Vite forwards `/api/*` requests to `http://127.0.0.1:5000` by default. Set `FLASK_BACKEND_URL` before `npm run dev` if Flask uses another local URL.

5. Run checks:

   ```powershell
   npm run lint
   npm run build
   python -m pytest medimate_ai\tests -v
   ```

Image OCR requires Tesseract installed on the Flask host. PDFs must contain selectable text; scanned PDFs without a text layer are rejected rather than represented as successfully extracted.

## Deployment

### Flask + MySQL backend

1. Provision a managed MySQL database with TLS enabled. Create a dedicated application database/user and apply `medimate_ai/schema.sql`; if upgrading an existing installation, apply both migration SQL files above first.
2. Create a Python web service on Render (or another persistent Python host) using the repository root:
   - Build command: `pip install -r medimate_ai/requirements.txt`
   - Start command: `gunicorn medimate_ai.run:app --bind 0.0.0.0:$PORT`
3. Configure backend environment variables on that host only:
   - `FLASK_ENV=production`
   - `SECRET_KEY` — a randomly generated value of at least 32 characters
   - `DATABASE_URL` — a MySQL SQLAlchemy URL starting with `mysql+pymysql://`, with reserved password characters URL-encoded
   - `SUPABASE_URL` — the Supabase project origin, without `/rest/v1/`
   - `SUPABASE_PUBLISHABLE_KEY` — the Supabase publishable key, used only to confirm email ownership when linking a pre-existing MySQL account
   - `GEMINI_API_KEY` — server-side only
   - `UPLOAD_FOLDER=/var/data/medimate/uploads`
4. Attach a persistent disk mounted at `/var/data/medimate` and ensure the directory is writable by the service. Use encrypted storage and backups appropriate for health records. Do not rely on a host's ephemeral filesystem for uploaded documents.
5. Install the Tesseract OCR system package on the backend host for image uploads. Verify `/api/health` on the backend before connecting Netlify.

### Netlify frontend

1. Import this existing repository into Netlify and keep the configured build command `npm run build` and publish directory `dist`.
2. Deploy the Flask + MySQL backend first and confirm its `/api/health` endpoint responds successfully.
3. In Netlify, open **Site configuration → Environment variables** and add `BACKEND_URL` with the Flask service origin only (for example, `https://your-flask-service.onrender.com`; do not add `/api` or `/api/health`), plus `VITE_SUPABASE_URL` (the Supabase project origin without `/rest/v1/`) and `VITE_SUPABASE_PUBLISHABLE_KEY`. Vite variables are embedded at build time, so redeploy after setting or changing them. Never add a Supabase secret or `service_role` key.
4. Deploy the latest frontend code. The Netlify Function at `/api/*` forwards API requests, including session cookies, to Flask; Netlify serves the frontend and function but does not run Flask. If `BACKEND_URL` is missing, the API returns a clear configuration error instead of silently trying an unrelated database.
5. Verify the deployed site by registering a test account, uploading a real report, confirming it appears after sign-out/sign-in, checking that another account cannot access its document URL, and testing chat citations, comparison, timeline, and original-file download.

In Supabase **Authentication → URL Configuration**, set the Site URL to your deployed Netlify origin and add that origin plus `http://localhost:5173` (or the local Vite URL) to the allowed redirect URLs. Keep email/password enabled under **Authentication → Providers → Email**. If email confirmations are enabled, new users see a verification notice and cannot sign in until Supabase issues an authenticated session. Flask verifies access tokens against Supabase's public JWKS endpoint; use an asymmetric JWT signing key (such as ES256) in Supabase. Existing MySQL accounts are linked only after Flask verifies the Supabase user's email confirmation; their MySQL records are retained. The existing MySQL schema remains the user-profile and health-data store; no Supabase tables were created, so no new RLS policy is required.

Do not put `DATABASE_URL`, `SECRET_KEY`, or `GEMINI_API_KEY` in `VITE_*` variables or client code. The publishable key is intended for browser use; never use a Supabase secret or `service_role` key. No deploy, push, or production migration was performed as part of this work.

## Important limitations

- Live ABHA/ABDM identity and OTP verification is not integrated; the UI reports this instead of simulating a successful link.
- FHIR-format export is generated from the signed-in user's stored data; it is not an ABDM connection or certification.
- OCR and AI output must be checked against the original report and reviewed by a clinician. MediMate is not a diagnostic or emergency-care service.
- Storage encryption, backups, and retention depend on the configured hosting providers and remain deployment responsibilities.
