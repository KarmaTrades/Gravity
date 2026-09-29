---
type: handoff
project: "[[Convex]]"
target: terminal
created: 2026-09-28
vault_path: Projects/Convex/Handoffs
status: in-progress
tags: [handoff, convex]
---

# Handoff: Convex → Claude Code (terminal)

Save this file in the repo root as `HANDOFF.md`. Add it to `.gitignore` if you prefer; it holds no secrets, but it does hold account context.

Follow plan mode: propose a numbered plan and wait for approval before editing.

## Mission

Carry the `convex-trade` portfolio bot through the two-week paper test. Help Wero make the arm-or-cancel decision before the Combine's next rebill. If he approves, research whether splitting the portfolio's legs across 2–3 Topstep accounts adds real breadth.

## Goals and definition of done

| # | Goal | Done when | Status |
|---|---|---|---|
| 1 | Two-week paper test of the portfolio bot (Tue–Fri, 8 sessions) | Two files exist: `docs/paper-journal.md` with 8 session entries, and `docs/paper-report.md` with PASS/FAIL and evidence for the 5 pre-registered criteria in `docs/PAPER-TEST.md` §5 | unknown (unverified): the protocol is dated 2026-09-17 and the journal template's first session is 2026-09-22. Check `docs/paper-journal.md` and `~/.convex/logs/` |
| 2 | Arm-or-cancel decision for weeks 3–4 | Wero has decided either (a) arm with `--override-gate`, or (b) cancel the Combine subscription before its 30-day rebill (≈ 2026-10-13, per the findings log) | blocked: this is Wero's decision; it needs goal 1's mechanics verdict |
| 3 | Multi-account breadth research: different legs on different accounts, each with its own $1,000 DLL | A lab script prices split-leg configurations on the seven-year store through the Topstep model. It includes a hard guard against opposite MNQ positions across accounts. The results are written to `results/` | not started: only proposed in chat, needs plan approval |
| 4 | Expected-returns one-pager in the vault | Note filed under `Projects/Convex/` | not started: text exists only in the claude.ai chat (see "Where things stand") |

## Where things stand

The research phase is finished. Across ~160 rule and geometry combinations, 3,402 ranked candidates, 8 other prop firms and an LLM-overseer test, nothing clears the robust gate. Topstep's structure needs an annualised Sharpe of about 2 to pay for itself (Tier A) and about 4 to be robust (Tier B). Every price-only effect measured sits at 0.7–1.4.

**What is built.** The bot runs `strategy portfolio` with `profile mnq_portfolio`. That is three legs, one position at a time, Tuesday to Friday:
- orbc: 50-pt stop, time-stopped at 13:55.
- tod14: 25-pt stop, 1:1, time-stopped at 14:49.
- im1450: 25-pt stop, held to the 15:05 flatten.

Size is 3 micros in the Combine and 2 in the XFA. Its arming gate says `passed: false` (tier "−"; P2, P4, P5 fail), so arming requires `--override-gate`.

**Expected returns (this chat, from `results/stress_portfolio.json` and the paper protocol §0):**

| Scenario | EV/month | P(year > 0) | Combines/yr |
|---|---|---|---|
| Base, 3/2 micros | $168 (≈ $155 after the $14.50 API fee) | 48% | 2.0 |
| Tuned 5/3 micros (RTP flag) | $242 | 54% | 3.6 |
| +1 tick slippage per side | $165 | 47% | 4.2 |
| 2024–2026 history only | $155 | 43% | — |
| Stress (−0.05R, +1 tick) | $11 | — | — |

- The two-week paper expectation is ≈ +$365 with ±$1,600 of noise.
- The earlier "Tier A, $291/month, 62%" figure is withdrawn (unit-clipping artefact).

**Multi-account (this chat).** Copying the bot across k accounts scales EV about ×k: ≈ $825/month at 5 accounts. It leaves P(year > 0) at ≈ 48%, because the paths are identical. It is also flagged:
- Loss-limit hits on several accounts in one day are a Responsible Trading trigger.
- Mass abandon-and-rebuy looks like stacking.
- A Live call-up closes all XFAs.

These numbers are linear extrapolations, not new simulations (unverified). The one untested idea with potential is to put different legs on different accounts, which gets past the per-account DLL wall that sank the five-mechanism portfolio.

