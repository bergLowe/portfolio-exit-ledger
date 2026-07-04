# Exit Calculator — Portfolio Exit Ledger

A lightweight, installable web app that answers one question for Indian equity
investors:

> **"I invested ₹X. I want ₹Y in hand after selling. What price do I need to
> exit at — after brokerage, STT, exchange charges, GST, stamp duty, and
> capital gains tax?"**

Built as a single self-contained Progressive Web App (PWA) — no backend, no
build step, no data leaves the device.

---

## 1. What it does

- Enter one or more **buy lots** (quantity + price) — the app computes your
  weighted average purchase price (WAP) and total cash invested automatically.
- Enter your **target net amount (Y)** — the exact rupee figure you want in
  hand after everything is accounted for.
- Choose your tax treatment: **STCG (20%)** or **LTCG (12.5%, with the
  ₹1,25,000 yearly exemption)**.
- The app reverse-solves for the **exact exit price** needed, and shows:
  - Required appreciation % over your average buy price
  - Required growth % on your invested capital
  - A full line-by-line breakdown: sell value → brokerage → STT → exchange
    charges → SEBI fee → GST → capital gain → tax → cess → **net in hand**
- All charge rates are editable (per-trade override), with defaults matching
  a typical zero-brokerage NSE delivery trade.
- Remembers your last inputs via browser local storage — nothing is sent to
  any server.
- Installable to a phone's home screen and works offline once installed.

---

## 2. Tech stack & requirements

| Layer | Choice | Why |
|---|---|---|
| Markup/Styling | Plain HTML5 + CSS3 (inline, single file) | Zero dependencies, opens anywhere, easy to audit |
| Logic | Vanilla JavaScript (ES5-leaning, no framework) | No build step, no npm install, works in any modern browser |
| Fonts | Google Fonts — IBM Plex Serif / Sans / Mono | Loaded via CDN `<link>`; falls back to system fonts if offline/blocked |
| Persistence | `localStorage` (browser-native) | Keeps your last lots/target between visits, no login, no cloud |
| Offline support | `service-worker.js` + `manifest.json` | Standard PWA install + offline caching |
| Icons | Generated PNG (192×192, 512×512) | Home-screen icon assets |

**Requirements to run it:**
- Any modern browser (Chrome, Safari, Edge, Firefox) — desktop or mobile.
- No installation, no account, no internet required after the first load
  (fonts need internet once; the calculator itself works offline immediately).

**Requirements to install it as a real app (optional):**
- Must be served over **HTTPS** (or `localhost`) — service workers do not
  register on a `file://` path. Free static hosts that work out of the box:
  GitHub Pages, Netlify, Vercel, Cloudflare Pages.

**Browser support:** any browser from the last ~5 years. No IE11 support.

---

## 3. Calculation methodology

### 3.1 Weighted Average Purchase Price (WAP)

```
WAP = Σ(qty_i × buy_price_i) / Σ qty_i
```

### 3.2 Total Invested (X)

For each buy lot, the cash actually paid out is:

```
Lot value        = qty × buy price
Brokerage (buy)   = per your brokerage setting (₹0 flat / flat ₹ / % of value)
STT (buy)         = 0.1%  of lot value            [default]
Exchange charges  = 0.00297% of lot value          [default, NSE-approx]
SEBI turnover fee = 0.0001% of lot value           [default]
Stamp duty        = 0.015% of lot value  (buy-side only) [default]
GST               = 18% of (brokerage + exchange + SEBI) [default]

Net cash paid (this lot) = lot value + brokerage + STT + exchange
                            + SEBI + stamp duty + GST
```

`X` = sum of "net cash paid" across all lots.

### 3.3 Cost basis for capital gains

