# PredictCorp — User Acceptance Testing Plan

**App URL:** `https://{username}.github.io/{repo}/` (or `http://localhost:8000` for local)
**Firebase project:** `corp-pred`
**Tester setup:** Use two different browser sessions (e.g. Chrome normal + Chrome Incognito) to simulate two different users.

---

## Before You Start

| Step | Action |
|---|---|
| 1 | Open the app URL in Browser A |
| 2 | Open the same URL in Browser B (Incognito) |
| 3 | In Firebase Console → Realtime Database, confirm the `/markets` node exists (auto-seeded on first load) |
| 4 | Keep Firebase Console open in a third tab to verify database writes as you test |

---

## T-01 · Login and Session Persistence

**Goal:** Verify name-based login, session restore, and anonymous login.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | On Browser A landing page, enter name `Alice` and click Join | Navigates to Markets view. Toast: "Welcome Alice! 1,000 PredictCoins credited." Credit balance shows 1,000. |
| 2 | On Browser B, enter name `Bob` and click Join | Same welcome flow for Bob. |
| 3 | Refresh Browser A | App reloads directly to Markets view as Alice. Toast: "Welcome back Alice! Session restored." |
| 4 | Open Firebase Console → `/user_trades` | Two nodes exist, one for Alice, one for Bob, each with `credits: 1000` and `name` fields. |
| 5 | On Browser A, open DevTools → Application → Local Storage. Find key `predictcorp_session`. | Contains `{ uid: "u_...", name: "Alice" }` |
| 6 | Delete the `predictcorp_session` key in Local Storage and refresh. | App shows landing page again (session cleared). |

### Pass Criteria
- [ ] New users are created in Firebase on first login
- [ ] Returning users restore their session without re-entering name
- [ ] Clearing localStorage logs user out

---

## T-02 · Quarterly Token Reset (Phase 5)

**Goal:** Verify users receive 1,000 tokens at the start of a new quarter.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | Log in as a returning user (Alice) | Normal session restore, no extra token toast (same quarter). |
| 2 | In Firebase Console, navigate to `/user_trades/{alice_uid}/lastTokenQuarter` and change the value to `2025-Q4` (a past quarter). | Value saved in Firebase. |
| 3 | Refresh Browser A (Alice) | Toast appears: "New quarter! 1,000 PredictCoins added." Credit balance increases by 1,000. |
| 4 | Check Firebase `/user_trades/{alice_uid}/lastTokenQuarter` | Updated to current quarter (e.g. `2026-Q1`). |
| 5 | Refresh again without changing the Firebase value. | No additional token toast. Balance unchanged. |

### Pass Criteria
- [ ] Tokens only awarded once per quarter per user
- [ ] `lastTokenQuarter` is updated after award
- [ ] No double-award on subsequent logins in the same quarter

---

## T-03 · Market Templates (Phase 1)

**Goal:** Verify Binary / Scalar / Milestone type selector works in market creation.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | As Alice, click **New** in the nav. | Create Market modal opens. |
| 2 | Observe the "Forecast Type" section at the top of the modal. | Three buttons visible: Binary (blue), Scalar (green), Milestone (purple). Binary is selected by default. |
| 3 | Click **Scalar**. | Scalar button highlights green. Hint text below updates to: `e.g. "Will Q4 revenue exceed $500M?"` |
| 4 | Click **Milestone**. | Milestone button highlights purple. Hint: `e.g. "Will regulatory approval be obtained before Q3?"` |
| 5 | Select **Binary**, fill in all required fields (see T-04 for governance fields), and create the market. | Market card appears in the Markets view with a blue "Binary" badge. |
| 6 | Create another market with type **Scalar**. | Market card shows green "Scalar" badge. |
| 7 | Open either market detail. | Type badge appears next to the category badge in the header. |
| 8 | Check Firebase `/markets/{new_id}`. | `type` field is present with value `"binary"`, `"scalar"`, or `"milestone"`. |

### Pass Criteria
- [ ] All three types selectable; correct hint text displayed
- [ ] Selected type is stored in Firebase
- [ ] Badge appears on market card and detail view

---

## T-04 · Restricted Participants & Governance Fields (Phase 2)