## What went right (keep doing)

- **Pre-registered gates** (n ≥ 350, lb95 > 0, ≥ 5/7 years positive, holdout ≥ 2026-07-16 not negative) fixed before looking. They kept the project honest through ~20 hypotheses.
- **Red-team passes.** Every one found a real defect, and every fix made the verdict stricter.
- **`eval/fastlab.py` precompute.** A rule becomes a signal array scored on 2,295 sessions in ~30 ms, trade-for-trade identical to `convex.live.backtest` (`tests/test_fastlab.py`).
- **Seven years of Databento `MNQ.v.0` history.** It killed the one-year VWAP "edge"; always evaluate on the full store first.
- **Fresh-seed acceptance in the learning loop.** It stopped coin-flip "passes" at 3,000 paths.

## What went wrong (do not repeat)

- **Tried:** `convex-trade data pull --days 365` for a year of bars.
  **Result:** `57668 bars over 52 sessions (2026-07-16 .. 2026-09-14), 0 failed requests`. The gateway serves only ~60 days of minute bars; older contracts come back empty, not as errors.
  **Outcome:** fixed. The store merges weekly pulls, and history comes from `scripts/databento_pull.py`.
- **Tried:** judging the VWAP fade on one year of data (+0.06–0.10R).
  **Result:** over seven years it scored −0.038R (4,267 OOS trades, 1 of 7 years positive).
  **Outcome:** abandoned. One year cannot separate +0.06R from zero.
- **Tried:** `families.Day` defining "previous session" as the previous CT calendar day.
  **Result:** `prev_range` was NaN on Mondays, so 0 of 376 Mondays traded in every family.
  **Outcome:** fixed (previous RTH session, forward-filled). The Tue–Fri rule is now explicit and flagged as post-hoc.
- **Tried:** the `combo` stream unit of $200 while the ORB leg had 2 attempts (worst day $300).
  **Result:** 199 of 1,512 days had their losses clipped, producing a fake "Tier A $291/mo, 62%".
  **Outcome:** fixed in `learn.measure` (unit = true worst day); the claim is withdrawn.
- **Tried:** the overnight drift on the continuous `MNQ.v.0` series.
  **Result:** 16 roll nights carried a ~+110 bps calendar-spread jump (switch at 00:00 UTC).
  **Outcome:** roll nights are excluded. The drift fails post-2021 anyway.
- **Tried:** the P5 edge haircut as −0.05R added to outcomes.
  **Result:** 63% of it was absorbed by the stop clip.
  **Outcome:** fixed; the haircut is now charged as friction (d_edge × stop dollars).
- **Tried:** an LLM overseer (month-by-month learning, and blind prior-informed rules).
  **Result:** blind rules scored ΔSharpe −0.07 out of sample (P(Δ > 0) = 0.42); rules learned on one synthetic year contradicted the other.
  **Outcome:** abandoned as a live input. It is kept only as the forward shadow log (`convex-trade overseer record`), gated at ≥ 500 sessions.

## Decisions log

- **Portfolio, one slot, time-stopped ORB** over the concurrent version: the runner holds one bracket at a time, and freeing the slot for the afternoon legs beats holding the ORB. The selection among 8 variants was post hoc.
- **Tue–Fri only**: Monday is −$29/day vs Tue–Fri (t = −2.7). It is post-hoc, but the only calendar rule with consistent sign.
- **Topstep over other venues**: no venue changes the verdict (±$100/month). Topstep is also the only firm with a direct API key.
- **No overseer in the sizing path**: it stays a shadow log until its pre-registered gate passes.
- **Multi-account copier not recommended**: it adds size without diversification and trips RTP triggers.

## Constraints

