# CLAUDE.md — Streamline AI Website

## Project Overview

**Repo:** amrinlally12-oss/amrinlally12-oss.github.io
**Live URL:** https://streamlinedai.net
**Hosting:** GitHub Pages (custom domain)
**Stack:** Vanilla HTML / CSS / JS — no frameworks, no build step
**Backend:** Supabase (PostgreSQL + Auth + Row Level Security)
**Owner:** Amrin Ially — Streamline AI Consulting

This is the public-facing website and client portal for Streamline AI Consulting. Every page is a standalone HTML file with inline CSS and JS. Supabase handles authentication, user profiles, and all client data.

---

## Supabase Configuration

| Key | Value |
|---|---|
| Project ref | `swxbuphqvqtfuijuilah` |
| Region | us-west-2 |
| URL | `https://swxbuphqvqtfuijuilah.supabase.co` |
| Anon key | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InN3eGJ1cGhxdnF0ZnVpanVpbGFoIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NzM5ODcxMDQsImV4cCI6MjA4OTU2MzEwNH0.nDPxNStgq_hCw2n2K9d_i7UEIdc5W5rnZRNpXZB8-3M` |
| CDN | `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js` |

Supabase client is initialized on every authenticated page as:
```js
const _sb = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
```

---

## Pages

| File | URL | Description | Auth Required |
|---|---|---|---|
| `index.html` | `/` | Landing page + login modal | No |
| `dashboard.html` | `/dashboard.html` | Client portal — tools, jobs, flagged transactions | Yes (Supabase session) |
| `admin.html` | `/admin.html` | Admin console — client management, tool toggles, analytics | Yes (admin role) |
| `blog.html` | `/blog.html` | Blog listing page | No |
| `get-started.html` | `/get-started.html` | Intake / contact form | No |
| `faq.html` | `/faq.html` | FAQ page | No |
| `privacy.html` | `/privacy` | Privacy policy | No |
| `terms.html` | `/terms` | Terms of service | No |
| `reconciler.html` | Not deployed to GH Pages yet | Transaction Reconciler tool UI | Yes |
| `favicon.svg` | `/favicon.svg` | Site favicon | N/A |

**Note:** `admin.html` has `<meta name="robots" content="noindex,nofollow"/>` to prevent indexing.

---

## Database Schema

RLS is enabled on all tables. Six tables total:

### users
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | Supabase auth.users FK |
| email | text | |
| name | text | |
| role | text | `client` or `admin` |
| client_id | uuid (FK) | References clients.id |
| created_at | timestamptz | |

### clients
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| name | text | Business name |
| owner_email | text | |
| plan | text | `retainer`, `starter`, `growth`, `full_stack` |
| created_at | timestamptz | |

### client_tools
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| client_id | uuid (FK) | References clients.id |
| tool_type | text | See tool types below |
| enabled | boolean | Toggle per client |
| config | jsonb | Tool-specific settings |
| created_at | timestamptz | |

### jobs
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| client_id | uuid (FK) | References clients.id |
| tool_type | text | |
| status | text | `running`, `completed`, `failed` |
| token_usage | integer | Claude API tokens consumed |
| created_at | timestamptz | |

### transactions
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| client_id | uuid (FK) | References clients.id |
| vendor | text | |
| amount | numeric | |
| date | date | |
| flagged | boolean | |
| flag_reason | text | |
| created_at | timestamptz | |

### integrations
| Column | Type | Notes |
|---|---|---|
| id | uuid (PK) | |
| client_id | uuid (FK) | References clients.id |
| provider | text | `gmail`, `quickbooks`, `plaid`, `google_sheets` |
| credentials | jsonb | OAuth tokens (encrypted at rest) |
| created_at | timestamptz | |

### Tool Types
- `flip_analyzer`
- `reconciler`
- `invoice_tracker`
- `trip_matcher`
- `expense_categorizer`

---

## Row Level Security (RLS)

All tables have RLS enabled. Key policies:

| Policy | Table | Rule |
|---|---|---|
| `jobs_select_own` | jobs | Users can only SELECT rows where `client_id` matches their profile's `client_id` |
| `transactions_select_own` | transactions | Same — scoped to the user's `client_id` |
| Admin policies | All tables | Users with `role = 'admin'` have full access via the `is_admin()` database function |

The `is_admin()` function checks the requesting user's role in the `users` table against their `auth.uid()`.

---

## Seeded Data

| Entity | Value |
|---|---|
| Client | Guardian Medical Services |
| Admin user | amrin@guardianms.org (role: `admin`) |

---

## Authentication Flow

### index.html (Landing Page)
The login modal on index.html currently uses **sessionStorage only** — it does NOT call Supabase auth. It stores `sl_user` JSON (`{name, email, biz}`) and redirects to `/dashboard.html`. This is a known issue; it needs to be wired to Supabase `signInWithPassword`.

```
Login modal → doLogin() → sessionStorage.setItem('sl_user', ...) → redirect to /dashboard.html
```

Hash navigation: `/#login` triggers `openLogin()` via hashchange listener.

