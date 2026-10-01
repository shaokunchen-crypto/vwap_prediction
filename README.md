# VWAP / Order-Book Workshop

## Overview

This project studies whether progressively richer order-book VWAP
constructions can improve prediction of the **next 5-second midpoint**.

The supplied HTML contains:

-   20 order-book snapshots
-   3 displayed levels per side
-   5-second spacing
-   A one-step-ahead target
-   Three VWAP constructions
-   Quote-age decay experiments
-   A lagged trade-flow extension
-   Train / holdout model selection
-   An interactive order-book explorer
-   A directional trading-rule replay and P&L calculation

The analysis is deliberately time ordered: parameters are selected on
**T01--T12** and then frozen for the **T13--T20 holdout**.

------------------------------------------------------------------------

## 1. Prediction Target

At snapshot (T_t), the model uses information available at that snapshot
only.

The target is the midpoint at the next snapshot:

\[ Target_t = Mid\_{t+1} = `\frac{Bid1_{t+1}+Ask1_{t+1}}{2}`{=tex} \]

There are 19 forecastable transitions from the 20 snapshots.

The HTML uses **MAE (Mean Absolute Error)** as the primary measure of
predictiveness, with RMSE and signed error/bias as secondary checks.

------------------------------------------------------------------------

# 2. VWAP Models

## VWAP 1 --- Direct Crossed Quantity Calculation

The first construction directly pairs quantities on one side with prices
on the opposite side:

\[ VWAP1 = `\frac{
\sum_i(q_i^a p_i^b + q_i^b p_i^a)
}{
\sum_i(q_i^a+q_i^b)
}`{=tex} \]

where:

-   (q_i\^b) = bid quantity at level (i)
-   (p_i\^b) = bid price at level (i)
-   (q_i\^a) = ask quantity at level (i)
-   (p_i\^a) = ask price at level (i)

This is the direct crossed-quantity construction shown in the workshop.

------------------------------------------------------------------------

## VWAP 2 --- Normal Side-Specific Book VWAP

VWAP 2 first calculates a weighted VWAP separately for the bid and ask
books.

The level weights are:

\[ w_i=\[1,`\lambda`{=tex},`\lambda`{=tex}\^2\] \]

with:

\[ `\lambda=0.25`{=tex} \]

so the weights are:

\[ \[1,;0.25,;0.0625\] \]

### Weighted bid VWAP

\[ B = `\frac{
\sum_i w_i q_i^b p_i^b
}{
\sum_i w_i q_i^b
}`{=tex} \]

### Weighted ask VWAP

\[ A = `\frac{
\sum_i w_i q_i^a p_i^a
}{
\sum_i w_i q_i^a
}`{=tex} \]

Define weighted displayed depth:

\[ V_b=`\sum`{=tex}\_i w_iq_i\^b \]

\[ V_a=`\sum`{=tex}\_i w_iq_i\^a \]

The final theo is:

\[ Theo = `\frac{
B V_a + A V_b
}{
V_a+V_b
}`{=tex} \]

This is the normal side-specific book VWAP construction used by the
workshop.

------------------------------------------------------------------------

# 3. VWAP 3 --- Quote-Age Decay

VWAP 3 keeps the VWAP 2 level structure but also makes the contribution
of each quote depend on its age.

The general form is:

\[ w_i\^t=\[1,w_1,w_2\]\_i `\times `{=tex}f(age_i\^t;`\tau`{=tex}) \]

The selected implementation uses **exponential decay**:

\[ f(age;`\tau`{=tex})=e\^{-age/`\tau`{=tex}} \]

with:

\[ `\tau=45`{=tex}`\text{ seconds}`{=tex} \]

Thus:

\[ `\tilde `{=tex}w\_{i,b,t} = \[1,0.25,0.0625\]*i e\^{-BidAge*{i,t}/45}
\]

and:

\[ `\tilde `{=tex}w\_{i,a,t} = \[1,0.25,0.0625\]*i e\^{-AskAge*{i,t}/45}
\]

These time-adjusted weights are then used in the same side-specific VWAP
construction as VWAP 2.

The purpose is to separate:

1.  **Level importance** --- deeper levels receive lower base weights.
2.  **Quote freshness** --- older quotes receive lower effective
    weights.

------------------------------------------------------------------------

# 4. Decay Model Selection

The HTML compares several quote-age decay families.

  Decay family      Train MAE   Holdout MAE   Best τ
  --------------- ----------- ------------- --------
  Exponential        £0.02750      £0.05437      45s
  Reciprocal         £0.02753      £0.05494      20s
  Gaussian           £0.02751      £0.05266      45s
  Linear cutoff      £0.02891      £0.05796      90s

