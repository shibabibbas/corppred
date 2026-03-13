# PredictCorp — Corporate Prediction Markets

A single-file web app that lets employees forecast corporate outcomes using an internal token market. Built for a university project (FNCE313 — Financial Innovation, Blockchains & DeFi, SMU 2026).

---

## What It Does

Employees join with their name, receive 1,000 PredictCoins each quarter, and trade YES/NO shares on corporate forecast questions (e.g. "Will APAC revenue exceed $50M?"). An LMSR automated market maker prices shares in real time. Probabilities aggregate across all trades into a live forecast signal for leadership.

### Key Features

| Feature | Description |
|---|---|
| LMSR AMM | Logarithmic market scoring rule (b=100) prices every trade |
| Market Templates | Binary / Scalar / Milestone question formats |
| Restricted Participants | Creator and named verifier cannot trade in their own market |
| First Estimate | User submits an independent probability before seeing the market price |
| Reputation & Super Predictor | Brier score computed per user across resolved markets; badge at avg < 0.10 over 5+ markets |
| Quarterly Token Reset | 1,000 PredictCoins issued each quarter; tracked per user in Firebase |
| Two-Verifier Resolution | Creator claims outcome → named verifier confirms → payouts execute |
| 48h Dispute Window | Any participant can dispute a pending resolution within 48 hours |
| Early-Warning Dashboard | Leadership table: probability, trend (24h snapshot diff), participation range, status |
| Anonymity Safeguards | Participation counts shown as ranges (e.g. "Several participants"), never exact |
| Real-time Sync | Firebase Realtime Database; all users see the same live state |

---

## Architecture

```
index.html          ← entire app (single file, no build step)
  ├── CDN React 18  ← UI framework (Babel transpiles JSX in browser)
  ├── CDN Tailwind  ← styling
  ├── CDN Recharts  ← bar/pie charts in Analytics view
  └── Firebase SDK  ← Realtime Database (auth is name-based, not Firebase Auth)
```

**No build tool. No bundler. No server.** The file opens directly in a browser or is served as a GitHub Pages static site.

---

## Firebase Setup

### 1. Create a Firebase project

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Create a new project (or use existing `corp-pred`)
3. Enable **Realtime Database** (not Firestore)
   - Region: Asia Southeast 1 (or your preference)
   - Start in **test mode** for development

### 2. Get your config

In Firebase Console → Project Settings → Your Apps → Web app config:

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "your-project.firebaseapp.com",
  databaseURL: "https://your-project-default-rtdb.asia-southeast1.firebasedatabase.app",
  projectId: "your-project",
  storageBucket: "your-project.firebasestorage.app",
  messagingSenderId: "...",
  appId: "..."
};
```

Paste this into `index.html` lines 27–34.

### 3. Database rules (recommended minimum)

In Firebase Console → Realtime Database → Rules:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

> For production, tighten these so users can only write to their own `user_trades/{uid}` node.

### 4. Firebase database schema

```
/markets
  /{marketId}
    title             string
    category          string  — "M&A" | "Product" | "Sales" | "Strategy" | "Operations" | "Custom"
    type              string  — "binary" | "scalar" | "milestone"
    q                 [number, number]   — LMSR quantity vector [qYes, qNo]
    deadline          string  — "YYYY-MM-DD"
    creator           string  — user name
    department        string  — responsible department
    resolutionSource  string  — e.g. "ERP System"
    resolutionVerifier string — user name of verifier
    description       string
    resolved          null | "yes" | "no"
    pendingResolution null | { outcome, claimedBy, claimedAt }
    dispute           null | { filedBy, reason, filedAt }
    trades            number  — total trade count
    volume            number  — total credit volume

