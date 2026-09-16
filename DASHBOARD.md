# Trading Bot Dashboard — Design Document

**Date:** 2026-09-15 (revised, final pass — screenshot comparison)

---

## 0. What This Is

A single-page web app that reads `state.sqlite` (read-only) and shows you what the bot did today, whether it's healthy, and whether the strategy is working over time. Three views, one Python process, no external dependencies beyond what's already on the VPS.

**Design philosophy:** Professional trading-terminal density. Every pixel earns its place. The live view should feel like a prop-desk monitor — compact rows, real-time tape, equity curve — not a marketing landing page.

---

## 1. Visual Language

### 1.1 Light Mode Only

No dark mode. No toggle. The Pico CSS default light theme is the base. Override only for density and color accents.

| Token | Value | Use |
|---|---|---|
| `--bg-page` | `#ffffff` | Page background |
| `--bg-surface` | `#f8f9fa` | Card/panel backgrounds, table header row |
| `--bg-row-alt` | `#f1f3f5` | Alternating table rows |
| `--border` | `#dee2e6` | Panel borders, table lines, dividers |
| `--text-primary` | `#212529` | Body text, numbers, labels |
| `--text-secondary` | `#495057` | Muted labels, timestamps, column headers |
| `--text-muted` | `#868e96` | Disabled, "N/A", placeholders |
| `--green` | `#198754` | Profit, target hit, criteria met, positive P&L |
| `--red` | `#dc3545` | Loss, stop hit, halt active, negative P&L |
| `--blue` | `#0d6efd` | Links, active tab indicator, clickable elements |
| `--amber` | `#fd7e14` | Warnings, in-progress, marginal thresholds |

**Rules:**

- No element has a background darker than `#e9ecef`.
- Green and red appear **only** on P&L values, W/L indicators, status dots, and chart elements. Never as full-row backgrounds — use a subtle left border (`3px solid var(--green)`) on profitable trade rows instead.
- Blue is for interactive elements only: links, the active tab underline, clickable ticker symbols.
- All charts use white backgrounds with light gray gridlines (`#e9ecef`).

### 1.2 Typography and Density

- **Font:** System font stack (Pico default). Monospace (`'SF Mono', 'Cascadia Code', 'Consolas', monospace`) for all numbers, prices, quantities, timestamps, and the event tape.
- **Base font size:** 13px for table data, 12px for the event tape, 11px for secondary labels. The goal is Bloomberg-terminal density, not a blog.
- **Row height:** 28–32px in all tables. No padding luxury.
- **Number alignment:** Right-aligned in all table columns. Decimal points line up.

### 1.3 Status Indicators

| State | Indicator |
|---|---|
| Normal / profit | Small filled circle `●` in `--green` |
| Warning / marginal | Small filled circle `●` in `--amber` |
| Error / loss / halt | Small filled circle `●` in `--red` |
| Flat / inactive | Small filled circle `●` in `--text-muted` |

No emoji in production UI. Use colored dots and text only.

### 1.4 Badge / Pill Styling

Status pills and event-type badges follow a consistent pattern:

- **Pill shape:** `border-radius: 3px`, `padding: 1px 6px`, `font-size: 11px`, `font-weight: 600`, `text-transform: uppercase`, `letter-spacing: 0.3px`.
- **Variants:**
  - Default (status/info): `background: var(--bg-row-alt)`, `color: var(--text-secondary)`.
  - Pass/success: `background: rgba(25, 135, 84, 0.12)`, `color: var(--green)`.
  - Alert/entry: `background: rgba(13, 110, 253, 0.12)`, `color: var(--blue)`.
  - Warning: `background: rgba(253, 126, 20, 0.12)`, `color: var(--amber)`.
  - Error/stop: `background: rgba(220, 53, 69, 0.12)`, `color: var(--red)`.
- Use pills for: event types in the tape, pipeline status in the strategy table, scanning status indicator, session status in the top bar.

---

## 2. Views

### 2.0 Persistent Top Navigation Bar

A fixed-height bar (36px) at the very top of every view. `--bg-surface` background, 1px bottom border.

**Left side:**
- System name: `TRADING BOT` in 13px bold, `--text-primary`.
- Live indicator: `● LIVE` (green dot + text) or `● OFFLINE` (muted dot + text). The dot pulses gently (CSS animation, `@keyframes pulse`) when live.
- Book type: `SHADOW` in `--text-muted`, uppercase, 11px.
- Last update: `LAST UPDATE 09:47:09` in `--text-muted`, 11px monospace. Shows the timestamp of the most recent data fetch, not the current clock.