### dashboard.html (Client Portal)
Uses real Supabase auth. On load:
1. `_sb.auth.getSession()` — if no session, redirects to `/`
2. Queries `users` table for the profile (name, role, client_id)
3. `Promise.all` fetches `client_tools`, `jobs`, `transactions` for that `client_id`
4. Renders tool cards, activity feed, flagged transactions, stat counters
5. Logout: `_sb.auth.signOut()` then redirect to `/`

### admin.html (Admin Console)
Uses real Supabase auth. On load:
1. `_sb.auth.getSession()` — if no session, redirects to `/`
2. Queries `users` for profile, checks `role === 'admin'`
3. If not admin, redirects to `/dashboard.html`
4. `Promise.all` fetches `clients`, `users`, `client_tools`, `jobs`
5. Renders client cards with tool toggles, user list with password reset buttons, job analytics
6. Can toggle `client_tools.enabled` via UPDATE
7. Can trigger `_sb.auth.resetPasswordForEmail(email)`
8. Logout: `_sb.auth.signOut()`

---

## Dashboard Data Flow

```
getSession()
  → query users WHERE id = auth.uid()
  → get profile (name, role, client_id)
  → Promise.all([
      client_tools WHERE client_id = profile.client_id,
      jobs WHERE client_id = profile.client_id (limit 20, desc),
      transactions WHERE client_id = profile.client_id AND flagged = true (limit 5)
    ])
  → renderTools(tools)
  → renderActivity(jobs)
  → renderFlagged(transactions)
  → update stat cards (total tools, jobs this month, flagged count)
```

---

## Security Patterns

- **XSS prevention:** `esc()` function on every page that renders user data — creates a text node via `document.createElement('div')` and reads `.innerHTML`
- **URL validation:** Protocol checks before rendering links
- **Number coercion:** `Number()` used for all numeric display values
- **data-email attributes:** Used for password reset buttons — passed through `esc()` before insertion
- **Admin gate:** Server-side RLS + client-side role check before rendering admin UI
- **Auth redirect:** Unauthenticated users are immediately bounced to `/` on protected pages

---

## Design System

