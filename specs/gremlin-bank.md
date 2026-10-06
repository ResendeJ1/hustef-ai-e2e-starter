# Gremlin Bank: sign in, dashboard and domestic transfer

## Application overview

Gremlin Bank is a fictional bank used as practice material. Explored on 2026-10-06 against
`http://localhost:8787`, **Release 1** (the footer shows the release). Releases 2 and 3 rename some
controls (see `tests/walls/support.ts` → `names`) and show a cookie consent dialog on the first page view.

Every new browser context starts with a fresh bank state (cookie `gb_state`). All values below are
for that fresh state of user `GREMLIN_USER`. Credentials and the PIN come from `.env`
(`GREMLIN_USER`, `GREMLIN_PASSWORD`, `GREMLIN_PIN`). They never appear in the tests or in this plan.

**Seed:** `seed.spec.ts` signs in and stops on the dashboard (h1 "Accounts").

### Business rules (the oracle for Lab 3)

Source: **rule** = given business rule, the expected values are derived from it. **seed** = fixed test data of
the fresh state. **observed** = seen while exploring and not yet confirmed as a rule (see Open questions).

| Rule | Value | Source |
|---|---|---|
| Opening balances | Everyday Account 1,250,000 HUF; Savings Account 5,400,000 HUF | seed |
| Fee (domestic) | 0.3 % of the amount, at least 200 HUF, at most 6,000 HUF: `min(6000, max(200, amount × 0.003))` | rule |
| Fee rounding | half up to whole HUF | observed |
| Funds check | amount + fee must be ≤ balance of the source account | rule (equal allowed: observed) |
| Daily limit | 2,000,000 HUF of amount per day, earlier transfers that day included. The fee does not count | rule (fee excluded: observed) |
| Per-transfer limit | 10,000,000 HUF | rule |
| Amount | whole HUF > 0. Spaces and thousands commas are ignored, leading zeros are dropped | rule (> 0); normalisation: observed |
| IBAN | Hungarian IBAN (`HU` + 26 digits) with a valid checksum. Spaces and lower case are allowed. It must be checked with **Check IBAN** before **Continue** | rule |
| Reference | optional, `maxlength` 140 | observed |
| Confirmation | Transaction PIN in `<gb-secure-pin>` (closed shadow root, reachable only with the keyboard: focus **Confirm transfer**, then Shift+Tab), then **Approve payment** in the "Confirm payment" dialog (iframe titled "Gremlin Secure") | observed |

Risk tags: `[high]` money movement, balances, authentication; `[medium]` validation and data display that does not move money; `[low]` cosmetic and informational widgets.

### Fixed test data

| Item | Value |
|---|---|
| Everyday Account IBAN | HU39 9992 0265 3141 5926 5358 9797 |
| Savings Account IBAN | HU03 9992 0265 2718 2818 2845 9043 |
| Saved payee Kiss Péter | HU72 9990 1017 1618 0339 8874 9892 |
| Saved payee Nagy Eszter | HU71 9990 2025 1414 2135 6237 3099 |
| Saved payee Tóth Bence | HU03 9990 3033 1732 0508 0756 8879 |

### Random values (assert the format only)

- Tip of the day: a non-empty sentence that changes on every load.
- EUR/HUF rate: `^\d{3}\.\d{2}$` (seen: 385.40, 386.98, 393.62).
- Transfer reference on the done page: `^Reference: GB-[A-Z0-9]{6}$`.
- Codes like `GRM-XXXX-XXXX` (session code, chart caption, IBAN verified, PIN check passed): `GRM-[A-Z0-9-]+`.

---

## Test scenarios

### 1. Sign in and sign out

**Seed:** `seed.spec.ts` (scenarios 1.1 to 1.3 start signed out, in a fresh context)

#### 1.1 [high] A valid user signs in and lands on the dashboard

**Steps:**
1. Open `/login`.
2. Fill **Username** with `GREMLIN_USER` and **Password** with `GREMLIN_PASSWORD`.
3. Click **Sign in**.

