# LLM Prompt Log

**Assistant used:** Claude (Opus 4.7, 1M context) via Claude Code.
**Total assistant-hands-on time:** ~3 hours over a few sittings.

---

## Note on how I used it (< 200 words)

I used Claude as a senior pair-programmer — for scaffolding the notebook structure, drafting boilerplate (SARIMA grid search, walk-forward harness, diagnostics helpers), and as a sparring partner on framing. Two places I had to push back:

1. **Calendar alignment.** Claude's first draft used `resample("QE")` on RSXFS, which aligns to Mar/Jun/Sep/Dec — *not* Walmart's fiscal Jan/Apr/Jul/Oct. I caught this when I noticed the merge was dropping rows. Fixed by writing `retail_to_walmart_quarter()` that explicitly maps months M-2, M-1, M to Walmart's quarter ending in month M. The brief actually flagged this exact pitfall.
2. **Look-ahead bias on the "full quarter" feature.** Claude's first SARIMAX used all 3 months of retail as the quarter feature, which leaks the last month (released ~3 weeks after Walmart's quarter-end, i.e. roughly when Walmart reports). I switched the primary feature to `retail_q_first2_mean` and noted both regimes (full-quarter vs first-2-months) in the EDA.

Where Claude was helpful: scaffolding the walk-forward loop correctly (expanding window, fitting inside the loop, not outside — a common mistake), suggesting block-bootstrap for the MAPE-lift CI (right answer for non-iid time-series residuals), and pushing back when I asked for an LSTM ("sample size doesn't justify it").

I verified by reading every code cell line-by-line and re-deriving the calendar mapping by hand for three example dates.

---

## Prompt log (chronological)

### Prompt 1 — Kickoff

> I am solving a take-home assignment for YipitData. The customer asks: "We track the monthly retail-sales series from FRED for our subsector. We are thinking about using it as a leading indicator of quarterly revenue for Walmart. Does it predict Walmart's revenue better than a naive baseline? If yes, by how much? What should we worry about? If not, what evidence would change our minds?"
>
> Files: retail_sales_fred.csv (monthly), walmart_revenue.csv (quarterly).
>
> Act as a Senior Data Scientist. Do this in eight phases (EDA → feature engineering → baselines → model selection → improvement → walk-forward eval → business insights → memo). Don't jump to modeling. Challenge your own assumptions. Flag leakage. Be intellectually honest, not blind coding.

(The full kickoff prompt is preserved verbatim in the conversation history; this is the abridged version.)

### Prompt 2 — Calendar pushback

> Wait — Walmart fiscal Q1 ends April 30, not March 31. If you `resample("QE")` on the monthly retail data, you get calendar quarters ending Mar/Jun/Sep/Dec, which don't match Walmart's fiscal calendar. Show me how you're aligning the two.

*(This was the corrective prompt that caught the calendar bug.)*

### Prompt 3 — Leakage check

> For the SARIMAX, what retail months are you feeding in to predict Walmart Q1 (ending Apr 30)? If you're using April retail, that value isn't published until mid-May, which is the same time Walmart reports — so the model has the answer key. Either use only Feb/Mar retail, or be explicit that you're doing post-hoc attribution, not nowcasting.

### Prompt 4 — Model menu

> I want quality over quantity. Pick 2–3 models. SARIMA + SARIMAX + a distributed-lag OLS feels right. No LSTM — sample size is 64 quarters. Justify the choice.

### Prompt 5 — Robustness

> What's the CI on the MAPE-lift? Quarterly time-series residuals aren't iid so a regular bootstrap is wrong. Block bootstrap with block-size 4 (one year) is the move. Add it.

### Prompt 5b — The pivot

> The first cut shows SARIMAX_retail beats seasonal-naive by +2.46 pp MAPE with CI [+1.67, +3.28] — looks like a clear win. But seasonal-naive is a strawman because it ignores trend. Re-run the bootstrap with seasonal-naive-plus-drift AND with AR(1)-on-Walmart as the baselines. If retail doesn't beat AR(1), that's the real story.

*(This pushback produced the headline finding. SARIMAX_retail vs AR(1): median lift −0.00 pp, CI [−0.20, +0.19]. The signal does not add over a univariate Walmart-only model. The original framing — "retail beats naive baseline" — was technically true but materially misleading.)*

### Prompt 6 — Sub-period split

> Split metrics into pre-COVID / COVID / post-COVID. If the model wins overall but loses in COVID, that's the most important finding for the buy-side reader. Make sure the table shows it.

### Prompt 7 — Memo tone

> The memo is for a PM who took stats in college but hasn't done much since. No jargon-heavy paragraphs. Lead with the answer ("yes, but modest and regime-dependent"). Make the "what should we worry about" section the longest one — that's what they actually want to know.

---

## What the assistant got wrong, what it got right

**Got wrong (and I caught):**
- Calendar alignment using `resample("QE")` (caught by reading `merged` after the join — rows were silently missing).
- Initial SARIMAX used all 3 months of retail → leakage; switched to first-2-months-mean as the no-leakage feature.
- Walk-forward harness bug: `predict_sarimax` was forecasting with `exog.iloc[[-1]]` (the *previous* quarter's retail), not the target quarter's retail. Same bug in `predict_dlm_ols`. Both ran without error and produced plausible numbers — exactly the kind of silent bug that ruins a submission. Caught when I traced through one quarter by hand and the printed prediction didn't match what should have come out for the published exog row.
- Initial framing reported "retail beats naive baseline by +57% MAPE" — technically true, but the baseline was seasonal-naive (no trend), which is a strawman. Pushed back and added bootstrap CIs against SN+drift and AR(1) — that's where the actual answer lives (the signal doesn't beat them).
- First-pass plots had every series colored differently with no consistent palette — cluttered. Asked it to standardize on three colors (retail blue, WMT red, baselines gray, best-model green).

**Got right (with light steering):**
- Walk-forward harness with expanding window and refit-inside-loop.
- Block bootstrap for MAPE-lift CI (suggested when I asked "how do I put error bars on the MAPE difference?").
- Clear separation of EDA into "are they clean / are they comparable / are they predictive" — that framing came from the LLM and I kept it.
- Pushing back on me when I floated "what about a tiny LSTM?" — correctly pointed out 64 quarterly observations is far too few.

**How I verified:**
- Re-derived the fiscal-calendar mapping by hand for three example dates (2010-04-30, 2020-01-31, 2024-04-30) and confirmed it picked months Feb/Mar/Apr, Nov/Dec/Jan, Feb/Mar/Apr respectively.
- Ran the notebook end-to-end on a clean venv to confirm it executes without errors.
- Spot-checked the seasonal-naive baseline against a hand calculation on two quarters.
- Read every model-fitting block to make sure no training data leaked from inside the walk-forward loop.