### Theme
Liquid glass — dark background (#04070f), frosted glass panels, animated color orbs, noise texture overlay.

### CSS Variables
```css
:root {
  --bg: #04070f;
  --gold: #C9973A;
  --gold-l: #E8C97A;
  --gold-d: #8a6522;
  --white: #ffffff;
  --dim: rgba(255,255,255,0.55);
  --faint: rgba(255,255,255,0.22);
  --glass-bg: rgba(255,255,255,0.04);
  --glass-border: rgba(255,255,255,0.10);
  --glass-highlight: rgba(255,255,255,0.08);
  --green: #8fda3a;
  --green-bg: rgba(99,153,34,0.15);
  --green-b: rgba(99,153,34,0.3);
  --red: #f07a5a;
  --red-bg: rgba(226,75,74,0.12);
  --red-b: rgba(226,75,74,0.3);
  --amber: #f0b840;
  --amber-bg: rgba(240,184,64,0.12);
  --amber-b: rgba(240,184,64,0.3);
  --blue: #5ab4f0;
  --blue-bg: rgba(90,180,240,0.12);
  --r: 16px;
}
```

### Typography
- **Headings:** Syne (weights 700, 800)
- **Body:** Figtree (weights 300, 400, 500, 600)
- dashboard.html also loads Inter (used in some elements)

```html
<link href="https://fonts.googleapis.com/css2?family=Syne:wght@400;500;600;700;800&family=Figtree:wght@300;400;500;600&display=swap" rel="stylesheet"/>
```

### Glass Card
```css
.glass {
  background: rgba(255,255,255,0.04);
  backdrop-filter: blur(24px) saturate(180%);
  -webkit-backdrop-filter: blur(24px) saturate(180%);
  border: 1px solid rgba(255,255,255,0.10);
  border-radius: 14px;
}
```

### Animated Background Orbs
Every page includes:
```html
<div class="bg-canvas">
  <div class="orb orb1"></div>
  <div class="orb orb2"></div>
  <div class="orb orb3"></div>
  <div class="orb orb4"></div>
</div>
<div class="noise"></div>
```

### Logo SVG
```html
<svg width="32" height="32" viewBox="0 0 32 32" fill="none">
  <rect width="32" height="32" rx="7" fill="rgba(201,151,58,0.15)"/>
  <path d="M4 11 C7.5 7,12 17,16 11 C20 5,24.5 15,28 11" stroke="#C9973A" stroke-width="2.2" stroke-linecap="round" fill="none"/>
  <path d="M4 17 C7.5 13,12 23,16 17 C20 11,24.5 21,28 17" stroke="#E8C97A" stroke-width="1.8" stroke-linecap="round" fill="none"/>
  <path d="M4 23 C7.5 19,12 29,16 23 C20 17,24.5 27,28 23" stroke="rgba(255,255,255,0.25)" stroke-width="1.4" stroke-linecap="round" fill="none"/>
</svg>
```

---

## Code Preferences

- Always use CSS variables from `:root` — never hardcode hex colors
- Mobile responsive by default — test at 768px and 480px breakpoints
- No inline styles unless unavoidable
- Single-file HTML preferred (CSS and JS inline per page)
- Real copy only — no lorem ipsum or placeholder text
- Use `esc()` for any user-supplied data rendered into the DOM
- Skip preamble — just do the work
- Flag issues proactively

---

## Known Issues

1. **Auth bridge broken:** index.html login modal writes to `sessionStorage` but dashboard.html checks Supabase `getSession()` — these are disconnected. index.html needs to call `_sb.auth.signInWithPassword()` instead.
2. **`/#login` hash:** The hashchange listener exists but the login modal may not trigger reliably on all navigation patterns.
3. **Blog posts are placeholders:** Blog cards link to pages that don't exist yet (404s).
4. **Get-started form has no backend:** The intake form doesn't submit anywhere — needs FormSubmit, Formspree, or a Supabase edge function.
5. **Tool launch buttons:** Some dashboard tool cards show `alert()` / `comingSoon()` toasts instead of linking to real tools.
6. **Empty tables:** `jobs` and `transactions` tables have no data until clients actually use tools.
7. **Session auto-refresh:** No `onAuthStateChange` listener to handle token refresh or session expiry gracefully.
8. **dashboard.html anon key mismatch:** dashboard.html uses an older anon key (`iat: 1742589972`) while admin.html uses the current one (`iat: 1773987104`). Both should use the same key.

---

## Deployment

To deploy changes to streamlinedai.net:
1. Edit files locally
2. Drag into the `amrinlally12-oss.github.io` repo on GitHub (or push via git)
3. Click Commit
4. Live in approximately 1 minute via GitHub Pages

The backend API (flip-analyzer-server) is a separate repo deployed on Railway — do not mix backend files into this repo.