The selected VWAP 3 specification in the workshop is the **exponential
decay with τ = 45 seconds**.

The HTML emphasizes that the sample is small, so very small MAE
differences can be sampling noise.

------------------------------------------------------------------------

# 5. Model Parameters

The workshop exposes the following parameters:

  Parameter   Meaning                         Current value
  ----------- ----------------------------- ---------------
  λ           VWAP 2 depth decay                       0.25
  (w_1)       Independent level-2 weight               0.25
  (w_2)       Independent level-3 weight             0.0625
  τ           Quote-age time scale                      45s
  γ           Flow adjustment coefficient            -0.100

The HTML recommends using a small, interpretable parameter grid and
time-ordered validation.

------------------------------------------------------------------------

# 6. Trade-Flow Extension

The workshop then adds the **last completed trade flow**.

For forecasting (T_t `\rightarrow `{=tex}T\_{t+1}), only trades printed
by (T_t) are used.

Flow is defined as signed aggressive quantity divided by total traded
quantity:

\[ flow_t= `\frac{\sum signed\_qty}`{=tex} {`\sum `{=tex}qty} \]

The adjusted theo is:

\[ Theo\^{flow}\_t = VWAP3_t + `\gamma `{=tex}`\cdot `{=tex}flow_t
`\cdot `{=tex}spread_t \]

The selected coefficient is:

\[ `\gamma=-0.100`{=tex} \]

The HTML notes that the negative coefficient means that, in this replay,
the fitted flow relationship moves the theo opposite to recent signed
flow. It explicitly treats this as a property of the sample rather than
a general market rule.

------------------------------------------------------------------------

# 7. VWAP 3 + Flow Results

The workshop reports:

  Model             Train MAE   Holdout MAE   Full MAE
  --------------- ----------- ------------- ----------
  VWAP 2             £0.03364      £0.05533   £0.04163
  VWAP 3             £0.02750      £0.05437   £0.03740
  VWAP 3 + flow      £0.02697      £0.05079   £0.03575

The HTML's stated result is that the lagged-flow feature reduces:

-   Holdout MAE from **£0.05437 → £0.05079**
-   Full-sample MAE from **£0.03740 → £0.03575**

The workshop also highlights the small-sample caveat.

------------------------------------------------------------------------

# 8. Train / Holdout Design

The data are split chronologically.

### Training / parameter-selection block

Forecasts associated with:

\[ T01`\rightarrow `{=tex}T02
`\quad`{=tex}`\text{through}`{=tex}`\quad`{=tex}
T12`\rightarrow `{=tex}T13 \]

Parameters are selected using this block.

### Holdout block

The selected parameters are then frozen and evaluated on:

\[ T13`\rightarrow `{=tex}T14
`\quad`{=tex}`\text{through}`{=tex}`\quad`{=tex}
T19`\rightarrow `{=tex}T20 \]

No future book or trade information is used for the one-step forecasts.

The purpose is to avoid choosing parameters using the same observations
used to evaluate them.

------------------------------------------------------------------------

# 9. Trading Rule

The HTML converts the VWAP 3 + flow theo into a directional trading
signal.

First define:

\[ edge = Theo\^{flow}-Mid \]

The threshold is:

\[ threshold=0.12`\times `{=tex}Spread \]

The rule in the supplied HTML is:

``` text
BUY  if edge > +threshold
SELL if edge < -threshold
FLAT otherwise
```

In equivalent form:

\[ BUY `\quad`{=tex}`\text{if}`{=tex}`\quad`{=tex}
Theo\^{flow}\>Mid+0.12`\times `{=tex}Spread \]

\[ SELL `\quad`{=tex}`\text{if}`{=tex}`\quad`{=tex}
Theo\^{flow}\<Mid-0.12`\times `{=tex}Spread \]

Otherwise the strategy is flat.

The model therefore only trades when the estimated fair value is at
least 12% of one displayed spread away from the midpoint.

------------------------------------------------------------------------

# 10. Execution Assumptions

The supplied HTML uses:

-   Position size: **50 units**
-   BUY execution: **best Ask**
-   SELL execution: **best Bid**
-   Signal change: **close/reverse**
-   Final open position: marked at the **T20 midpoint**
-   Trading cost: **£0.20 per fill**

A signal change first closes the existing position and, when the new
signal is non-flat, opens the new position in the opposite direction.

------------------------------------------------------------------------

# 11. P&L Replay --- Corrected VWAP 3 + Flow