**Right side:**
- Tab navigation: `Live` · `Daily` · `Performance`. The active tab has a 2px bottom border in `--blue`. Inactive tabs in `--text-secondary`. No background on tabs — they live inline in the top bar itself. This replaces having a separate tab row.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TRADING BOT  ● LIVE  SHADOW  LAST UPDATE 09:47:09          Live  Daily  Perf │
└─────────────────────────────────────────────────────────────────────────────┘
```

This keeps nav permanently visible without wasting a second row on tabs.


### 2.1 Live Session (default view, 08:00–11:15 ET)

The primary monitoring view. Four horizontal bands: **nav bar** (top, §2.0), **summary hero** (below nav), **data table + activity tape** (middle, two-column), **equity curve** (bottom).

#### Summary Hero

A prominently-sized summary zone — **not** a thin single-line bar. Height: ~120–140px. `--bg-surface` background, 1px bottom border. Contains three horizontal zones:

**Left block (P&L focus, ~40% width):**

- **Daily P&L** in large, bold monospace: 36px font weight 700. Green or red. Example: `+$222.21`. This is the single most important number and should be readable from 2 meters away.
- Below it, in 12px `--text-secondary`: `of $45.00 target · 41%` — the daily target and percentage toward it.
- Below that, a thin progress bar (4px tall, full width of the left block): `--green` fill proportional to target completion. `--bg-row-alt` track.
- Below the progress bar, in 11px `--text-muted`: `2h 14m left in session` on the left, `Yesterday +$15.73 · 35% of target` on the right. This contextualizes today against yesterday.

**Center block (KPI grid, ~35% width):**

Four metrics in a 2×2 grid. Each metric: value in 18px bold monospace, label in 11px uppercase `--text-secondary` below it.

| Cell | Content | Format |
|---|---|---|
| **Win Rate** | Wins / (Wins + Losses) today | `66.7%` large, `2W 1L settled today` small underneath. Green if ≥50%, red if <50%, `--text-muted` if no trades. |
| **Fill Rate** | Filled $ / Wanted $ | `87.3%` large, `$614 filled of $703 wanted` small underneath. Answers "is the position sizer asking for more than we can get?" |
| **Trades Today** | Count + detail | `2` large, `1 open · 1 closed` small underneath. |
| **Equity** | Starting capital + cumulative P&L | `$10,018.40` large, `+ positions in 2 tickers` small underneath. |

**Right block (mini trailing chart, ~25% width):**

- **Last 7 days, net:** Label in 11px uppercase `--text-muted`.
- A tiny inline bar chart (60px tall, no axes) showing the last 7 trading days' daily P&L as green/red vertical bars. Net total shown to the right in 14px monospace, green/red.
- Below the bars, day labels in 9px `--text-muted` (e.g., `-$37`, `+$81`, `+$5`, `-$3`, `-$241`, `+$16`, `+$222`).

This trailing bar chart answers "is this a good day in context?" without navigating to the Performance tab. It's tiny but extremely useful.

When a risk halt is active, a full-width red banner (`--red` background, white text) appears **above** the summary hero: `⚠ DAILY LOSS HALT — $45 risk budget exhausted`. This is the only element with a dark/colored background.

#### Middle Section — Two Columns (≈65% / 35% split)

##### Lanes Header Row

Between the summary hero and the strategy table, a thin row (28px) with:

- Left: Section label `Lanes` in 13px bold + descriptive text in `--text-secondary` (e.g., `ORB and GAP strategies, live universe`).
- Right: A **scanning status pill** — a live-updating badge showing current pipeline state:
  - During scan: `SCANNING 147 CANDIDATES` in a `--blue` outlined pill (1px border, no fill).
  - After freeze: `5 QUALIFIED · 2 PLANNED · 3 REJECTED` in a `--text-secondary` outlined pill.
  - This pill updates at the 5s cadence. It shows pipeline progress at a glance without needing to read the tape.

##### Left: Strategy Table (multi-column data table)

This is the primary information surface. One row per candidate discovered today. Columns:

| Column | Width | Content |
|---|---|---|
| **●** | 16px | Status dot: `--green` filled = active/profitable. `--text-muted` unfilled = inactive/no trades. `--red` filled = in a losing position. Provides instant row-level status scanning. |
| **Symbol** | 80px | Ticker in bold monospace + strategy label in 11px `--text-secondary` on the same line (e.g., `PLTR ORB`). Combining these saves a column. Blue if clickable (expands detail row). |
| **Config** | 80px | Truncated with `…`, 11px `--text-muted`. Shows `live 90% to 97%` or similar. Tooltip on hover for full text. |
| **Wanted** | 60px | Shares the position sizer calculated. Right-aligned. `--text-muted` if rejected. |
| **Filled** | 60px | Shares actually filled. Right-aligned. `0` in `--text-muted` if not filled. |
| **W / L** | 90px | Two sub-cells: (1) a tiny inline **win/loss bar** — a 40px-wide horizontal bar where green segments = wins, red segments = losses, proportional to counts; (2) text `14 / 1` in monospace. This inline sparkline is extremely information-dense and lets you see the W/L ratio without reading numbers. |
| **Win %** | 50px | Running win percentage for this strategy type today. Right-aligned. |
| **Realized** | 80px | Dollar amount, right-aligned, green/red. `$0.00` in `--text-muted` if no trades. |
| **Avg Entry** | 70px | Average entry price across trades. Right-aligned. `—` if not traded. |
| **Open** | 60px | Count of currently open positions for this row. `0` in `--text-muted` if flat. |

**Table behavior:**

- Rows are sorted: open positions first (highlighted with a `3px left border` in `--blue`), then exited trades (green/red left border by P&L), then pending/planned (no border), then rejected (muted text, no border).
- Rejected rows can be collapsed/hidden via a toggle: `Show rejections (14)` / `Hide rejections`.
- Below the rejections toggle, a single line of **retirement text** in 11px `--text-muted` summarizing any strategies that were retired/disabled and why: `Retired: GAP under $1 (no edge in 25h paper; record -$6.53 over 11 trades)`. This keeps historical context visible without cluttering the table.
- Clicking a symbol row expands an inline detail panel showing: full feature vector, decision chain timestamps, entry/exit prices, stop/target, spread at entry vs. current, fill divergence.
- Alternating row backgrounds: `#fff` / `--bg-row-alt`.
- Sticky header row with `--bg-surface` background.

##### Right: Activity Tape (real-time event feed)

A vertically scrolling panel showing the last 100 events, newest at top. Monospace, 12px.

**Tape header (sticky, 32px):**

- Left: `Tape` label in 13px bold, plus a filter description: `every decision` in `--text-secondary`.
- Right: Three toggle buttons (small, 11px, pill-shaped): `SHOW PASSES` · `SHOW FILLS` · `PAUSE`.
  - `SHOW PASSES` and `SHOW FILLS` are filters — when active, the tape only shows that event type. When off, shows all events. Active state: `--blue` background, white text. Inactive: `--bg-row-alt` background, `--text-secondary` text.
  - `PAUSE` freezes auto-scroll. When active: `--amber` text. Replaces the old "pin to bottom" description.