**Goal:** Verify required governance fields and that creators/verifiers cannot trade.

### Setup
Create a test market as Alice with:
- **Forecast Type:** Binary
- **Question:** "Will the test pass?"
- **Department:** Engineering
- **Resolution Source:** Manual Review
- **Resolution Verifier:** Bob ← this is Browser B's user

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | Open the Create Market modal. Leave Department blank. | Create Market button is greyed out / disabled. |
| 2 | Fill in Question only. Leave Resolution Source and Verifier blank. | Button still disabled. |
| 3 | Fill in all three governance fields and the question. | Create Market button becomes active. |
| 4 | Create the market. | Market appears in the list. |
| 5 | Open the new market detail as Alice (the creator). | Right panel shows grey "Restricted Participant" box: "You created this market. Trading is not permitted." Trade buttons (YES/NO) are NOT visible. |
| 6 | Scroll down in the detail panel. | "Creator Controls → Claim Resolution" button is visible (amber). |
| 7 | Switch to Browser B (Bob) and open the same market. | Right panel shows grey "Restricted Participant" box: "You are the resolution verifier." Trade buttons are NOT visible. Verifier controls are NOT yet shown (no pending resolution yet). |
| 8 | Log in as a third user (Charlie) in a new session. Open the same market. | Trade buttons ARE visible. No restriction notice. |
| 9 | Check the market detail → governance row below the stats. | Shows "Department: Engineering", "Resolution Source: Manual Review", "Verifier: Bob". |

### Pass Criteria
- [ ] Create button disabled until all 3 governance fields + question are filled
- [ ] Creator cannot see trade buttons in their own market
- [ ] Named verifier cannot see trade buttons in the market they verify
- [ ] Other users can trade normally
- [ ] Governance fields visible on market detail

---

## T-05 · Independent First Estimate (Phase 3)

**Goal:** Verify the estimate modal appears before the market price is shown, only on first visit.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | As Charlie (a user who has never opened this market), click on any active market card. | A full-screen overlay modal appears BEFORE the market detail is visible. Modal title: "Your Independent Estimate". |
| 2 | Observe the modal. | Shows the market question, a 0–100% slider defaulting to 50, and a "Submit & View Market" button. The market detail page is obscured behind the modal. |
| 3 | Move the slider to 70%. | The probability bar below the slider updates in real time to show 70%. |
| 4 | Click "Submit & View Market". | Modal closes. Full market detail is now visible, including the current market price. |
| 5 | Scroll down in the market detail. | A note shows: "Your initial estimate: 70% YES". |
| 6 | Navigate away and click the same market again. | No estimate modal. Market detail loads directly. |
| 7 | Check Firebase `/user_trades/{charlie_uid}/firstEstimates/{marketId}`. | `{ estimate: 70, submittedAt: <timestamp> }` |
| 8 | As Alice (restricted creator) open any market. | No estimate modal appears. |
| 9 | Open an already-resolved market as any user. | No estimate modal appears. |

### Pass Criteria
- [ ] Modal appears only on first visit to an unresolved market
- [ ] Slider updates preview bar in real time
- [ ] Estimate stored in Firebase after submission
- [ ] No modal for creators, verifiers, or on resolved markets
- [ ] Estimate shown as a note in the market detail on subsequent visits

---

## T-06 · Trading and LMSR Pricing

**Goal:** Verify trades move probabilities correctly and deduct credits.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | As Charlie, open an active market and note the current YES%. | Initial price visible. |
| 2 | Click **YES**, set shares to 20, confirm the cost preview. | Cost shown in credits. New Prob shown higher than current. |
| 3 | Click Buy YES. | Toast: "Bought 20 YES for X cr". Credit balance decreases by X. |
| 4 | Check the probability bar on the market card. | YES% has increased. |
| 5 | Switch to Browser B (Bob is restricted). Try to trade. | Trade buttons not visible (Bob is verifier for the test market). On a different market Bob can trade. |
| 6 | Attempt to buy more shares than available credits. | Buy button disabled. Error text: "Need X, have Y". |
| 7 | Check Firebase `/markets/{id}/q`. | `q[0]` has increased (YES shares bought). |

