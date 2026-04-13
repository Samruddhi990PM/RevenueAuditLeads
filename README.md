# Strategy Cues — Holiday Home Revenue Audit
### Lead Generation Web App · Vercel · Supabase · Resend

---

## 1. Project Overview

**What it is:** A lead-generation web app built for Strategy Cues, targeting holiday home owners. Visitors complete a 9-question Revenue Health Check, submit their contact details, view a personalised revenue readiness score, then choose to connect via WhatsApp or book a strategy call on Calendly.

**Business goal:** Capture warm leads and immediately notify the Strategy Cues team so they can follow up within minutes.

**What happens when someone submits:**
1. Their details are saved to a Supabase database table (`revenueaudit_leads`)
2. A Supabase Edge Function fires instantly, calling the Resend API
3. A professional HTML email alert lands in 3 team inboxes within seconds — containing name, email, phone, score, and all 9 questionnaire responses

---

## 2. Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | Plain HTML + CSS + Vanilla JS | Single-file app, no framework needed |
| Hosting | Vercel | Static site hosting via Git |
| Database | Supabase (PostgreSQL) | Stores every lead submission |
| Email alerts | Resend API | Sends formatted HTML email to 3 recipients |
| Serverless function | Supabase Edge Function (Deno/TypeScript) | Bridges Supabase insert → Resend email |
| Fonts | Google Fonts — Cormorant Garamond + Inter | Typography |
| Background image | Base64-encoded inside HTML | No external image dependency |

---

## 3. Repository Structure

```
your-repo/
│
├── index.html                          ← The entire frontend app (rename from revenue-health-check.html)
├── README.md                           ← This file
│
└── supabase/
    └── functions/
        └── RevenueAuditLead/
            └── index.ts                ← Edge Function: receives lead → sends email via Resend
```

> **Note on naming:** Vercel serves `index.html` automatically at the root URL. Rename `revenue-health-check.html` to `index.html` before pushing to Git.

---

## 4. How the App Works

### User Journey (4 Sections, Sequential)

```
┌─────────────────────────────────────────────┐
│  SECTION 1 — Revenue Questionnaire           │
│  9 questions · Yes / Partial / No pills      │
│  Score badge updates per row in real time    │
│  Submit button unlocks when all 9 answered   │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│  SECTION 2 — Lead Capture Form               │
│  Full Name · Phone · Email                   │
│  Validated before proceeding                 │
└──────────────────────┬──────────────────────┘
                       ↓ On Submit:
              ① POST → Supabase REST API
                 saves row to revenueaudit_leads
              ② POST → Supabase Edge Function
                 RevenueAuditLead → Resend → email to 3 inboxes
                       ↓
┌─────────────────────────────────────────────┐
│  SECTION 3 — Revenue Readiness Score         │
│  Animated score ring (0–100)                 │
│  Green ≥80 · Amber 50–79 · Red <50          │
│  Personalised message per band               │
└──────────────────────┬──────────────────────┘
                       ↓ "Claim Your FREE Revenue Strategy"
┌─────────────────────────────────────────────┐
│  SECTION 4 — Connect                         │
│  💬 WhatsApp: +971 52 877 1983               │
│  📅 Calendly: Book a Kick Off Call           │
└─────────────────────────────────────────────┘
```

### Scoring Formula

| Answer | Points |
|--------|--------|
| Yes | 100 |
| Partial | 50 |
| No | 0 |

```
Final Score = Sum of all 9 answers ÷ 9
```

**Example:** 5×Yes + 3×Partial + 1×No = 500 + 150 + 0 = 650 ÷ 9 = **72.2 / 100**

### Score Bands

| Range | Label | Colour |
|-------|-------|--------|
| 80–100 | Excellent Momentum 🟢 | Green `#1A9E52` |
| 50–79 | Good Potential 🟡 | Amber `#D4820A` |
| 0–49 | Strong Improvement Opportunity 🔴 | Red `#C43434` |

---

## 5. Local Development

No build tools, no package manager, no install step needed.

```bash
# Clone the repository
git clone https://github.com/YOUR_ORG/YOUR_REPO.git -->https://github.com/Samruddhi990PM/RevenueAuditLeads
cd YOUR_REPO

# Open the app directly in your browser
# Mac
open index.html

# Windows
start index.html
```