**Tape meta-stats (below header, single line, 11px `--text-muted`):**

- `7 decisions / min` — current throughput.
- `last sweep 33s` — time since the last full pipeline sweep completed.
- `newest first` — sort indicator.

These meta-stats answer "is the bot actively processing?" without reading individual events.

**Tape entries:**

Each line is one event:

```
09:31:15  ENTRY_FILLED   PLTR   47 sh @ $23.42   stop $22.89   target $24.48
09:31:02  ENTRY_PENDING  PLTR   limit $23.45     timeout 120s
09:30:58  ORDER_SENT     PLTR   BUY 47 LIMIT $23.45
09:28:00  FREEZE         5 qualified  2 planned  3 rejected
09:27:45  RANK           PLTR #1  SOFI #2  AMD #3
09:15:03  SCAN           147 discovered  12 pass filters
09:00:01  SESSION_START  config abc1234  cash $614.20
```

**Tape formatting:**

- Timestamp in `--text-muted`.
- Event type as a **pill badge** (§1.4) instead of plain monospace text. Color by type:
  - `PASS` / `SCAN` → pass/success pill (green tint).
  - `ENTRY_FILLED` / `EXIT_FILLED` → alert/entry pill (blue tint).
  - `BORN` / `SESSION_START` → warning pill (amber tint).
  - `STOP_HIT` / `HALT` → error pill (red tint).
  - `STATUS` / `RANK` / `FREEZE` → default pill (gray).
- Payload in `--text-primary`.
- `hx-trigger="every 2s"` on this panel (faster refresh than the table).

**Between the table and tape:** A thin 1px vertical divider in `--border`.

#### Bottom: Mini Equity Curve

A Chart.js line chart, 120px tall, spanning the full width below the two columns. White background, light gray gridlines.

- **X-axis:** Today's trading minutes (09:30–11:00 ET), or all session dates if switching to cumulative mode.
- **Y-axis:** Cumulative P&L in dollars. Labeled at min, max, and zero.
- **Line:** 2px solid `--green` when above zero, `--red` when below. Fill below zero with `rgba(220, 53, 69, 0.08)` (very faint red wash).
- **Zero line:** 1px dashed `--border`.
- **No legend.** One line, self-explanatory.
- **Tooltip on hover:** Time + P&L value.
- **Target line:** Horizontal dashed line at $45.00 in `--text-muted`.

This chart answers "am I trending up or bleeding out" at a glance without taking significant vertical space.

#### Live Session — Full Layout

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TRADING BOT  ● LIVE  SHADOW  09:47:09              Live   Daily   Perf   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  +$222.21              WIN RATE    FILL RATE     LAST 7 DAYS, NET  +$42.21 │
│  of $45 target · 41%   66.7%       87.3%                ▃█▂▁  ▂█           │
│  ▓▓▓▓▓▓▓▓░░░░░░░░░░   2W 1L       $614/$703                               │
│  2h14m left             TRADES     EQUITY                                   │
│  Yest +$15.73 · 35%    2           $10,018          -37 +81 +5 -3 .. +222  │
│                         1 open      + 2 tickers                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ Lanes  ORB and GAP, live                    ┃ SCANNING 147 CANDIDATES ┃    │
├────────────────────────────────────────────────┬────────────────────────────┤
│ ●  SYM     CFG     WTD  FIL  W/L   WIN% REAL │ Tape every decision        │
│─────────────────────────────────────────────── │ SHOW PASSES · FILLS PAUSE │
│ ● PLTR ORB live..  47   47  ▓▓▓░ 14/1  93% +$32│ 7 dec/min  last sweep 33s│
│ ● TSLA GAP live..  35   35  ▓░   3/2   60% -$18│                          │
│ ● AMD  ORB live..  28   28  —    0/0    —   $0 │ 09:31:15 ┃FILL┃ PLTR 47  │
│ ○ SOFI ORB live..  40    0  —    0/0    —   $0 │ 09:31:02 ┃PEND┃ PLTR lmt │
│ ▸ Show rejections (14)                        │ 09:28:00 ┃FRZN┃ 5q 2p 3r │
│ Retired: GAP <$1 (no edge in 25h paper)       │ 09:27:45 ┃RANK┃ PLTR#1.. │
│                                               │ 09:15:03 ┃PASS┃ 147d 12p │
│                                               │ 09:00:01 ┃BORN┃ SESSION  │
├───────────────────────────────────────────────┴────────────────────────────┤
│  ╭─── Equity ───────────────────────────────────────── $45 target ─ ─ ─ ─ │
│  │          ╱╲__╱‾‾                                                       │
│  │    ╱‾‾‾╱                                                               │
│  │  ╱╱                                      $0 ─────────────────────      │
│  ╰────────────────────────────────────────────────────────────────────────│
└───────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Daily Summary (available after 11:05 ET)

What you see after the session closes. Answers: "how did today go, and does anything need attention before tomorrow?"

This is a rendered version of the `reports/YYYY-MM-DD.md` that `eod_reconcile.py` already writes, structured for scanning.

**Header cards (same summary hero position, but with more detail):**

| Card | Content |
|---|---|
| Daily P&L | Dollar amount + R-multiple, large font, green/red |
| Win/Loss | `1W 0L 1T` (wins, losses, time-barrier exits) |
| Fill Rate | Filled $ / Wanted $, with percentage |
| Trades taken / available | `2/3` |
| Expectancy (today) | R-multiple of average trade |
| Tracking error | Today's shadow P&L vs. backtest-predicted P&L. Show both numbers + delta. Green if within 0.5R, amber 0.5–1.0R, red >1.0R. |

**Trade table (same column structure as live view, all columns filled in):**