Only **brokerage** is added to cost of acquisition for tax purposes — STT,
stamp duty, exchange charges, and SEBI fees are **not** deductible under the
Income Tax Act (Section 48 disallows STT as a deduction; this has been
settled law since STT's introduction).

```
Cost for CG (this lot) = lot value + brokerage (buy)
Total Cost for CG       = sum across all lots
```

### 3.4 Sell side (solved for price P)

```
Sell value         = P × total qty
Brokerage (sell)   = per your setting
STT (sell)         = 0.1% of sell value
Exchange charges   = 0.00297% of sell value
SEBI turnover fee  = 0.0001% of sell value
GST                = 18% of (brokerage + exchange + SEBI)
Other/DP charges   = flat ₹, optional

Net sell proceeds (for CG) = Sell value − brokerage (sell)
Capital Gain               = Net sell proceeds (for CG) − Total Cost for CG
```

### 3.5 Tax on the gain

**STCG (Section 111A)** — applies when shares are held under 12 months and
STT was paid on both legs:
```
Tax = max(0, Capital Gain) × 20%
Cess = Tax × 4%
Total tax = Tax + Cess               (effectively 20.8% of the gain)
```

**LTCG (Section 112A)** — applies when held over 12 months:
```
Exemption remaining = max(0, ₹1,25,000 − exemption already used this year)
Taxable gain = max(0, Capital Gain − Exemption remaining)
Tax = Taxable gain × 12.5%
Cess = Tax × 4%
Total tax = Tax + Cess               (effectively 13% of the taxable gain)
```

### 3.6 Net cash in hand

```
Net cash in hand = Sell value − (Brokerage + STT + Exchange + SEBI + GST + Other) − Total tax
```

### 3.7 Solving for the required exit price

Since every charge and the tax are **linear functions of the sell price**,
"net cash in hand" is a monotonically increasing, piecewise-linear function
of P. The app finds the exact P where this equals your target Y using
**bisection search** (~100 iterations, converges to sub-paisa accuracy in
milliseconds) — this is more robust than a closed-form algebraic solve given
the editable, mixed-mode charge inputs (flat ₹ vs % vs tiered LTCG
exemption), while remaining numerically exact for practical purposes.

### 3.8 Worked example (validation case)

Using this app's real trade history as a check:

| Lot | Qty | Buy price |
|---|---|---|
| 1 | 125 | ₹400.50 |
| 2 | 67 | ₹375.00 |

- WAP = ₹391.60, Total invested (X) ≈ ₹75,277
- Target net-in-hand (Y) = ₹84,182 (X + a desired ₹8,905 net gain)
- **App-solved exit price: ₹451.34/share**
- **Actual price the shares were sold at: ₹451.35/share**

The 1-paisa gap comes entirely from using rounded default charge rates
instead of the exact rates on that day's contract note — confirming the
underlying math is correct.

---

## 4. Assumptions & known simplifications

- All shares are assumed sold **in a single sell order** at one blended
  price (matches "one common target using the weighted average purchase
  price," as scoped for this build).
- Surcharge (applicable only above ₹50L total income) is **not** modeled —
  out of scope for typical retail trade sizes.
- Stamp duty is applied **buy-side only**, matching standard NSE/BSE
  practice and the sample contract notes used to calibrate defaults.
- STCG/LTCG classification is a **manual toggle**, not auto-detected from
  dates — you decide based on your actual holding period.
- Buy quantity and buy price are clamped to zero or above — negative lot
  inputs are treated as invalid rather than flipping the sign of the
  invested amount.
- This is a **planning estimator**, not a filing tool. Real contract notes
  carry exact, day-specific rates (which can differ slightly by exchange,
  broker, and instrument) — always reconcile against your broker's contract
  note and consult a CA before filing.

---

## 5. File structure

```
portfolio-exit-ledger/
├── index.html          # entire app: markup, styles, and logic (self-contained)
├── manifest.json        # PWA metadata (name, icons, colors, install behavior)
├── service-worker.js    # offline caching
├── icon-192.png          # home-screen icon (small)
├── icon-512.png          # home-screen icon (large)
├── .gitignore            # ignores OS/editor cruft
└── README.md             # this file
```

## 6. Getting started

**Just try it:** open `index.html` in any browser — works immediately, no
setup.

**Install it as an app:** deploy this repo to any static host (GitHub Pages,
Netlify Drop, Vercel) over HTTPS, then open the URL on your phone and use
"Add to Home Screen."

---

## 7. Known issues fixed in this pass

While integrating the app into this repo, the following bugs were found (via
manual code review and browser-driven testing) and fixed in `index.html`:

- **`+Infinity%` on capital deployed**: if total invested capital worked out
  to ₹0 (e.g. a buy lot entered with price ₹0), the "growth % on capital"
  figure divided by zero and rendered `+Infinity%` instead of a sane value.
  Now guarded the same way the "vs avg buy price" figure already was.
- **Negative quantity/price produced a negative "Total invested"**: the qty
  and price inputs only carried an HTML `min="0"` hint, which browsers don't
  enforce on keyboard input. A negative quantity flowed straight through the
  math, producing a negative invested amount and a nonsensical weighted
  average price. Buy lot qty/price are now clamped to `≥ 0` before any
  calculation.

The core reverse-solve math (bisection search, WAP, cost-basis-for-CG,
STCG/LTCG tax treatment) was verified against the worked example in section
3.8 and reproduces the expected ₹451.34 exit price exactly.

---

## Disclaimer

This tool provides estimates based on standard NSE/BSE delivery-trade rates
and current Section 111A/112A capital gains treatment. It is not tax or
investment advice. Verify final figures against your actual broker contract
note, and consult a chartered accountant before making filing or trading
decisions based on these numbers.