**Expected results:**
- URL is `/dashboard` and the h1 is "Accounts".
- The banner shows "Signed in as `GREMLIN_USER`" and a **Sign out** button.
- The session code matches `^Session code: GRM-[A-Z0-9-]+$`.

#### 1.2 [high] Invalid credentials are rejected with one generic message

**Steps:** For each row, open `/login`, fill the fields and click **Sign in**.

| Username | Password | Expected |
|---|---|---|
| (empty) | (empty) | stays on `/login`, "Wrong username or password." |
| `GREMLIN_USER` | `wrong-password` | stays on `/login`, "Wrong username or password." |
| `nobody` | `GREMLIN_PASSWORD` | stays on `/login`, "Wrong username or password." |

**Expected results:**
- The message is the same for all rows, so it does not reveal whether the user exists.
- No "Signed in as" text in the banner.

#### 1.3 [high] Protected pages redirect to sign in when signed out

**Steps:**
1. In a fresh context, open `/dashboard`, then `/transfer`.

**Expected results:**
- Each one ends on `/login` with the h1 "Sign in to Gremlin Bank".

#### 1.4 [high] Sign out ends the session

**Steps:**
1. Start from the seed (signed in on the dashboard).
2. Click **Sign out**.
3. Press the browser Back button.
4. Open `/transfer` directly.

**Expected results:**
- After step 2 the URL is `/login` and the "Sign in to Gremlin Bank" heading is shown.
- After steps 3 and 4 the user is still on `/login`. The dashboard and the form are not shown.

---

### 2. Dashboard

**Seed:** `seed.spec.ts`

#### 2.1 [high] Both accounts show name, IBAN and opening balance

**Steps:**
1. Wait until "Loading accounts..." disappears.

**Expected results:**
- Region "Everyday Account": IBAN `HU39 9992 0265 3141 5926 5358 9797`, Balance `1,250,000 HUF`.
- Region "Savings Account": IBAN `HU03 9992 0265 2718 2818 2845 9043`, Balance `5,400,000 HUF`.
- Link **New transfer** goes to `/transfer`.

#### 2.2 [medium] Recent transactions list the seeded history, newest first

**Steps:**
1. Wait until "Loading accounts..." disappears.
2. Read the table "Recent transactions".

**Expected results:** the table "Recent transactions" (columns Date, Description, Amount) has exactly these rows, in this order:

| Date | Description | Amount |
|---|---|---|
| 2026-09-30 | Grocery store, Budapest | -18,450 HUF |
| 2026-09-29 | Salary, Gremlin Works Ltd. | +685,000 HUF |
| 2026-09-27 | Mobile phone bill | -7,990 HUF |
| 2026-09-25 | Card payment, bookshop | -12,300 HUF |
| 2026-09-24 | Transfer from Savings Account | +50,000 HUF |

Debits start with `-` and credits with `+`. These rows are seed data. If the seed dates turn out to be relative to
today, compare descriptions and amounts only.

#### 2.3 [low] Spending chart data can be shown and hidden

**Steps:**
1. Check that no "Spending in the last 30 days" table is visible.
2. Click **Show chart data**.
3. Read the table.
4. Click **Hide chart data**.

**Expected results:**
- After step 2 the button is **Hide chart data** (`aria-expanded=true`) and the table appears.
  Its caption matches `^Spending in the last 30 days \(GRM-[A-Z0-9-]+\)$`.
- 30 data rows with consecutive dates, from today minus 29 days to today.
- Every amount matches `^\d{1,3}(,\d{3})* HUF$`, so it is never negative. Days with no spending show `0 HUF`.
- Do not assert individual daily values or their sum: no business rule defines them (see Open questions).
- After step 4 the table is gone and the button is **Show chart data** again.

#### 2.4 [low] Changing widgets have the right format

**Steps:**
1. Wait until "Loading accounts..." disappears.
2. Reload the page once.

**Expected results:** (before and after the reload)
- "Tip of the day" shows a non-empty sentence. Do not assert its text.
- "Exchange rate" shows `EUR/HUF` and a value matching `^\d{3}\.\d{2}$`, plus "Indicative rate. Updated on every page load."
- The footer shows "Release 1" (or the pinned `GREMLIN_RELEASE`).