| Ticker | Signal | Entry | Exit | R | Hold (min) | Exit reason | Spread (bps) | Fill vs. Alpaca |
|---|---|---|---|---|---|---|---|---|
| PLTR | ORB | $23.42 | $24.48 | +1.4R | 18 | TARGET | 8 | +$0.03 |
| TSLA | GAP | $281.10 | $278.90 | -1.0R | 42 | STOP | 14 | -$0.12 |

Fill vs. Alpaca: green if <$0.05, amber $0.05–0.15, red >$0.15.

**Rejection breakdown (horizontal bar chart, Chart.js):**

Light gray bars on white background. Each bar labeled with count. Sorted by count descending.

```
spread_too_wide     ████████████  7
below_rvol_threshold ██████████  5
daily_loss_halt      ████        2
gap_over_100pct      ██          1
no_settled_cash      ██          1
```

This is the most underrated diagnostic. If `spread_too_wide` is rejecting 60% of candidates, the spread threshold needs examination. If `daily_loss_halt` appears early, the risk engine fired.

**System health summary:**

- Feed interruptions: count and total seconds of stale data
- Reconciliation mismatches: count (should be 0, red if not)
- 10:55 flatten: did it fire, was it clean
- Backup status: last successful `rclone sync` timestamp
- `git_sha` and `config_hash` of today's run vs. yesterday's — highlighted in `--amber` if changed

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  2026-09-15  MONDAY                                   [← Prev]  [Next →]  │
├─────────┬──────────┬──────────┬───────────┬─────────────────────────────────┤
│ P&L     │ Record   │ Trades   │Expectancy │ Tracking Error                  │
│ +$8.41  │ 1W 1L    │ 2/3      │ +0.2R     │ Shadow +$8 / BT +$11  Δ=-$3   │
├─────────┴──────────┴──────────┴───────────┴─────────────────────────────────┤
│                                                                             │
│  TRADES                                                                     │
│  Ticker  Signal  Entry     Exit     R     Hold  Reason  Spread  FillΔ      │
│  PLTR    ORB     $23.42    $24.48  +1.4R  18m   TARGET  8bps    +$0.03     │
│  TSLA    GAP     $281.10   $278.90 -1.0R  42m   STOP    14bps   -$0.12    │
│                                                                             │
│  REJECTIONS              │  SYSTEM HEALTH                                   │
│  spread_too_wide     7   │  Feed interrupts: 0                              │
│  below_rvol          5   │  Recon mismatches: 0  ●                          │
│  daily_loss_halt     2   │  10:55 flatten: clean  ●                         │
│  gap_over_100        1   │  Backup: 11:07 ET  ●                             │
│  no_settled_cash     1   │  Code: abc1234 (unchanged)                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 Performance Over Time

Answers: "is the strategy working, and should I keep going or invoke a kill criterion?"

Check this weekly or when doubting your life choices. Maps directly to the shadow-mode exit criteria in §8.1.

**Shadow-mode scoreboard (top, always visible):**

Table with `--bg-surface` background. Each row: criterion, current value, required threshold, status dot.

| Criterion | Current | Required | Status |
|---|---|---|---|
| Shadow days | 34 | ≥ 60 | ● In progress |
| Completed trades | 52 | ≥ 40 | ● Met |
| Tracking error correlation | 0.71 | ≥ 0.6 | ● Met |
| Shadow expectancy / backtest | 68% | ≥ 60% | ● Met |
| Expectancy (R) | +0.11R | CI excludes 0 | ● CI: [-0.02, +0.24] |
| Best-day robustness | +0.08R without best day | >0 | ● Met |
| Max drawdown | 8.2% | ≤ 15% | ● Met |
| Recon mismatches | 0 | 0 | ● Met |
| Duplicate orders | 0 | 0 | ● Met |
| Missed flattens | 0 | 0 | ● Met |
| Sessions without intervention | 97% | ≥ 95% | ● Met |

**Charts (stacked vertically, shared x-axis = trading days):**

All charts: white background, `#e9ecef` gridlines, `--text-secondary` axis labels, 12px monospace tick values.

1. **Cumulative P&L curve** — Shadow (2px solid `--green`) and backtest-predicted (2px dashed `--text-muted`) on the same axes. Fill between them with `rgba(0, 0, 0, 0.03)` to make tracking error visible.

2. **Daily P&L bar chart** — One bar per day, `--green` / `--red`. Overlay: 20-day rolling expectancy as a 1px `--blue` line.

3. **Drawdown chart** — Current drawdown from peak as a filled area (`rgba(220, 53, 69, 0.15)`). Horizontal dashed line at -15% threshold.

4. **Trade scatter** — Each trade as a 6px dot, x = date, y = R-multiple. Color by exit reason: `--green` = target, `--red` = stop, `--text-muted` = time barrier. Shows whether losses cluster by date (regime) or are spread evenly.

5. **Fill-model divergence** — Daily average |your_fill - alpaca_fill| in cents, as a 1px line in `--text-secondary`. Trend matters more than level.

6. **Rejection reason stacked area** — Daily counts of each rejection reason, stacked. Use a muted palette (no bright colors): `#adb5bd`, `#ced4da`, `#dee2e6`, `#e9ecef`, etc. Legend below chart.

**Below the charts:**

- Experiment log summary: total experiments run, current incumbent model version, last promotion date, last rejection reason
- Data backup: last 7 days of backup status (green dots = success, red = failure, gray = not yet)
- Kill criteria check: red/green list of §9 kill criteria

---

## 3. Tech Stack