### Pass Criteria
- [ ] Trade moves probability in the correct direction
- [ ] Credits deducted correctly
- [ ] Insufficient-credit guard works
- [ ] Firebase `q` vector updated after each trade

---

## T-07 · Two-Verifier Resolution + 48h Dispute (Phase 6)

**Goal:** Verify the staged resolution flow: claim → (optional dispute) → verify → payout.

### Setup
Use the market created in T-04 ("Will the test pass?") where Alice = creator, Bob = verifier. Have Charlie trade YES shares first.

### Steps — Claim

| # | Action | Expected Result |
|---|---|---|
| 1 | As Alice, open the test market detail. | "Claim Resolution" amber button visible. |
| 2 | Click Claim Resolution. Modal opens. | Modal shows "Claim Resolution", market title, verifier name (Bob), and YES/NO buttons. |
| 3 | Click **YES**. | Modal closes. Toast: "Resolution claimed. Awaiting verifier confirmation." |
| 4 | Market card in the Markets list. | Badge changes to "Awaiting Verification" (amber). |
| 5 | Open the market detail. | Amber banner: "Resolution claimed: YES by Alice." Dispute link visible (for Charlie). Trade buttons replaced by "Trading paused" notice. |

### Steps — Dispute (optional path)

| # | Action | Expected Result |
|---|---|---|
| 6 | As Charlie (who traded in this market), open the market detail. | "Dispute this resolution (48h window)" link visible in the amber banner. |
| 7 | Click Dispute. Enter reason "Incorrect — outcome not verified yet." Submit. | Toast: "Dispute filed. Governance panel notified." |
| 8 | Market card badge. | Changes to "Under Review". |
| 9 | Verifier panel for Bob. | Confirm button is NOT shown (dispute blocks verification). |

> To continue to the verify step, clear the dispute in Firebase: delete `/markets/{id}/dispute`.

### Steps — Verify & Payout

| # | Action | Expected Result |
|---|---|---|
| 10 | As Bob, open the test market detail. | Right panel shows "Verifier Controls" (indigo box). "Creator claimed: YES". "Confirm & Resolve" button. |
| 11 | Click Confirm & Resolve. | Toast: "Resolved YES! Payouts sent." |
| 12 | Market card badge. | Changes to green "YES" resolved badge. Opacity reduced. |
| 13 | Check Charlie's credit balance. | Increased by the number of YES shares Charlie held. |
| 14 | Check Firebase `/markets/{id}/resolved`. | `"yes"` |
| 15 | Check Firebase `/markets/{id}/pendingResolution`. | `null` |

### Pass Criteria
- [ ] Creator can claim but not confirm
- [ ] Verifier sees confirm panel only after claim
- [ ] Dispute blocks verification; market shows "Under Review"
- [ ] Payouts distributed only after verifier confirmation
- [ ] Resolved market is read-only (no trading)

---

## T-08 · Reputation & Super Predictor (Phase 4)

**Goal:** Verify Brier scores are computed and Super Predictor badge awarded.

### Setup
Requires at least one resolved market where the user submitted a first estimate AND held a position.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | After a market resolves (T-07), go to Charlie's Portfolio view. | If Charlie had a first estimate and position, a reputation card appears below the summary tiles. Shows "Accuracy score: X/100 · 1 market resolved". |
| 2 | Check Firebase `/user_trades/{charlie_uid}/reputation`. | `{ brierSum: <value>, marketCount: 1 }` |
| 3 | Go to Leaderboard view. | Charlie's row has an "Accuracy" column showing their score (e.g. "85/100"). |
| 4 | For Super Predictor: resolve 4 more markets where the user had perfect estimates (set estimate to 100% on YES markets, then resolve YES). | After 5 markets with near-zero Brier score, ⭐ appears next to the user's name on the leaderboard. Portfolio card says "⭐ Super Predictor". |
| 5 | Users with no resolved estimates show "—" in the Accuracy column. | Confirmed. |

### Pass Criteria
- [ ] Brier score computed and stored in Firebase on resolution
- [ ] Reputation card appears on Portfolio once at least one market is scored
- [ ] Accuracy column in Leaderboard populated for real users, "—" for mock entries
- [ ] Super Predictor badge awarded at avg Brier < 0.10 over 5+ markets

---

