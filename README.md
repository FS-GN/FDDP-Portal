# FDDP Application Portal

**Fisheries Development and Diversification Program**  
Government of Nunavut — Department of Community Services, Fisheries & Sealing Division

---

## What This Is

A secure, web-based application portal for the FDDP. It allows community organizations, HTOs, and individual fishers to submit funding applications online and track their status using a private access code. Program staff manage and update applications through a password-protected admin dashboard.

The portal runs on [GitHub Pages](https://pages.github.com/) (free, no server required) and connects to [Supabase](https://supabase.com/) for secure data storage and authentication.

---

## Features

| Feature | Description |
|---|---|
| **Application Form** | Full FDDP intake form with validation, all 24 Nunavut communities, Inuit Beneficiary status, mailing address, budget line items, and optional quote attachment |
| **Status Checker** | Applicants enter their access code to see a visual 7-stage pipeline and any program notes — no personal data is ever exposed publicly |
| **Admin Dashboard** | Password-protected panel for GN staff to view all applications, filter by stage, and update status and notes |
| **Confirmation Email** | Automated email sent to the applicant on submission with their access code, file number, and next steps |
| **Demo Mode** | Runs fully in-browser with sample data when Supabase credentials are not yet configured |

---

## Files

```
fddp/
├── index.html              ← The portal (deploy this to GitHub Pages)
├── supabase_setup.sql      ← Run once in Supabase SQL Editor to set up the database
├── send-confirmation.ts    ← Supabase Edge Function for automated confirmation emails
└── README.md               ← This file
```

---

## How It Works

```
Applicant's browser
    │
    ├──► GitHub Pages ──► serves index.html (the portal UI)
    │
    └──► Supabase ──────► stores application data securely
              │
              └──► Edge Function ──► Resend ──► confirmation email to applicant
```

GitHub never touches applicant data. All form submissions go directly from the browser to Supabase over HTTPS.

---

## Security Model

- **Row Level Security (RLS)** is enabled on the database. Anonymous users can only INSERT new applications — they cannot read any data directly.
- **Status lookups** are handled by a `SECURITY DEFINER` function that returns only safe fields (stage, notes, title, community). Email, phone, address, dollar amounts, and contact details are never returned to the public.
- **Admin login** uses Supabase's built-in email/password authentication. Credentials are verified server-side and return a short-lived JWT. The hardcoded password from the prototype has been removed entirely.
- **The anon key** in `index.html` is safe to leave in the code. It is a public key by design — Row Level Security enforces what it can and cannot do.

---

## Setup Guide

### Step 1 — Supabase (database)

1. Create a free account at [supabase.com](https://supabase.com)
2. Click **New Project** → name it `fddp-portal` → choose a strong database password → select region **US East** → Create
3. Wait ~60 seconds for the project to provision
4. Go to **SQL Editor** → **New Query**
5. Paste the full contents of `supabase_setup.sql` and click **Run**
6. Run the additional columns needed for new fields:

```sql
ALTER TABLE applications
  ADD COLUMN is_inuit_beneficiary TEXT,
  ADD COLUMN address               TEXT,
  ADD COLUMN budget_items          JSONB,
  ADD COLUMN has_quote             BOOLEAN DEFAULT FALSE,
  ADD COLUMN quote_file_path       TEXT DEFAULT '';
```

7. Go to **Storage** → **New Bucket** → name it `quotes` → enable **Public bucket** → Save
8. Go to **Settings → API** → copy your **Project URL** and **anon public key** (you will need these in Step 4)

---

### Step 2 — Resend (email)

1. Create a free account at [resend.com](https://resend.com)
2. Go to **API Keys** → **Create API Key** → copy the key (starts with `re_`)
3. For the pilot, emails will send from `onboarding@resend.dev` — no domain setup required
4. When ready to send from an official address, go to **Domains** → add and verify your domain

---

### Step 3 — Edge Function (automated emails)

The Edge Function runs on Supabase's servers and fires automatically every time a new application is submitted.

**Option A — Supabase Dashboard (no CLI needed)**

1. In your Supabase project, go to **Edge Functions** → **New Function**
2. Name it `send-confirmation`
3. Paste the contents of `send-confirmation.ts` into the editor
4. Go to **Settings → Secrets** → add a secret named `RESEND_API_KEY` with your Resend key as the value
5. Click **Deploy**

**Option B — Supabase CLI**

```bash
# Install CLI
npm install -g supabase

# Login and link your project
supabase login
supabase link --project-ref YOUR_PROJECT_REF

# Create the function folder and paste send-confirmation.ts inside it
mkdir -p supabase/functions/send-confirmation
# (copy send-confirmation.ts to supabase/functions/send-confirmation/index.ts)

# Set your Resend API key as a secret
supabase secrets set RESEND_API_KEY=re_xxxxxxxxxxxx

# Deploy
supabase functions deploy send-confirmation --no-verify-jwt
```

**After deploying**, update the trigger in `supabase_setup.sql`:

- Replace `YOUR_PROJECT_REF` with your project ref (found in Settings → General)
- Replace `YOUR_SERVICE_ROLE_KEY` with your service role key (found in Settings → API — this is the **secret** key, not the anon key)
- Re-run the `EMAIL TRIGGER` block in the SQL Editor

Also update two lines in `send-confirmation.ts` before deploying:
- `FROM_ADDRESS` → your verified sender email (or leave as `onboarding@resend.dev` for pilot)
- `portalUrl` → your GitHub Pages URL

---

### Step 4 — Portal (GitHub Pages)

1. Open `index.html` in a text editor
2. Find the CONFIG block near the top (marked with a ⚙️ comment) and replace the two placeholder values:

```javascript
const SUPABASE_URL      = "https://yourproject.supabase.co";
const SUPABASE_ANON_KEY = "eyJ...your anon key...";
```

3. In your GitHub repository, create a folder called `fddp`
4. Upload `index.html` into that folder
5. Go to **Settings → Pages** → set source to `main` branch → Save
6. Your portal will be live at: `https://yourusername.github.io/your-repo/fddp/`

---

### Step 5 — Add admin users

1. In Supabase, go to **Authentication → Users → Add User**
2. Enter each team member's GN email address
3. They will receive an email to set their password
4. They can then sign in at the Admin tab of the portal

---

## Application Stages

| Stage | Meaning |
|---|---|
| Received | Application submitted, awaiting initial review |
| Eligibility Review | Checking applicant and project eligibility |
| Technical Assessment | Reviewing project merit, budget, and feasibility |
| Approval Committee | File presented to program committee |
| Decision | Final approval or denial communicated |
| Agreement Drafting | Contribution agreement being prepared |
| Funds Disbursed | Payment issued to applicant |

---

## Updating an Application Status

1. Go to the **Admin** tab → sign in with your GN email and password
2. Click any application in the list to open the detail panel
3. Change the **Stage** dropdown to the new stage
4. Optionally add a **Program Note** — this note will be visible to the applicant when they check their status
5. Click **Save Update**

The applicant will see the updated stage and note immediately when they look up their code.

---

## Moving to a GN Domain (future)

When ready to move from `onboarding@resend.dev` to an official sender address:

1. Add your domain in Resend → **Domains → Add Domain**
2. Add the DNS records Resend provides to your domain registrar
3. Once verified, update one line in `send-confirmation.ts`:

```typescript
from: FROM_ADDRESS,   // change from FROM_ADDRESS_PILOT
```

4. Redeploy the Edge Function

---

## Contact

Fisheries & Sealing Division  
Department of Community Services  
Government of Nunavut  
[fisheries@gov.nu.ca](mailto:fisheries@gov.nu.ca)