| Component | Choice | Why |
|---|---|---|
| Backend | **FastAPI** | Already Python. Lightweight. Serves both the API and static files. |
| Frontend | **Vanilla HTML + htmx + Chart.js** | htmx gives live-updating fragments without a React app. Chart.js handles the charts. No build step, no node_modules. Total JS payload: ~200KB. |
| CSS | **Pico CSS (light theme)** | Classless/near-classless CSS framework. **Light mode only** — set `<html data-theme="light">` explicitly and never offer a toggle. Custom overrides in `style.css` for table density, monospace numbers, and the summary bar. |
| DB access | **Read-only SQLite connection** | `sqlite3.connect("file:state.sqlite?mode=ro", uri=True)`. The dashboard never writes to the bot's DB. |
| Process manager | **systemd** | One unit file. Restarts on crash. |
| Polling/refresh | **htmx `hx-trigger`** | Summary hero + strategy table: `every 5s`. Activity tape: `every 2s`. Equity curve + trailing 7-day chart: `every 10s`. Daily summary and performance views are static after load. |

**What this is not:** No WebSockets (overkill for polling by a single user), no Celery, no Redis, no message queue, no ORM (raw SQL on a read-only connection is fine), no TypeScript, no bundler.

**Total new dependencies:** `fastapi`, `uvicorn`, `jinja2`. Everything else (`sqlite3`, `json`, `datetime`) is stdlib.

### 3.1 CSS Override Strategy

Pico CSS provides the structural foundation. The custom `style.css` overrides for density and the trading-terminal look:

```css
/* style.css — overrides on Pico light theme */

:root {
  --bg-page: #ffffff;
  --bg-surface: #f8f9fa;
  --bg-row-alt: #f1f3f5;
  --border: #dee2e6;
  --text-primary: #212529;
  --text-secondary: #495057;
  --text-muted: #868e96;
  --green: #198754;
  --red: #dc3545;
  --blue: #0d6efd;
  --amber: #fd7e14;
}

html { font-size: 13px; }
html[data-theme="light"] { /* lock light mode */ }

/* Top nav bar */
.top-nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border);
  padding: 6px 16px;
  height: 36px;
  position: sticky;
  top: 0;
  z-index: 200;
}

.top-nav .system-name {
  font-size: 13px;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: 0.5px;
}

.top-nav .live-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--green);
  margin: 0 6px;
  animation: pulse 2s ease-in-out infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.top-nav .meta {
  font-size: 11px;
  color: var(--text-muted);
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
}

.top-nav .tabs a {
  color: var(--text-secondary);
  text-decoration: none;
  font-size: 13px;
  padding: 6px 12px;
  border-bottom: 2px solid transparent;
}

.top-nav .tabs a.active {
  color: var(--blue);
  border-bottom-color: var(--blue);
}

/* Summary hero */
.summary-hero {
  display: flex;
  background: var(--bg-surface);
  border-bottom: 1px solid var(--border);
  padding: 12px 16px;
  min-height: 120px;
  gap: 24px;
}

.summary-hero .pnl-block {
  flex: 0 0 40%;
}

.summary-hero .pnl-block .pnl-value {
  font-size: 36px;
  font-weight: 700;
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
  line-height: 1.1;
}

.summary-hero .pnl-block .target-line {
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 4px;
}

.summary-hero .pnl-block .progress-bar {
  height: 4px;
  background: var(--bg-row-alt);
  border-radius: 2px;
  margin-top: 6px;
  overflow: hidden;
}

.summary-hero .pnl-block .progress-fill {
  height: 100%;
  border-radius: 2px;
}

.summary-hero .pnl-block .context-line {
  font-size: 11px;
  color: var(--text-muted);
  margin-top: 6px;
  display: flex;
  justify-content: space-between;
}

.summary-hero .kpi-grid {
  flex: 0 0 35%;
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px 16px;
  align-content: center;
}

.summary-hero .kpi-grid .kpi .value {
  font-size: 18px;
  font-weight: 600;
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
}

.summary-hero .kpi-grid .kpi .label {
  font-size: 11px;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.summary-hero .kpi-grid .kpi .sub {
  font-size: 11px;
  color: var(--text-muted);
}

.summary-hero .trailing-chart {
  flex: 0 0 25%;
  text-align: right;
}

.summary-hero .trailing-chart .chart-label {
  font-size: 11px;
  text-transform: uppercase;
  color: var(--text-muted);
  letter-spacing: 0.3px;
}

.summary-hero .trailing-chart .net-value {
  font-size: 14px;
  font-weight: 600;
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
}

/* Lanes header */
.lanes-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 4px 16px;
  border-bottom: 1px solid var(--border);
  height: 28px;
}

.lanes-header .section-label {
  font-size: 13px;
  font-weight: 700;
}

.lanes-header .section-desc {
  font-size: 12px;
  color: var(--text-secondary);
  margin-left: 8px;
}

.lanes-header .scan-pill {
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.3px;
  padding: 2px 10px;
  border: 1px solid var(--blue);
  border-radius: 3px;
  color: var(--blue);
}

/* Strategy data table */
.strategy-table {
  width: 100%;
  border-collapse: collapse;
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
  font-size: 13px;
}

.strategy-table th {
  background: var(--bg-surface);
  color: var(--text-secondary);
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  padding: 4px 8px;
  border-bottom: 2px solid var(--border);
  position: sticky;
  top: 36px; /* below top nav bar */
  text-align: right;
}

.strategy-table th:first-child,
.strategy-table th:nth-child(2),
.strategy-table th:nth-child(3) {
  text-align: left;
}

.strategy-table td {
  padding: 4px 8px;
  border-bottom: 1px solid var(--border);
  text-align: right;
  line-height: 1.4;
}

.strategy-table tr:nth-child(even) {
  background: var(--bg-row-alt);
}

.strategy-table tr.open {
  border-left: 3px solid var(--blue);
}

.strategy-table tr.win {
  border-left: 3px solid var(--green);
}

.strategy-table tr.loss {
  border-left: 3px solid var(--red);
}

.strategy-table tr.rejected {
  color: var(--text-muted);
}

/* Win/loss inline sparkline bar */
.wl-bar {
  display: inline-block;
  width: 40px;
  height: 10px;
  border-radius: 2px;
  overflow: hidden;
  vertical-align: middle;
  margin-right: 4px;
}

.wl-bar .win-segment {
  display: inline-block;
  height: 100%;
  background: var(--green);
}

.wl-bar .loss-segment {
  display: inline-block;
  height: 100%;
  background: var(--red);
}

/* Status dot (in table first column) */
.status-dot {
  display: inline-block;
  width: 8px;
  height: 8px;
  border-radius: 50%;
}

.status-dot.active { background: var(--green); }
.status-dot.losing { background: var(--red); }
.status-dot.inactive {
  background: transparent;
  border: 1.5px solid var(--text-muted);
}

/* Retirement note */
.retirement-note {
  font-size: 11px;
  color: var(--text-muted);
  padding: 4px 8px;
  border-top: 1px solid var(--border);
  font-style: italic;
}

/* Activity tape */
.activity-tape {
  font-family: 'SF Mono', 'Cascadia Code', 'Consolas', monospace;
  font-size: 12px;
  line-height: 1.6;
  overflow-y: auto;
  max-height: calc(100vh - 280px);
  padding: 0 8px 8px 8px;
  background: var(--bg-page);
}

.tape-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 4px 8px;
  position: sticky;
  top: 0;
  background: var(--bg-page);
  border-bottom: 1px solid var(--border);
  z-index: 10;
}

.tape-header .label {
  font-size: 13px;
  font-weight: 700;
}

.tape-header .filter-desc {
  font-size: 12px;
  color: var(--text-secondary);
  margin-left: 6px;
}

.tape-filter-btn {
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  padding: 2px 8px;
  border-radius: 3px;
  border: none;
  cursor: pointer;
  background: var(--bg-row-alt);
  color: var(--text-secondary);
}

.tape-filter-btn.active {
  background: var(--blue);
  color: #ffffff;
}

.tape-filter-btn.pause.active {
  background: transparent;
  color: var(--amber);
  font-weight: 700;
}

.tape-meta {
  font-size: 11px;
  color: var(--text-muted);
  padding: 2px 8px;
  display: flex;
  gap: 16px;
}

.activity-tape .event {
  white-space: nowrap;
  padding: 1px 0;
}

.activity-tape .ts {
  color: var(--text-muted);
}

.activity-tape .event-type {
  display: inline-block;
  border-radius: 3px;
  padding: 1px 6px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}

/* Event type pill variants */
.event-type.pass    { background: rgba(25, 135, 84, 0.12); color: var(--green); }
.event-type.fill    { background: rgba(13, 110, 253, 0.12); color: var(--blue); }
.event-type.born    { background: rgba(253, 126, 20, 0.12); color: var(--amber); }
.event-type.error   { background: rgba(220, 53, 69, 0.12); color: var(--red); }
.event-type.status  { background: var(--bg-row-alt); color: var(--text-secondary); }

/* Equity curve container */
.equity-curve {
  height: 120px;
  background: var(--bg-page);
  border-top: 1px solid var(--border);
  padding: 8px 16px;
}

/* P&L colors */
.pnl-positive { color: var(--green); }
.pnl-negative { color: var(--red); }
.pnl-neutral  { color: var(--text-muted); }

/* Halt banner — the ONLY dark background in the entire UI */
.halt-banner {
  background: var(--red);
  color: #ffffff;
  text-align: center;
  padding: 6px 16px;
  font-weight: 600;
  font-size: 13px;
}

/* Badge / pill generic */
.pill {
  display: inline-block;
  border-radius: 3px;
  padding: 1px 6px;
  font-size: 11px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.3px;
}
```