Before local testing works end-to-end, you need to fill in the 3 credentials in `index.html` — see [Section 8](#8-environment-variables).

> **Note:** If you open via `file://` and Supabase CORS rejects the request, use a simple local server instead:
> ```bash
> # Python (built into Mac/Linux)
> python3 -m http.server 3000
> # Then open: http://localhost:3000
> ```

---

## 6. Supabase Setup

### 6.1 — Create a Supabase Project

1. Go to accountonboarding -> SCGenericApps -> revenueaudit_leads
2. Account -> ishanstrategycues

### 6.2 — Create the Database Table

Go to **SQL Editor** in the left sidebar and run:

```sql
CREATE TABLE IF NOT EXISTS revenueaudit_leads (
  id           bigserial     PRIMARY KEY,
  full_name    text          NOT NULL,
  phone        text          NOT NULL,
  email        text          NOT NULL,
  final_score  numeric(5,1)  NOT NULL,
  answers      jsonb         NOT NULL DEFAULT '[]',
  submitted_at timestamptz   NOT NULL DEFAULT now()
);
```

### 6.3 — Enable Row Level Security (RLS)

```sql
ALTER TABLE revenueaudit_leads ENABLE ROW LEVEL SECURITY;
```

### 6.4 — Add the Anonymous INSERT Policy

> ⚠️ This is the most commonly missed step. Without it, form submissions return a 403 error.

```sql
CREATE POLICY "anon_can_insert"
  ON revenueaudit_leads
  FOR INSERT
  TO anon
  WITH CHECK (true);
```

### 6.5 — Add the Authenticated SELECT Policy (for team to read leads)

```sql
CREATE POLICY "auth_can_select"
  ON revenueaudit_leads
  FOR SELECT
  TO authenticated
  USING (true);
```

### 6.6 — Verify the Table Works

Run this test insert in the SQL Editor:

```sql
INSERT INTO revenueaudit_leads (full_name, phone, email, final_score, answers)
VALUES (
  'Test Lead',
  '+97234567',
  'tejabe@sts.com',
  72.2,
  '[{"question_no":1,"question":"Test question","response":"yes","score":100}]'::jsonb
);

-- Should return the inserted row
SELECT * FROM revenueaudit_leads ORDER BY id DESC LIMIT 1;
```

If you see the row — the table is ready. ✅

### 6.7 — Get Your Supabase Credentials

Go to **Project Settings → API**:

| Credential | Where to find it | Looks like |
|-----------|-----------------|-----------|
| **Project URL** | "Project URL" field | `https://abcxyz123.supabase.co` |
| **Anon / Public Key** | "Project API Keys" → `anon public` row | `eyJhbGciOiJIUzI1NiIs...` (long string) |

---

## 7. Resend + Edge Function Setup

### 7.1 — What's Already Done

| Item | Status |
|------|--------|
| Resend account | ✅ Exists |
| Domain `mail.strategycues.com` | ✅ Verified (GoDaddy, Tokyo region) |
| Sender address | ✅ `revenueauditleads@mail.strategycues.com` |
| Resend API key | ✅ Created — reuse from existing project |

The same API key works across all your Supabase projects. No new Resend setup needed.

### 7.2 — Add the Resend Secret to Supabase

1. **Supabase Dashboard → Project Settings** (gear icon, bottom left) **→ Edge Functions**
2. Scroll to **Secrets Management**
3. Click **Add new secret**

| Field | Value |
|-------|-------|
| Name | `RESEND_API_KEY` |
| Value | Your `re_...` key from Resend Dashboard → API Keys |

4. Click **Save**

### 7.3 — Create the Edge Function

1. **Supabase Dashboard → Edge Functions** (left sidebar)
2. Click **Create a new function**
3. Name it exactly: **`RevenueAuditLead`**
4. Click **Create function** — the inline code editor opens
5. **Select all existing code → Delete it**
6. Open `supabase/functions/RevenueAuditLead/index.ts` from your repo
7. **Copy the entire file contents → Paste into the editor**
8. Click **Deploy** — wait ~30 seconds for the green confirmation

### 7.4 — Get the Edge Function URL

After deploying, the URL is shown at the top of the Edge Functions page:

```
https://YOUR_PROJECT_ID.supabase.co/functions/v1/RevenueAuditLead
```

Copy this — you'll need it in the next section.

### 7.5 — Email Alert Content

Every form submission triggers an email to all 3 recipients containing:

| Field | Description |
|-------|-------------|
| **Subject** | `🏡 New Lead: [Name] — Score [X]/100 ([Band])` |
| **Header** | "New Revenue Audit Lead" in WhatsApp green + submission timestamp (UAE time) |
| **Score Banner** | Large score number with colour-coded band label |
| **Lead Details** | Name, email (clickable), phone (clickable), score badge |
| **Answers Table** | All 9 questions with colour-coded Yes/Partial/No pills and point scores |
| **Action Buttons** | WhatsApp Lead + Book a Call (Calendly) |

### 7.6 — Email Recipients

```typescript
const RECIPIENTS = [
  "tejalbe@segycues.com",
  "ian@stratees.com",
  "samr.waghure@stracues.com",
];
```

To add/remove recipients, edit this array in `index.ts` and redeploy the Edge Function.

---

## 8. Environment Variables

The app uses **no server-side environment variables** — it's a static HTML file. The 3 credentials are set directly inside `index.html`.

Open `index.html` in any text editor. Scroll to the bottom `<script>` block. Find these 3 lines and replace the placeholder values:

```js
const SUPABASE_URL  = 'https://YOUR_PROJECT_ID.supabase.co';
const SUPABASE_ANON = 'YOUR_ANON_PUBLIC_KEY';
const EDGE_FN_URL   = 'https://YOUR_PROJECT_ID.supabase.co/functions/v1/RevenueAuditLead';
```

| Constant | Where to find the value |
|----------|------------------------|
| `SUPABASE_URL` | Supabase → Project Settings → API → **Project URL** |
| `SUPABASE_ANON` | Supabase → Project Settings → API → **anon / public** key |
| `EDGE_FN_URL` | Supabase → Edge Functions → `RevenueAuditLead` → **URL shown at the top of the page** |

> `YOUR_PROJECT_ID` is the same 16-character string in both `SUPABASE_URL` and `EDGE_FN_URL`. It's also shown in Project Settings → General → **Reference ID**.

After updating, **commit and push** — Vercel auto-deploys from the `main` branch.

---

## 9. Deployements on GIT & Vercel


Vercel: Account - samruddhiwstrategycues -> Strategycues projects hobby -> revenue-audit-lead

GIT: Samruddhi990PM -> RevenueAuditLeads

Site Link: https://revenue-audit-leads.vercel.app/

---


## 13. Project Checklist

Use this to track setup progress from scratch.

### Supabase Database
- [ ] Supabase project created
- [ ] `revenueaudit_leads` table created (Section 6.2)
- [ ] Row Level Security enabled (Section 6.3)
- [ ] `anon_can_insert` policy created (Section 6.4)
- [ ] `auth_can_select` policy created (Section 6.5)
- [ ] Test SQL insert ran successfully — row visible in Table Editor (Section 6.6)
- [ ] Project URL and Anon key copied (Section 6.7)

### Resend + Edge Function
- [ ] `RESEND_API_KEY` secret added to Supabase project (Section 7.2)
- [ ] `RevenueAuditLead` Edge Function created in Supabase Dashboard (Section 7.3)
- [ ] `index.ts` code pasted into editor and deployed successfully (Section 7.3)
- [ ] Edge Function URL copied (Section 7.4)

### HTML App Configuration
- [ ] `SUPABASE_URL` replaced with real value in `index.html` (Section 8)
- [ ] `SUPABASE_ANON` replaced with real value in `index.html` (Section 8)
- [ ] `EDGE_FN_URL` replaced with real value in `index.html` (Section 8)
- [ ] File renamed to `index.html` (Section 3)
- [ ] Changes committed and pushed to Git

### Vercel Deployment
- [ ] Vercel project created and linked to Git repo (Section 9.1)
- [ ] Framework preset set to "Other" (Section 9.1)
- [ ] First deployment successful — live URL working (Section 9.1)
- [ ] Custom domain added and verified (Section 9.2) *(optional)*

### End-to-End Testing
- [ ] Open live Vercel URL — app loads correctly
- [ ] Complete all 9 questions — score updates per row
- [ ] Submit questionnaire — Section 2 form appears
- [ ] Submit lead form — row appears in Supabase Table Editor within 5 seconds
- [ ] All 3 email inboxes receive the alert email within 10 seconds
- [ ] Email shows correct name, phone, email, score, and all 9 answers
- [ ] Score ring animates correctly in Section 3
- [ ] "Claim Your FREE Revenue Strategy" button reveals Section 4
- [ ] WhatsApp button opens correct number
- [ ] Calendly button opens booking page ✅

---

*Last updated: April 2026 · Strategy Cues Revenue Intelligence Platform*