- **Never read, print or log the API key.** It is entered only by Wero through `convex-trade setup`'s hidden prompt. Don't touch `~/.convex/credentials`, the clipboard, or environment variables that might hold it.
- **Paper phase: never run `run --arm` or `--override-gate`.** Arming is Wero's decision and his keystroke.
- **Wero's Mac only.** No VPS, VPN or remote host (Topstep rule). Don't bypass `looks_like_remote_host`.
- **Minimal diff.** Don't change strategy logic, the guard, the executor or the tests during the paper test. Report anomalies in the journal instead.
- **Never delete anything under `~/.convex/`.** Never `git push`.
- **Don't interpret two weeks of P&L** in either direction.
- **Any multi-account work** must enforce "never opposite MNQ positions across accounts at the same time". Cross-account hedging is terminal and non-appealable at Topstep.
- **Red-team pass required** for any trading-logic change (Wero's standing rule for trading logic).

## Artifacts

- `~/Documents/convex` — repo; branch `master` per the bundle-pull command. Last commit unknown (unverified): run `git log -1` and `git status`.
- `docs/README-live.md` — runbook ([[Convex live — runbook]]).
- `docs/convex-paper-protocol.md` — protocol and numbers ([[Convex paper protocol]]).
- `docs/PAPER-TEST.md` — operator checklist and acceptance criteria.
- `docs/paper-journal.md`, `docs/paper-report.md` — goal 1 outputs; may or may not exist yet.
- `results/learn/measure_44dcdd08.json`, `results/learn/ledger_topstep_seq_seq_orb1_1355_tod_1449.json` — gate inputs.
- `results/stress_portfolio.json` — stress test.
- `eval/seq_portfolio.py`, `eval/stress_portfolio.py`, `eval/learn.py`, `eval/rank_all.py`, `eval/fastlab.py`, `eval/families.py` — the lab.
- `convex/venues.py`, `convex/live/overseer.py` — venue specs and overseer gate.
- `~/.convex/` — config, state, `data/MNQ_1m.csv`, `logs/trades_paper_<account>.csv`, `logs/convex-trade.log`, `gate_portfolio_mnq_portfolio.json`, `overseer_shadow.csv`.
- Env var names:
  - `TOPSTEPX_USERNAME`, `TOPSTEPX_API_KEY` — both `<TOPSTEPX_API_KEY>`; never read.
  - `DATABENTO_API_KEY` — `<DATABENTO_API_KEY>`; per-terminal only, never on disk.

## Commands

None were run in this session; it was a claude.ai chat without repo access. The commands below come from the runbook and protocol, and all are unverified here.

```bash
cd ~/Documents/convex && source .venv/bin/activate
python -m pytest -q                                   # expect green (64-65 tests; one needs the seven-year CSV)
TZ=America/Chicago date                               # cash session 08:30-15:05 CT = 07:30-14:05 Monterrey until 2026-11-01
convex-trade status
convex-trade run --paper                              # by 08:20 CT, Tue-Fri, until "flat for the day"
convex-trade data pull --days 3 && convex-trade reconcile   # after 15:10 CT
convex-trade overseer brief >> ~/.convex/briefs.log   # optional shadow input
```

## Open questions for Wero

- Has the paper test started, and on which date? How many sessions are logged?
- Arm for weeks 3–4 (≈ $155/month expected, ≈ 50% chance of a losing year), or cancel the Combine before ≈ 2026-10-13?
- Approve the split-legs multi-account research (goal 3)?
- Where should the one-pager be filed? Suggested: `Projects/Convex/Convex expected returns.md`.

## Next steps

1. Confirm state: `git status`, `git log -1`, check whether `docs/paper-journal.md` exists and how many sessions it holds, and `ls ~/.convex/logs/`. Report back before doing anything else.
2. If the paper test is running, keep to `docs/PAPER-TEST.md` §4 each Tue–Fri. If it is complete, write `docs/paper-report.md` against the five criteria (mechanics only).
3. Put the arm-or-cancel decision in front of Wero with the mechanics verdict and the §0 numbers. Remind him of the ≈ 2026-10-13 rebill date.
4. If goal 3 is approved: write a read-only plan for `eval/multi_account.py`. It should cover:
   - one leg per account;
   - a guard that rejects any configuration where two accounts could hold opposite MNQ positions at once;
   - pricing through `longterm.evaluate` with the same P1–P8 gate;
   - a red-team pass.

   Wait for approval before writing code.

## Related

- [[Convex findings]] · [[Convex venues]] · [[Convex paper protocol]] · [[Convex live — runbook]] · [[Topstep $50K rules reference (Convex)]] · [[convex-venue-api-survey]]