---

## 4. Data Flow

```
state.sqlite (WAL mode, written by bot processes)
    │
    │  read-only connection, opened per-request
    │  (WAL mode allows concurrent reads while bot writes)
    │
    ▼
dashboard.py (FastAPI)
    │
    ├── /api/live       → queries decisions, positions, orders, cash_ledger
    │                     for session_date = today
    │                     Returns: summary metrics + strategy table rows
    │                              + activity tape events + equity curve points
    │
    ├── /api/tape       → last 100 decision/event rows for today
    │                     (separate endpoint for faster 2s polling)
    │
    ├── /api/equity     → intraday P&L timeseries for today
    │                     (separate endpoint for 10s chart refresh)
    │
    ├── /api/trailing7  → last 7 trading days' daily P&L
    │                     (separate endpoint for 10s refresh of
    │                      the trailing bar chart in the summary hero)
    │
    ├── /api/daily/{date} → queries same tables + joins fill divergence
    │                       from alpaca_fills table
    │
    ├── /api/performance  → queries across all session_dates
    │                       computes rolling stats, drawdown, CI
    │
    ├── /api/health       → last quote timestamp, backup status file,
    │                       git sha from latest decisions row
    │
    └── serves static HTML/JS/CSS from dashboard/static/
```

**htmx fragment endpoints (return HTML, not JSON):**

```
/fragments/summary-hero      → full summary hero HTML (polled every 5s)
/fragments/lanes-header      → scanning status pill (polled every 5s)
/fragments/strategy-table    → full table body (polled every 5s)
/fragments/tape              → activity tape entries (polled every 2s)
/fragments/tape-meta         → tape throughput stats (polled every 5s)
/fragments/equity-data       → JSON array for Chart.js (polled every 10s)
/fragments/trailing7-data    → JSON array for trailing bar chart (polled every 10s)
```

**Key queries the dashboard needs (all read-only):**