---

### 3. Domestic transfer

**Seed:** `seed.spec.ts`. Every scenario opens **New transfer** from the dashboard.
"Valid form" means: From = Everyday Account, **Use Tóth Bence**, **Check IBAN** (wait for "IBAN verified: GRM-…"),
Reference = `Lunch`.

#### 3.1 [medium] An empty form shows a message for each required field

**Steps:**
1. Click **Continue** without filling anything.
2. Fill IBAN `HU71 9990 2025 1414 2135 6237 3099`, click **Check IBAN**, fill Amount `1000`, leave **Beneficiary name** empty, click **Continue**.
3. Fill **Beneficiary name** with three spaces and click **Continue**.

**Expected results:**
- Step 1: the URL stays `/transfer`. Messages: "Enter a beneficiary name.", "Check the IBAN first.", "Enter an amount greater than 0."
  Available balance shows "Available: 1,250,000 HUF" (with Savings Account selected: "5,400,000 HUF").
- Steps 2 and 3: the URL stays `/transfer` with only "Enter a beneficiary name."

#### 3.2 [high] Check IBAN accepts only valid Hungarian IBANs

**Steps:** For each row, fill **IBAN** and click **Check IBAN**.

| IBAN input | Expected status |
|---|---|
| (empty) | Invalid IBAN |
| `HU00 1234` (too short) | Invalid IBAN |
| `HU72 9990 1017 1618 0339 8874 9893` (last digit changed, bad checksum) | Invalid IBAN |
| `DE89 3704 0044 0532 0130 00` (valid German IBAN, not domestic) | Invalid IBAN |
| `HU72 9990 1017 1618 0339 8874 9892` | IBAN verified: GRM-… |
| `HU72999010171618033988749892` (no spaces) | IBAN verified: GRM-… |
| `hu72999010171618033988749892` (lower case) | IBAN verified: GRM-… |
| `␠HU72 9990 1017 1618 0339 8874 9892␠` (leading and trailing spaces) | IBAN verified: GRM-… |

#### 3.3 [high] The IBAN must be checked again after a payee is picked or the IBAN is changed

**Steps:**
1. Click **Use Kiss Péter**. Fill Amount `10000` and click **Continue** without **Check IBAN**.
2. Click **Check IBAN**, then change the IBAN to `HU71 9990 2025 1414 2135 6237 3099` and click **Continue**.

**Expected results:**
- After step 1, Beneficiary name = "Kiss Péter" and IBAN = `HU72 9990 1017 1618 0339 8874 9892`, but the form stays on `/transfer` with "Check the IBAN first."
- After step 2 the form stays on `/transfer` with "Check the IBAN first." The old check does not carry over.

#### 3.4 [medium] Amount input is validated and normalised

**Steps:** Valid form. For each row, fill **Amount (HUF)** and click **Continue**.

| Input | Expected |
|---|---|
| `0` | stays, "Enter an amount greater than 0." |
| `-5` | stays, "Enter an amount greater than 0." |
| `abc` | stays, "Enter an amount greater than 0." |
| `1.5` | stays, "Enter an amount greater than 0." (no fractions of a HUF) |
| `1e3` | stays, "Enter an amount greater than 0." |
| `1` (lower bound) | review, Amount 1 HUF |
| `1 000` | review, Amount 1,000 HUF |
| `1,000` | review, Amount 1,000 HUF |
| `00100` | review, Amount 100 HUF |

#### 3.5 [high] Fee is 0.3 % of the amount, at least 200 HUF, at most 6,000 HUF

**Steps:** Valid form with From = **Savings Account**. For each amount, click **Continue**, read Fee and Total in the review table, then click **Change details**.

**Expected results** (derived from the rule; every 0.3 % value is a whole number, so rounding plays no part):

