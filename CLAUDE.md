# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Workflow Orchestration

### 1. Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan immediately - don't keep pushing
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Subagent Strategy
- Offload research, exploration, and parallel analysis to subagents to keep main context window clean
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution

### 3. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until mistake rate drops
- Review lessons at session start for relevant project

### 4. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

### 5. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes - don't over-engineer
- Challenge your own work before presenting it

### 6. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests -> then resolve them
- Zero context switching required from the user
- Go fix failing CI tests without being told how

## Task Management

1. **Plan First**: Write plan to `tasks/todo.md` with checkable items
2. **Verify Plan**: Check in before starting implementation
3. **Track Progress**: Mark items complete as you go
4. **Explain Changes**: High-level summary at each step
5. **Document Results**: Add review to `tasks/todo.md`
6. **Capture Lessons**: Update `tasks/lessons.md` after corrections

## Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid introducing bugs.

---

## Project Overview

TomorrowTradeBias is a self-contained HTML web application that predicts market directional bias using the TFO (Trade From Open) methodology by TTrades. It analyzes OHLC data to determine whether price will draw on liquidity toward previous period highs or lows, displaying statistical success rates.

The full specification lives in `ttbias.txt`.

## Architecture

**Target**: Single HTML file (~2500 lines) or minimal 3-file split (index.html + css/styles.css + js/*.js). No build system, no npm — pure browser JavaScript with CDN dependencies only.

### CDN Dependencies
- **Google Fonts**: Inter (UI) + JetBrains Mono (data/monospace)
- **Lightweight Charts 4.1.3**: TradingView's charting library (`unpkg.com/lightweight-charts@4.1.3`)

### Data Flow
1. **Fetch**: Yahoo Finance (primary, via `corsproxy.io` CORS proxy) → Tradier API (fallback, needs token) → CSV upload (manual)
2. **Store**: IndexedDB (`TomorrowTradeBias` database) with `ohlcData` and `metadata` stores
3. **Calculate**: TFO bias algorithm compares current bar's close/high/low against previous period high (PPH) and previous period low (PPL)
4. **Display**: Bias hero card + probability matrix (30/60/90-bar success rates) + target zone metrics + swing metrics (ADR, MA bias, confluence) + day-of-week stats + candlestick chart with MA overlays

### Core Algorithm (TFO Bias Rules — order matters)
1. Close > PPH → BULLISH
2. Close < PPL → BEARISH
3. High > PPH but Close < PPH → BEARISH (failed breakout high)
4. Low < PPL but Close > PPL → BULLISH (failed breakout low)
5. Else → NEUTRAL (inside bar)

**Success validation**: BULLISH signal "hits" if next bar's high >= PDH; BEARISH hits if next bar's low <= PDL. NEUTRAL signals are excluded from statistics.

### Storage
- **IndexedDB**: OHLC data persistence (compound key: `[symbol, date]`), metadata per symbol
- **LocalStorage**: Settings only (Tradier token, selected timeframe, last symbol)

### Weekly Conversion
Daily bars aggregate to weekly using ISO week (Monday start). Open = Monday's open, High/Low = max/min of week, Close = Friday's close, Volume = sum.

## Key Design Constraints

- **Attribution is mandatory**: Footer must always credit TTrades Daily Bias (TFO) with link and @t2make with link
- Dark theme with neon mint (#00FFA3) for bullish, muted coral (#FF6B6B) for bearish, slate (#64748B) for neutral
- Chart displays last 60 bars with bias marker on current bar, PDH/PDL reference lines, and MA overlays (8 EMA, 21 EMA, 50 SMA, 200 SMA)
- Initial data load: 365 days; must complete in <15 seconds; cached loads <1 second
- Responsive: 3-column grid on desktop, 2-column on tablet (≤1280px), single column on mobile (≤768px)
- ES6+ JavaScript, no jQuery or frameworks

## Swing Trade Metrics

The app includes additional swing trade analysis beyond the core TFO bias:

- **ADR (Average Daily Range)**: ADR(5), ADR(20), ADR%, expansion ratio, and remaining ADR bar. Detects compression (ADR5 < 75% of ADR20) and overextension (expansion > 1.5x).
- **MA Bias Score**: Multi-timeframe scoring (-8 to +8) based on price position relative to 8 EMA, 21 EMA, 50 SMA, 200 SMA on both daily and weekly. Labels: Strongly Bullish/Bearish, Bullish/Bearish Lean, Choppy/Neutral.
- **TF Confluence**: Whether daily and weekly TFO bias signals agree (Aligned Bull/Bear), disagree (Divergent), or include neutral (Mixed).
- **Day-of-Week Stats**: TFO hit rates broken down by weekday (Mon-Fri), highlighting best-performing days.
- **Consecutive Closes**: Count of sequential up/down closes from the most recent bar.
- **Relative Volume**: Current bar volume / 20-bar average. Displayed as ratio with bar visualization.

All swing metrics use the same OHLC data already fetched — no additional API calls.

## Build Order (recommended phases)

1. HTML structure + CSS
2. IndexedDB manager
3. Data fetchers (Yahoo → Tradier → CSV)
4. Bias calculator
5. Timeframe converter (daily↔weekly)
6. UI controller
7. Chart integration (Lightweight Charts)
8. Modals & settings
9. Error handling & loading states
10. Testing & polish

## Development

No build tools or package managers. To develop:
- Edit the HTML/CSS/JS files directly
- Open `index.html` in a browser (Chrome 90+, Edge 90+, Firefox 88+, Safari 14+)
- No server required — works via `file://` protocol after initial data fetch

## Symbol Formats

Yahoo Finance format: `NQ=F` (futures), `EURUSD=X` (forex), `AAPL` (stocks), `BTC-USD` (crypto)
