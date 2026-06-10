# Memo: Does FRED retail sales predict Walmart revenue?

**To:** Portfolio Manager
**From:** Data Science
**Re:** Using FRED's monthly retail-sales series (RSXFS) as a leading indicator of Walmart quarterly revenue
**Date:** May 2026

---

## Question

You asked whether tracking FRED's monthly U.S. retail-sales index — series **RSXFS**, retail trade ex–food-services — gives us a usable lead on Walmart's quarterly revenue prints. Does it beat a naive forecast, by how much, and what should we worry about?

## Approach

We aligned RSXFS to Walmart's fiscal quarters (which end Jan / Apr / Jul / Oct, not calendar quarter-ends) and tested four models against three baselines on a strict expanding-window walk-forward back to FY2017 (37 evaluation quarters). To avoid look-ahead bias, the retail feature uses only the **first two months** of each Walmart fiscal quarter — the months that have actually been released to the public by the time the prediction is made.

## Result

**It depends on what you call "naive."**

| Comparison (lift in MAPE points; positive = challenger wins) | Median | 90% CI (block bootstrap) | Statistically real? |
|---|---:|---|:---:|
| SARIMAX_retail vs seasonal-naive | **+2.46 pp** | [+1.67, +3.28] | **yes** |
| SARIMAX_retail vs seasonal-naive + drift | +0.09 pp | [−0.12, +0.34] | no |
| **SARIMAX_retail vs AR(1) on Walmart's own past** | **−0.00 pp** | **[−0.20, +0.19]** | **no** |
| Walmart-only SARIMA vs AR(1) | −0.02 pp | [−0.27, +0.23] | no |

- Against the textbook seasonal-naive (`y_hat = y_{t−4}`), adding the retail signal gives a ~57% relative MAPE reduction — sounds great in isolation.
- Against a competent baseline that already knows Walmart grows ~4% YoY, the **retail signal adds nothing statistically distinguishable from zero**.

The bottom line: most of the apparent value of "retail predicts Walmart" is actually the value of *modeling the trend at all*. Once you control for that, RSXFS does not improve on a univariate Walmart-only model.

**Sub-period breakdown** explains why:

| Period | Best model | SARIMAX_retail vs AR(1) MAPE |
|---|---|---|
| Pre-COVID (FY2018–FY2020 Q1) | SARIMAX_retail (0.85% MAPE) | retail wins narrowly |
| COVID/stimulus (2020–2021) | SARIMA (1.56%) | **retail makes it worse** |
| Post-COVID (2022–present) | SARIMA (2.14%) | retail no help |

Pre-COVID, retail did real work. COVID broke the link — Walmart was a grocery/essentials winner while RSXFS surged on stimulus-driven discretionary spend. Post-COVID, inflation moved nominal retail dollars faster than Walmart's reported revenue, and the historical coefficient never re-anchored.

**Cross-correlation reading**: peak correlation is at lag 0, not at a positive lag. **RSXFS is coincident, not leading.** The useful framing is "nowcast a quarter in progress," not "forecast a future quarter."

## Recommendation

1. **Don't pay extra for RSXFS as a Walmart signal.** It works in calm regimes but doesn't add over a simple AR(1) on Walmart's own history. A team already running standard time-series models gets no incremental edge from this series alone.
2. **Don't trust it during macro shocks.** Whenever inflation > 5% YoY or there is large-scale fiscal transfer, drop back to a univariate model. Past behavior says retail *increases* error during those windows, not reduces it.
3. **Treat the headline "57% MAPE reduction vs seasonal-naive" as a strawman.** Anyone selling this externally that way is comparing against the wrong baseline.

## Risks (what would change my mind)

- **Mix mismatch.** RSXFS is total U.S. retail ex–food-services. Walmart is ~56% grocery, ~17% international, and has Sam's Club. A subsector-matched FRED series (`MRTSSM4451USN` food-and-beverage stores, `MRTSSM452USN` general-merchandise stores) — or a *weighted basket* of them matching Walmart's mix — would probably win where RSXFS doesn't.
- **Segment-level revenue.** Walmart files Walmart-U.S. and Sam's Club segment revenue separately. Modeling those directly (instead of consolidated top-line, which is contaminated by FX from the international segment) would isolate the retail signal more cleanly.
- **Higher-frequency private data.** This whole exercise uses public monthly data. YipitData's card-panel or web-scrape signals operate at *daily* frequency and capture Walmart-specific traffic — that's where the real edge would live. RSXFS is the public-data floor, and our finding is that the floor isn't very high.
- **Sample size.** 37 walk-forward quarters; the 90% CI on the SARIMAX-vs-AR(1) lift is ±0.2 MAPE points. Could become significant with more data — but the direction of the point estimate suggests we shouldn't expect it to.

## Next steps

1. Re-run with `MRTSSM4451USN` and `MRTSSM452USN` (single-line code change) and report.
2. Pull Walmart segment revenue from 10-Q filings and re-test on Walmart-U.S. only.
3. If we have a Walmart-specific YipitData panel, compare its lift vs FRED head-to-head.

---

*Technical detail and walk-forward code: `analysis.ipynb`. LLM prompt log: `prompts.md`.*