| Amount (HUF) | Fee (HUF) | Total (HUF) | Why |
|---|---|---|---|
| 1 | 200 | 201 | minimum (0.3 % = 0.003) |
| 10,000 | 200 | 10,200 | minimum (0.3 % = 30) |
| 66,000 | 200 | 66,200 | minimum, just below the switch point (0.3 % = 198) |
| 70,000 | 210 | 70,210 | above the minimum |
| 100,000 | 300 | 100,300 | 0.3 % |
| 1,000,000 | 3,000 | 1,003,000 | 0.3 % |
| 2,000,000 | 6,000 | 2,006,000 | maximum (0.3 % = 6,000 = cap) |

Rounding (observed, confirm with the product owner before using it as an oracle):

| Amount (HUF) | Fee (HUF) | Total (HUF) | 0.3 % |
|---|---|---|---|
| 66,833 | 200 | 67,033 | 200.499 |
| 66,834 | 201 | 67,035 | 200.502 |
| 100,167 | 301 | 100,468 | 300.501 |

The 6,000 HUF cap can't be told apart from 0.3 % through the UI: the daily limit stops every amount above
2,000,000, where both give 6,000. See Open questions.

#### 3.6 [high] Insufficient funds: amount plus fee must not exceed the balance

**Steps:** Valid form, From = Everyday Account (balance 1,250,000 HUF). For each amount, fill it and click **Continue**.

**Expected results:**

| Amount | Fee | Total | Expected |
|---|---|---|---|
| 1,246,000 | 3,738 | 1,249,738 | review page (total below the balance) |
| 1,247,000 | 3,741 | 1,250,741 | stays, "Insufficient funds." |
| 1,250,000 | 3,750 | 1,253,750 | stays, "Insufficient funds." (the amount alone fits, the fee does not) |
| 1,246,261 | 3,739 | 1,250,000 | review page: total exactly equals the balance (depends on the observed rounding) |
| 1,246,262 | 3,739 | 1,250,001 | stays, "Insufficient funds." (depends on the observed rounding) |

#### 3.7 [high] The daily limit is 2,000,000 HUF of amount, including earlier transfers that day

**Steps:**
1. From Savings, amount `2000000` → **Continue**. Then **Change details**, amount `2000001` → **Continue**.
2. In a fresh context: complete a transfer of 100,000 HUF (as in 3.11). Then from Savings try `1900000`, then `1900001`.

**Expected results:**
- 2,000,000: review page (Fee 6,000, Total 2,006,000). The fee does not count toward the limit.
- 2,000,001: stays, "Daily limit of 2,000,000 HUF exceeded."
- After the 100,000 HUF transfer: 1,900,000 goes to review (Total 1,905,700). 1,900,001 gives "Daily limit of 2,000,000 HUF exceeded."

#### 3.8 [high] The per-transfer maximum of 10,000,000 HUF is enforced

**Steps:** Valid form with From = Savings Account. Amount `10000000` → **Continue**. Then amount `10000001` → **Continue**.

**Expected results:**
- 10,000,000: within the per-transfer limit but above the daily limit, so it stays with "Daily limit of 2,000,000 HUF exceeded."
- 10,000,001: above both limits. It stays with "The maximum single transfer is 10,000,000 HUF." (that the per-transfer message wins is observed, see Open questions)
- Both: no review page, no booking.
- The limits note reads "Limits: up to 10,000,000 HUF per transfer and 2,000,000 HUF per day."

#### 3.9 [high] The review page shows the entered details, and Change details keeps them

**Steps:**
1. From Everyday, **Use Nagy Eszter**, **Check IBAN**, amount `100000`, Reference `Invoice 42`, **Continue**.
2. Click **Change details**.

**Expected results:**
- URL `/transfer/review`, h1 "Review transfer". The table "Transfer details" shows: From Everyday Account; To Nagy Eszter;
  IBAN HU71 9990 2025 1414 2135 6237 3099; Amount 100,000 HUF; Fee 300 HUF; Total 100,300 HUF.
- Buttons: **Confirm transfer**, link **Change details**.
- After step 2: URL `/transfer?edit=1`. All fields keep their values (name, IBAN, `100000`, `Invoice 42`), the IBAN stays verified,
  and **Continue** goes straight back to review.
- Nothing has been booked: the dashboard still shows Everyday 1,250,000 HUF.