/user_trades
  /{uid}
    name              string
    credits           number
    lastTokenQuarter  string  — e.g. "2026-Q1"
    positions
      /{marketId_outcome}
        key           string
        marketId      string
        outcome       0 | 1   — 0=YES, 1=NO
        shares        number
        avgCost       number
        title         string  — cached market title
    history
      /{pushId}
        marketId      string
        title         string
        outcome       "YES" | "NO"
        shares        number
        cost          string
        time          string
        prob          string
    firstEstimates
      /{marketId}
        estimate      number  — 0–100 (percent YES)
        submittedAt   number  — Unix ms
    reputation
        brierSum      number  — cumulative Brier score
        marketCount   number  — number of resolved markets scored

/snapshots
  /{marketId}
    /{hourKey}        number  — YES probability at that hour (key format: YYYYMMDDTHH)

/presence
  /{uid}              true    — removed on disconnect
```

---

## GitHub Pages Setup

1. Push `index.html` to a GitHub repository
2. Go to repo **Settings → Pages**
3. Source: `Deploy from a branch` → branch `main` → folder `/ (root)`
4. Save — GitHub will publish the site at `https://{username}.github.io/{repo}/`

No build step needed. The single `index.html` is the entire deployment artifact.

> **Firebase and GitHub Pages work together without a backend.** The browser connects directly to Firebase Realtime Database from the client.

---

## Session & Auth Model

- Users log in with a name (no Firebase Auth)
- `uid` is generated as `u_` + 9 random chars on first login
- Session stored in `localStorage` under key `predictcorp_session` as `{ uid, name }`
- On return visit, session is restored from localStorage and validated against Firebase
- Anonymous login is supported (`Anon_xxxx`) but does not earn reputation scores

---

## Market Resolution Flow

```
Creator → "Claim Resolution" (YES or NO)
  └─ Firebase: markets/{id}/pendingResolution = { outcome, claimedBy, claimedAt }
  └─ Market badge: "Awaiting Verification"
  └─ Trading paused

[48h window] Any participant who traded → "Dispute"
  └─ Firebase: markets/{id}/dispute = { filedBy, reason, filedAt }
  └─ Market badge: "Under Review — Governance Panel"
  └─ Resolution blocked

Resolution Verifier (named at market creation) → "Confirm & Resolve"
  └─ Firebase: markets/{id}/resolved = "yes" | "no"
  └─ Firebase: markets/{id}/pendingResolution = null
  └─ Payouts: 1 credit per winning share per user
  └─ Brier scores computed for all users with firstEstimates for this market
```

---

## Reputation (Brier Score)

When a market resolves, for every user who both:
- submitted a first estimate for that market, AND
- held a position in that market

The platform computes:

```
brierContribution = (estimate/100 − outcome)²
```

where `outcome` is `0` for NO and `1` for YES.

Running totals stored at `/user_trades/{uid}/reputation`:
- `brierSum` — cumulative score (lower = better)
- `marketCount` — how many markets scored

**Super Predictor** badge awarded when: `brierSum / marketCount < 0.10` AND `marketCount >= 5`

---

## LMSR Pricing

```js
// Liquidity parameter b = 100
cost(q)       = b * log(exp(q[0]/b) + exp(q[1]/b))
price(q, o)   = exp(q[o]/b) / sum(exp(q[i]/b))
costOfTrade   = cost(newQ) − cost(oldQ)
```

- Initial `q = [0, 0]` → 50% YES / 50% NO
- Buying YES shares increases `q[0]`, raising YES probability
- Payouts: 1 credit per winning share on resolution

---

## Local Development

```bash
# Any static file server works
python -m http.server 8000
# then open http://localhost:8000
```

Or just open `index.html` directly in a browser. Firebase connects from `file://` too, though some browsers restrict this — use a local server to be safe.

---

## Seed Data

On first load, if `/markets` is empty in Firebase, 8 seed markets are written automatically (defined in the `SEEDS` constant in `index.html`). They cover M&A, Product, Sales, Strategy, and Operations categories with realistic descriptions and initial LMSR quantities.

To reset: delete the `/markets` node in the Firebase console and reload the page.