The main P&L replay uses:

-   Normal side-specific VWAP
-   Level weights `[1, 0.25, 0.0625]`
-   Exponential quote-age decay
-   (`\tau`{=tex}=45s)
-   Flow coefficient (`\gamma=-0.100`{=tex})
-   12% of spread trading threshold
-   50-unit position size
-   Best bid / best ask execution
-   £0.20 per fill

The HTML reports:

  Metric            Result
  ----------- ------------
  Gross P&L         £15.00
  Fills                 12
  Fees               £2.40
  Net P&L       **£12.60**

Calculation:

\[ Gross P&L=£15.00 \]

\[ Trading costs=12`\times`{=tex}£0.20=£2.40 \]

\[ Net P&L=£15.00-£2.40=£12.60 \]

The HTML explicitly notes that this result uses the **normal
side-specific book VWAP** and not the earlier incorrect cross-book
pairing formula.

------------------------------------------------------------------------

# 12. Interactive Order-Book Explorer

The HTML contains an interactive explorer for all 20 snapshots.

For each snapshot it displays:

-   Three bid levels
-   Three ask levels
-   Price
-   Quantity
-   Quote age
-   Midpoint
-   Model theo values

The central column places the midpoint and theo values at their
corresponding prices.

The snapshot selector allows the user to move through:

\[ T01,`\ldots`{=tex},T20 \]

------------------------------------------------------------------------

# 13. Example Trade-Flow Snapshot

The workshop highlights T18:

-   Net aggressive quantity: **-160**
-   Total traded quantity: **160**
-   Flow fraction: **-1.00**
-   T18 → T19 midpoint: **£8.475**

The flow fraction of -1.00 means all displayed aggressive trade quantity
in that interval was selling in the supplied replay.

------------------------------------------------------------------------

# 14. Workshop Questions

The HTML addresses several practical questions.

## How do you measure predictiveness?

At every snapshot:

1.  Calculate the theo using information available at that time.
2.  Compare it with the next midpoint.
3.  Use MAE as the primary metric.
4.  Use RMSE and signed error/bias as secondary diagnostics.
5.  Compare against simple baselines.

## What could reduce effectiveness?

The workshop identifies:

-   Rapid cancellations
-   Spoofing-like displayed liquidity
-   Hidden liquidity
-   Sudden volatility changes
-   Widening or collapsing spreads
-   Directional regime shifts
-   Aggressive trade sequences that move the market faster than the book
    statistic updates

## How can the VWAP be made more robust?

The HTML suggests:

-   Quote-age decay
-   Capping extreme flow adjustments
-   Monitoring MAE by market regime
-   Rolling-window re-estimation
-   Comparison with simpler benchmarks
-   Treating trade flow as an optional feature that must demonstrate
    out-of-sample improvement

## What parameters can be introduced?

The workshop suggests:

-   λ, or independent (w_1,w_2)
-   Quote-age decay scale τ
-   Flow coefficient γ

It recommends keeping parameter grids small and using walk-forward
validation.

------------------------------------------------------------------------

# 15. Key Takeaways From the Supplied HTML

1.  **VWAP 2** is the normal side-specific book VWAP with level weights
    `[1, 0.25, 0.0625]`.
2.  **VWAP 3** adds quote-age decay to those level weights.
3.  The selected decay specification is **exponential, τ = 45s**.
4.  The flow extension uses the immediately preceding completed
    interval.
5.  The selected flow coefficient is **γ = -0.100**.
6.  On this replay, VWAP 3 + flow has full MAE **£0.03575**.
7.  Its holdout MAE is **£0.05079**.
8.  The trading replay uses a **12% of spread threshold**.
9.  The main corrected replay reports **£12.60 net P&L** after fees.
10. The HTML treats all results as descriptive of this replay and
    emphasizes the limited sample size.

------------------------------------------------------------------------

# 16. Reproducibility / Caveats

This is a small 20-snapshot replay. The HTML explicitly separates
parameter selection from holdout evaluation, but the number of
observations remains limited.

The reported results should therefore be interpreted as **results of the
supplied replay**, not as evidence that the methodology will generalize
to a larger live market.

For further testing, the natural next steps are:

-   More snapshots
-   More independent trading periods
-   Rolling or walk-forward validation
-   Transaction-cost sensitivity
-   Threshold sensitivity
-   Position-size sensitivity
-   Spread-regime analysis
-   Volatility-regime analysis
-   Comparison against simpler fair-value baselines
-   Out-of-sample testing on entirely new data

------------------------------------------------------------------------