## T-09 · Dashboard — Early-Warning (Phase 7)

**Goal:** Verify the leadership table shows correct data with trend arrows.

### Steps

| # | Action | Expected Result |
|---|---|---|
| 1 | Click the **Dashboard** tab in the nav. | New view loads. Title "Early-Warning Dashboard". Table with columns: Market, Type, YES Probability, Trend, Participation, Status. |
| 2 | Review the rows for all active markets. | Each row shows the market title, type badge, probability bar + %, status badge. |
| 3 | Find a market with YES probability below 40%. | Row is highlighted amber. AlertCircle icon next to the probability. Status badge: "Low Confidence". |
| 4 | Make several trades on one market to shift its probability. Wait a moment. Revisit Dashboard. | Trend column starts populating with ↑ Rising or ↓ Falling after the first hourly snapshot is written. |
| 5 | Check Firebase `/snapshots/{marketId}`. | Keys in `YYYYMMDDTHH` format with probability values. |
| 6 | Click a row in the Dashboard table. | Navigates to that market's detail view. |
| 7 | Observe the amber info box at the bottom of the Dashboard. | Text: "Markets below 40% YES probability may warrant a project review…" |

### Pass Criteria
- [ ] Dashboard tab appears in nav and is clickable
- [ ] All active markets shown (no resolved markets)
- [ ] Sub-40% rows highlighted amber
- [ ] Clicking a row navigates to the market detail
- [ ] Snapshots written to Firebase on each trade

---

## T-10 · Anonymity Safeguards (Phase 8)

**Goal:** Verify no exact participant counts are exposed anywhere.

### Checklist — check each location

| Location | What to look for | Pass if |
|---|---|---|
| Market card (bottom left) | Participation text | Shows range like "Several participants", never "31 trades" |
| Market detail stats row — "Trades" cell | Participation label | Shows a range string |
| Dashboard table — Participation column | Per-market count | Range string |
| Nav bar — live user count | Users online | Shows range string (e.g. "A few participants") |
| Resolve modal | Trade count line | Shows range string |

### Pass Criteria
- [ ] No integer trade/participant count visible anywhere in the UI
- [ ] All counts replaced with bucketed range labels
- [ ] Ranges: "No participants yet" / "A few" / "Several" / "~15–30" / "~30–60" / "60+"

---

## T-11 · End-to-End User Journey

**Goal:** Full happy-path walkthrough as a new employee.

| # | Step | Expected |
|---|---|---|
| 1 | New user Dan joins with his name. | 1,000 credits, redirected to Markets. |
| 2 | Dan clicks a market he has an opinion on. | First Estimate modal appears. He enters 65%. Clicks submit. |
| 3 | Dan sees the current market price (e.g. 72%). He notes the market is higher than his estimate. | Market detail fully visible. His estimate (65%) shown at bottom. |
| 4 | Dan buys 10 NO shares. | Trade executes. Credits decrease. Probability shifts slightly. |
| 5 | Dan visits Portfolio. | Shows his 1 position with current P&L. No reputation card yet (no resolved markets). |
| 6 | Dan visits Leaderboard. | His name appears with his credits. Accuracy shows "—". |
| 7 | Dan visits Dashboard. | Sees the market he just traded on. Participation shows "A few participants". |
| 8 | The market creator (separate session) claims resolution as YES. | Market shows "Awaiting Verification". Dan's market detail shows dispute option. |
| 9 | The verifier (separate session) confirms YES. | Payouts sent. Dan's NO shares pay nothing. |
| 10 | Dan's Portfolio shows updated P&L. Reputation card appears. | Accuracy score visible. |

### Pass Criteria
- [ ] All steps complete without errors
- [ ] Firebase state correct at each step
- [ ] No browser console errors (check DevTools → Console)

---

## Known Limitations (Out of Scope for UAT)

- Wallet connection (MetaMask/WalletConnect/Coinbase) is a UI mock — does not connect to a real wallet
- No server-side validation — all logic runs client-side
- Firebase security rules are open in test mode; production deployment needs rule hardening
- Trend arrows in Dashboard require at least 2 hourly snapshots (i.e. trades in different hours)
- Super Predictor badge requires 5 resolved markets scored — hard to test without many resolutions