```sql
-- Live: strategy table rows
SELECT d.ticker, d.signal_type, d.gap_pct, d.rvol,
       d.stage, d.outcome, d.reason,
       p.wanted_qty, p.filled_qty,
       CASE WHEN p.realized_pnl_cents > 0 THEN 'W'
            WHEN p.realized_pnl_cents < 0 THEN 'L'
            WHEN p.realized_pnl_cents = 0 AND p.status='CLOSED' THEN 'T'
            ELSE NULL END as wl,
       p.realized_pnl_cents,
       p.unrealized_pnl_cents,
       p.r_multiple,
       p.exit_reason
FROM decisions d
LEFT JOIN positions p ON d.ticker = p.ticker AND d.session_date = p.session_date
WHERE d.session_date = ?
ORDER BY
  CASE WHEN p.status = 'OPEN' THEN 0
       WHEN p.status = 'CLOSED' THEN 1
       WHEN d.stage IN ('PLANNED','RANKED') THEN 2
       ELSE 3 END,
  d.ts_utc DESC;

-- Live: summary hero metrics
SELECT
  SUM(CASE WHEN p.realized_pnl_cents > 0 THEN 1 ELSE 0 END) as wins,
  SUM(CASE WHEN p.realized_pnl_cents < 0 THEN 1 ELSE 0 END) as losses,
  SUM(p.realized_pnl_cents) as total_realized,
  SUM(p.unrealized_pnl_cents) as total_unrealized,
  COUNT(CASE WHEN p.status IN ('OPEN','CLOSED') THEN 1 END) as trades_taken,
  SUM(p.wanted_qty * p.avg_entry_cents) as total_wanted_value,
  SUM(p.filled_qty * p.avg_entry_cents) as total_filled_value
FROM positions p WHERE p.session_date = ?;

-- Live: activity tape
SELECT ts_utc, event_type, ticker, payload
FROM events
WHERE session_date = ?
ORDER BY ts_utc DESC
LIMIT 100;

-- Live: tape throughput meta-stats
SELECT
  COUNT(*) as events_last_60s,
  MAX(CASE WHEN event_type = 'SWEEP' THEN ts_utc END) as last_sweep
FROM events
WHERE session_date = ?
  AND ts_utc >= datetime('now', '-60 seconds');

-- Live: intraday equity curve
SELECT ts_utc,
       SUM(realized_pnl_cents + unrealized_pnl_cents) OVER (ORDER BY ts_utc) as cumulative_pnl
FROM pnl_snapshots
WHERE session_date = ?
ORDER BY ts_utc;

-- Live: trailing 7-day bar chart
SELECT session_date, SUM(realized_pnl_cents) as daily_pnl
FROM positions
WHERE session_date >= date(?, '-10 days')
  AND status = 'CLOSED'
GROUP BY session_date
ORDER BY session_date DESC
LIMIT 7;

-- Live: feed staleness
SELECT MAX(ts_utc) FROM quotes_log;

-- Live: settled cash
SELECT SUM(amount_cents) FROM cash_ledger
WHERE settles_on <= ?;

-- Live: scanning pipeline status
SELECT
  COUNT(*) as total_candidates,
  SUM(CASE WHEN stage = 'QUALIFIED' THEN 1 ELSE 0 END) as qualified,
  SUM(CASE WHEN stage = 'PLANNED' THEN 1 ELSE 0 END) as planned,
  SUM(CASE WHEN outcome = 'REJECTED' THEN 1 ELSE 0 END) as rejected
FROM decisions
WHERE session_date = ?;

-- Daily: rejection histogram
SELECT reason, COUNT(*) FROM decisions
WHERE session_date = ? AND outcome = 'REJECTED'
GROUP BY reason ORDER BY COUNT(*) DESC;

-- Performance: daily P&L series
SELECT session_date,
       SUM(realized_pnl_cents) as daily_pnl,
       COUNT(*) as trade_count
FROM fills
GROUP BY session_date
ORDER BY session_date;

-- Performance: tracking error
SELECT session_date, shadow_pnl, backtest_predicted_pnl
FROM daily_reconciliation
ORDER BY session_date;
```

**One addition to the bot's schema:** The dashboard needs a `daily_reconciliation` table (or it reads the `reports/YYYY-MM-DD.md` files). Since `eod_reconcile.py` already computes the tracking-error numbers, have it write a summary row:

```sql
CREATE TABLE daily_reconciliation (
    session_date       TEXT PRIMARY KEY,
    shadow_pnl_cents   INTEGER,
    backtest_pnl_cents INTEGER,
    trades_taken       INTEGER,
    trades_available   INTEGER,
    wins               INTEGER,
    losses             INTEGER,
    time_exits         INTEGER,
    max_drawdown_cents INTEGER,
    feed_interruptions INTEGER,
    recon_mismatches   INTEGER,
    flatten_clean      BOOLEAN,
    backup_ok          BOOLEAN,
    git_sha            TEXT,
    config_hash        TEXT
);
```

This is the only schema addition. Everything else reads existing tables.

---

## 5. Deployment and Access

### Directory Structure

```bash
trading-bot/
├── strategy/
├── scripts/           # universe_build.py, session_runner.py, etc.
├── dashboard/
│   ├── dashboard.py   # FastAPI app
│   ├── templates/     # Jinja2 templates
│   │   ├── base.html  # <html data-theme="light">, top nav bar, includes
│   │   ├── live.html  # Summary hero + lanes header + strategy table + tape + equity curve
│   │   ├── daily.html # Post-session summary
│   │   └── perf.html  # Performance over time
│   ├── static/
│   │   ├── htmx.min.js
│   │   ├── chart.min.js
│   │   └── style.css  # Light-mode overrides (§3.1)
│   └── queries.py     # All SQL queries, named, read-only
└── state.sqlite
```

### systemd unit