#### 3.10 [high] A wrong PIN, or cancelling the approval, does not send money

**Steps:** Valid form (Tóth Bence) with amount `100000` from Everyday → **Continue** to review.
1. Click **Confirm transfer** without entering a PIN.
2. Enter PIN `0000` (keyboard: focus **Confirm transfer**, Shift+Tab, type) and confirm.
3. Enter a 3-digit PIN `246` and confirm.
4. Enter `GREMLIN_PIN` and confirm. In the "Confirm payment" dialog click **Cancel**.
5. Open the dashboard.

**Expected results:**
- Steps 1 to 3: still on `/transfer/review`, alert "Wrong PIN.", no "Confirm payment" dialog.
- Step 4: the dialog opens. Its iframe "Gremlin Secure" shows "Approve this payment of 100,300 HUF" and
  "To Tóth Bence, HU03 9990 3033 1732 0508 0756 8879". After **Cancel** the dialog closes and the page stays on review.
- Step 5: Everyday balance is still 1,250,000 HUF and no new transaction is listed.

#### 3.11 [high] A confirmed transfer is booked and the balance goes down by amount plus fee

**Steps:**
1. From Everyday, **Use Tóth Bence**, **Check IBAN**, amount `100000`, Reference `Lunch`, **Continue**.
2. Enter `GREMLIN_PIN`, click **Confirm transfer**, then **Approve payment** in the dialog.
3. Click **Back to accounts**.

**Expected results:**
- URL `/transfer/done`, h1 "Transfer submitted".
- "Reference: GB-XXXXXX" matches `^Reference: GB-[A-Z0-9]{6}$`. "PIN check passed: GRM-…" is shown.
- Paid to Tóth Bence; IBAN HU03 9990 3033 1732 0508 0756 8879; Amount 100,000 HUF; Fee 300 HUF; Total 100,300 HUF;
  "New balance, Everyday Account" 1,149,700 HUF (1,250,000 − 100,300).
- Dashboard: Everyday Account 1,149,700 HUF, Savings unchanged at 5,400,000 HUF. The first recent transaction is
  today's date, "Transfer to Tóth Bence", "-100,300 HUF".
- Funds after the transfer: available amount shows 1,149,700 HUF, and the funds boundary moves: 1,146,261 → review (Total 1,149,700). 1,146,262 → "Insufficient funds."

---

## Observations and open questions (for the product owner)

These are not marked as bugs yet. Confirm the expected behaviour before writing `test.fail()` tests.

1. **The reference is dropped.** The Reference typed in the form (for example "Lunch") does not appear on the review page,
   the done page or the dashboard transaction. Expected: the reference should probably appear on review and on the receipt.
2. **The spending chart data does not change.** Today's value stays the same after a transfer, and the 30-day total
   (309,190 HUF on 2026-10-06) does not match the recent transactions. Is the chart static demo data? Until this is
   answered, scenario 2.3 checks structure and format only.
3. **The per-transfer limit cannot be reached.** 10,000,000 HUF is above the 2,000,000 HUF daily limit, so every amount
   from 2,000,001 to 10,000,000 gets the daily limit message. Only 10,000,001 and above show the per-transfer message.
4. **Usernames are case-insensitive** (upper-case `GREMLIN_USER` signs in). Is this intended?
5. **PIN length.** The PIN field takes at most 4 digits. Extra digits typed after a correct PIN are ignored and the PIN is accepted.
6. **Testability request.** The PIN is inside a closed shadow root (`<gb-secure-pin>`) and only the keyboard can reach it.
   An open shadow root or a labelled input would allow `getByLabel('Transaction PIN')`.
7. Pressing Back after a confirmed transfer shows the `/transfer/done` receipt again. No second transfer is made.
8. **Fee rounding and the 6,000 HUF cap.** Half-up rounding was observed, not specified. The cap can't be reached
   through the UI because of the daily limit. Is the cap meant for a higher limit (for example a premium account)?
9. **Which message wins** when an amount breaks both limits (10,000,001: per-transfer message) or both the funds
   and the daily limit. The plan only asserts the observed order for 10,000,001.
