# Yourfolio

**A personal investment portfolio tracker that answers a sharper question than most: not just "what do I own?" but "what did my decisions actually earn?"**

Yourfolio lets you record every stock purchase you make — across both US and Korean markets — and see your real gain or loss per position and across the whole portfolio, with allocation, concentration risk, and currency conversion handled in one clean view.

🔗 **[Live demo](https://ghunjin.github.io/yourfolio/)** · Built with vanilla HTML, CSS, and JavaScript. No frameworks, no build step, no dependencies.

> **Note on the demo:** the landing page shows a sample portfolio so you can see the tool in context. The tracker itself starts empty — add a holding to see it work. Your data is saved locally in your own browser and is never sent anywhere.

---

## Why this project exists

Most people who invest can't actually tell you their return. They know roughly what they own and have a vague sense of whether they're "up," but the precise number — *how much did my choices earn?* — is buried in brokerage statements or a spreadsheet they stopped updating.

Yourfolio is built around that gap. The goal was never to clone a brokerage app; it was to build the smallest tool that turns "stuff I own" into "a clear answer about how I'm doing," and to make every design choice serve that clarity.

## What it does

- **Transaction log as the source of truth.** Every buy is recorded individually and preserved. The portfolio summary — positions, average cost, totals, allocation — is *derived* from the log, never stored.
- **Real gain/loss per position and overall**, in money and percent, with weighted-average cost basis across repeat purchases.
- **Dual-currency portfolios (USD + KRW)** with a single exchange rate and a display toggle, so US and Korean holdings total cleanly in one view.
- **Allocation donut** (hand-drawn SVG, no chart library) with a consistent color per ticker across every page.
- **Concentration warning** that scales with portfolio size — it flags outsized single positions without nagging small, still-growing portfolios.
- **A full transactions page** where any individual buy can be edited or deleted, with the summary recomputing automatically.
- **Local-only persistence.** Everything lives in the browser's `localStorage`. No account, no server, no data leaves the device.

## Design decisions and tradeoffs

This section is the heart of the README. Every feature above is the result of a decision that had a cost, and I want those costs to be visible.

### 1. Transactions, not positions

**Decision:** the stored data is an append-only list of individual buys. Positions (shares held, average cost, dates) are computed on demand by `derivePositions()`.

**Why:** the first version stored merged positions directly, and it kept creating small contradictions — the row needed the *most recent* purchase date while a future value-over-time chart needs the *earliest*, and a merged position can only remember one. A log remembers everything, so every view derives exactly what it needs. It also matches how real financial systems work: events are facts; summaries are opinions computed from facts.

**Tradeoff:** it was a genuine refactor — the summary page became read-only, editing moved to a dedicated transactions page, and old users' data needed a one-time migration path (which the app still carries). More moving parts in exchange for a model that can't self-contradict.

### 2. Currency handling that refuses to guess

**Decision:** each ticker has exactly one native currency, and the app **blocks** any action that would require a currency conversion when no exchange rate has been entered — you can't add a KRW holding while viewing in USD, or flip the display currency, until you provide today's rate.

**Why:** the earlier behavior silently treated a missing rate as 1:1, which displayed a ₩50,000 stock as $50,000. That's not a rounding issue; it's a wrong answer wearing the same font as a right one. In a tool whose entire purpose is a trustworthy number, failing loudly beats guessing quietly.

**Tradeoff:** blocking adds friction — one extra required input before certain actions. And the converted view is still labeled as approximate (an in-app notice explains that converting historical cost basis at today's rate is inherently fuzzy), because pretending otherwise would be a quieter version of the same lie.

### 3. A concentration warning that scales with context

**Decision:** the "one position is too big" warning uses a sliding threshold: no warning under 3 holdings, 70% at exactly 3, 50% at 4, 40% at 5 or more.

**Why:** a fixed 40% threshold false-alarmed constantly on small portfolios — if you own two stocks, one of them being over 40% is just arithmetic, not risk-taking. The threshold now encodes what the warning is actually *for*: distinguishing "still building a portfolio" from "making a concentrated bet."

**Tradeoff:** the thresholds are judgment calls, not finance theory, and I can defend the shape of the curve more confidently than any specific number. There's also an exemption hook (`isDiversified`) already in place so that, once live data can identify index ETFs, a diversified fund never trips a warning designed for single-company risk.

### 4. Warnings the user can silence — per ticker

**Decision:** the concentration warning has two dismissals: an × that hides it for the session, and a "don't remind me again" that mutes it permanently *for that ticker only*.

**Why:** a warning the user has consciously considered and rejected is noise, and noise trains people to ignore all warnings — including the next real one. Per-ticker muting respects the decision ("yes, I know half my portfolio is NVDA") without disabling protection for a *different* stock that concentrates later.

**Tradeoff:** two dismissal paths is more UI than one. The alternative — a global mute — was simpler and worse.

### 5. Three severities of message, and the discipline to use them

**Decision:** the app distinguishes a blocking error (red — something was prevented, e.g. a missing exchange rate), a caution (amber — the concentration warning), and an informational notice (blue — converted values are approximate). Non-blocking confirmations, like merging a new buy into an existing position, appear as a quiet toast that dismisses itself.

**Why:** severity is information. If everything is an alert, nothing is. The visual language teaches the user what deserves attention before they read a word.

**Tradeoff:** more CSS and more code paths than a single `alert()` box. But a browser alert for "you bought more AAPL" punishes normal usage.

### 6. Stateless, consistent colors

**Decision:** ticker colors are assigned by sorting all held tickers alphabetically and mapping position → palette index. The same function runs on every page.

**Why:** the same ticker must be the same color everywhere — donut, badges, transaction log — or the color stops carrying meaning. Doing it statelessly (recomputed from the ticker set, nothing saved) means the pages can never drift out of sync.

**Tradeoff:** adding or removing a holding can shift *other* holdings' colors, since alphabetical positions change. I accepted that flicker in exchange for zero persistence and zero cross-page sync bugs.

### 7. Accessibility that doesn't rely on color

**Decision:** every gain/loss shows a directional triangle (▲/▼) alongside the green/red coloring.

**Why:** red–green color blindness affects roughly 1 in 12 men. In a finance tool, up vs. down is the single most important distinction on the screen — it can't be conveyed by hue alone.

### 8. Manual prices, on purpose (for now)

**Decision:** current prices are entered by hand rather than fetched from an API.

**Why:** this is a static site with no backend, and a market-data API key embedded in client-side JavaScript is public the moment the page loads. The honest options are "manual entry" or "build a serverless layer," and shipping a working tool now beat shipping a security hole. The roadmap below is sequenced around crossing that wall properly.

**Tradeoff:** stale prices between updates, and the user does more typing. The data model was designed so that swapping in a live feed later changes *one input path*, not the architecture.

## How it's built

| Layer | Choice |
|---|---|
| Structure | Three static HTML pages: landing (`index.html`), tracker summary (`app.html`), transaction log (`transactions.html`) |
| Logic | Vanilla JavaScript, inline per page — no framework, no build step |
| Charts | Hand-drawn SVG (donut computed from stroke-dasharray arcs) |
| Storage | Browser `localStorage`, shared keys across pages |
| Design | Inspired by Toss: dark charcoal base, one blue-violet accent, Sora + Spline Sans typography, restrained motion |
| Hosting | GitHub Pages |

Run it locally with any static server:

```bash
# from the project folder
python3 -m http.server 8000
# then open http://localhost:8000
```

## Roadmap (deferred deliberately, not forgotten)

1. **Live price feed** — replace manual entry with real quotes. The gateway feature; ticker validation comes with it for free (invalid ticker → no price).
2. **Performance over time** — chart portfolio value against time, using the per-purchase dates the transaction log already records.
3. **Benchmark comparison** — overlay "what if the same money had gone into the S&P 500 instead," reframing the tool from a tracker into a judgment aid.
4. **Plain-language portfolio summary** — a brief read on composition and implied risk, built last because it's only as insightful as the real data beneath it.

All four sit behind the same wall: a static site can't safely hold an API key, so they require a small serverless layer first. The current version is intentionally complete and self-contained without them.

## A note on how this was built

This project was built with significant use of an AI coding assistant. The code was AI-assisted; the **product and architecture decisions were mine** — what to build, how to model the data, when a feature was worth its complexity, and when the simpler path was the wiser one. Several of the decisions documented above came from deliberately choosing *against* the first or more elaborate solution. I'm able to walk through the reasoning and tradeoffs behind any part of this project.

---

*Built by Ghun Jin.*