```ini
# /etc/systemd/system/trading-dashboard.service
[Unit]
Description=Trading Bot Dashboard
After=network.target

[Service]
Type=simple
User=trading
WorkingDirectory=/home/trading/trading-bot
ExecStart=/home/trading/trading-bot/.venv/bin/uvicorn dashboard.dashboard:app --host 127.0.0.1 --port 8271
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.default
```

Port 8271 (arbitrary, high, unlikely to conflict). Binds to `127.0.0.1` only — not exposed to the internet.

### Access via Tailscale

1. Install Tailscale on the VPS and on your laptop. `tailscale up` on both.
2. Dashboard binds to `127.0.0.1:8271`.
3. Access from your laptop: `http://<vps-tailscale-ip>:8271`.
4. No port forwarding, no nginx, no TLS cert management. Tailscale's WireGuard tunnel handles encryption.

If you don't want Tailscale, an SSH tunnel works identically:
```bash
ssh -L 8271:localhost:8271 trading@your-vps
# then open http://localhost:8271
```

### Authentication

For a solo developer over Tailscale, the Tailscale ACL *is* your auth — only your devices can reach the VPS's Tailnet IP.

If you're paranoid (reasonable), add HTTP Basic Auth in FastAPI as a middleware:

```python
# In dashboard.py — 8 lines, not a framework
import secrets
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBasic, HTTPBasicCredentials

security = HTTPBasic()

def verify(credentials: HTTPBasicCredentials = Depends(security)):
    if not (secrets.compare_digest(credentials.username, DASH_USER)
            and secrets.compare_digest(credentials.password, DASH_PASS)):
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
```

Do not add OAuth, JWT, or session management. You are the only user.

---

## 6. Build Order

3–4 days of work, done during or after week 8 (VPS deployment).

| Day | What | Why this order |
|---|---|---|
| 1 | `queries.py` + `dashboard.py` with `/api/live`, `/api/tape`, `/api/equity`, `/api/trailing7`, `/api/health` endpoints returning JSON. Test with `curl`. | Get the data layer right first. If your queries are wrong, the UI doesn't matter. |
| 1 | Add the `daily_reconciliation` table to `eod_reconcile.py`. | The dashboard needs this; add it while you're looking at the schema. |
| 2 | `base.html` with Pico CSS (light theme locked), top nav bar, `style.css` overrides. Live session template: summary hero (with trailing 7-day chart), lanes header with scanning pill, strategy table with W/L sparklines and htmx 5s refresh, activity tape with filter tabs and throughput meta-stats on 2s refresh, equity curve with Chart.js 10s refresh. | This is the view you'll use most. The summary hero and tape filters are the biggest visual upgrades from the screenshot comparison. |
| 2 | Daily summary template. | Second most-used view. |
| 3 | Performance view with Chart.js charts (all 6). | Most complex, but you don't need it until you have >10 days of data. |
| 3 | systemd unit, Tailscale setup, basic auth. | Deploy last, after it works locally. |

---

## 7. What Not to Build

- **No dark mode.** Light backgrounds only. You're monitoring this in daylight hours (pre-market through 11 AM). If you want dark, open TradingView on another tab.
- **No real-time price chart.** You have TradingView open anyway. The dashboard shows the *system's* view, not the market.
- **No alerting from the dashboard.** Alerts go through ntfy.sh / Telegram (§10.3 of the bot design). The dashboard is for looking, not pushing.
- **No write operations.** The dashboard never modifies `state.sqlite`. No "cancel trade" buttons, no "override halt" toggles. If you need to intervene, ssh in. The dashboard is a window, not a control panel.
- **No historical trade replay or chart annotation.** This is a monitoring tool, not a research notebook. Research stays in DuckDB and Jupyter.
- **No mobile optimization.** You're looking at this on a laptop. If it works on a laptop, ship it.
- **No user management, roles, or permissions.** One user. One password. Done.

---

## Appendix: Screenshot Comparison Notes

Changes made in this final revision after comparing against a professional trading dashboard screenshot (dark-mode betting/prediction platform). Key patterns adopted, adapted to light mode:

1. **Summary hero instead of summary bar.** The screenshot's top zone is ~150px tall with a massive P&L number, progress bar, time remaining, and yesterday's context. Upgraded from a thin single-row bar to a full hero zone with 36px P&L, progress bar, session countdown, and yesterday comparison.

2. **Trailing 7-day inline bar chart.** The screenshot embeds a tiny daily P&L bar chart in the top-right of the summary area. Added as the "right block" of the summary hero.

3. **Persistent top nav with live indicator.** The screenshot has a dedicated header row with system name, live dot, book type, last update timestamp, and nav links. Added as §2.0 — separates navigation from content.

4. **Fill Rate as a first-class KPI.** The screenshot shows fill rate prominently (65.5%, dollars filled vs wanted). Added to the KPI grid.

5. **Win/loss inline sparkline bars.** The screenshot shows tiny colored bars inline in the W/L column (green segments for wins, red for losses). Added `.wl-bar` component to strategy table rows.

6. **Tape filter tabs and throughput meta-stats.** The screenshot has SHOW PASSES / SHOW BOOKS / PAUSE buttons above the tape, plus "7 decisions / min" and "last sweep 33s" meta-stats. Added tape header with filter toggles and throughput line.

7. **Event type pill badges.** The screenshot uses colored pill badges (PASS, BORN, SWEEP, STATUS) for event types instead of plain monospace. Added §1.4 pill styling and applied to tape events.

8. **Scanning pipeline status pill.** The screenshot shows "SCANNING 94 CANDIDATES" as a prominent outlined pill. Added lanes header row with live-updating scanning pill.

9. **Strategy lane status dots.** The screenshot uses filled/unfilled colored indicators per strategy row. Added status dot column (●/○) to table.

10. **Retirement notes.** The screenshot includes a "Retired:" text block explaining why certain strategies are disabled. Added retirement note below the rejections toggle.
