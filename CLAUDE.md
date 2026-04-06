# CLAUDE.md — FIN 452 Quantitative Trading Model (QTP)

## SECTION 1: PROJECT OVERVIEW

**Assignment:** FIN 452 Quantitative Trading Model — Individual, due 2026-04-07 at 6pm EDT
**Deliverable:** R Markdown (.qmd + .html, embed-resources: true), 10-minute presentation
**Universe:** SPY + 11 Select Sector SPDR ETFs (XLB, XLC, XLE, XLF, XLI, XLK, XLP, XLRE, XLV, XLU, XLY)
**Training period:** June 2000 – Dec 2024
**Testing period:** Jan 2025 – March 2026
**Frequency:** Daily data, monthly rebalancing
**Data:** All via API (tidyquant::tq_get). No local files.
**Requirement:** Clustering as primary signal driver + Kalman Filter integration
**GitHub:** Public repo, daily commits showing work progression (5-pt penalty for missing)

---

## SECTION 2: RUBRIC PRIORITIES (from grading sheet, /20)

### Intuitive Rationale (3 pts)
- Clarity of exposure: what macro view does this strategy express?
- Specific references to academic papers (cite both SSRN papers)
- When it works AND when it does not — be explicit about failure conditions

### Model Implementation (16 pts)
**Hard penalties to avoid:**
- Look-ahead / survivorship bias
- Optimization not implemented (-2 if absent)
- Trades on same day — signal at Close(t), execute at Open(t+1), never same-day
- Only cumulative returns / single performance criteria (-1)
- Strategy function and optimization not implemented (-10%)

**Must haves (each individually graded):**
- How signals are generated clearly documented (1 pt)
- Rules for trade generation / execution (1 pt)
- Risk appetite clearly articulated with reasoning (2 pts)
- Training period results (0.5 pt)
- Testing period results (0.5 pt)
- Multiple performance criteria: Sharpe, Omega, Sortino, max drawdown, Calmar — NOT just cumulative return

### Presentation (penalties only: -3 to -1)
- [-3] Lack of clarity of flow including charts vs text (intersperse charts WITH commentary)
- [-1 to -2] Unnecessary messages, clutter, raw df output, squished charts
- No leftover code artifacts, no message=TRUE, no warning=TRUE in final render

### Learnings (1 pt — Prof weights this most heavily)
- What worked, what failed, what you would do differently
- Must reference FIN 440 QTP failures and how this design explicitly addresses them
- Minimum 5 substantive points, 3-5 sentences each

---

## SECTION 3: HIGH-LEVEL ARCHITECTURE (THREE-LAYER DESIGN)

### Layer 1: Asset Clustering (static, fit on training period only)
**Purpose:** Which ETFs are structurally similar?

Features per ETF computed over full training window:
- Mean annualized return, volatility (std dev)
- Skewness, kurtosis
- Beta to SPY
- Sharpe ratio, Sortino ratio
- Upside capture / downside capture vs SPY
- Omega Ratio (threshold = 0)
- Maximum Drawdown
- 5% VaR, 5% CVaR (Expected Shortfall)
- Average pairwise correlation to other 10 ETFs

Dimensionality: ~14 features, 11 ETFs. PCA is required before clustering (underdetermined otherwise).
- Normalize first: scale() all features
- PCA: report variance explained per PC, scree plot, biplot with ETF labels
- Label PCs economically (PC1 likely "return quality", PC2 likely "tail risk / drawdown")
- Cluster on top 2-3 PCs using both K-Means and Hierarchical — compare results
- Validate: silhouette width (must be > 0.3), within-cluster sum of squares elbow plot
- Output: ETF buckets with commercial labels (e.g., defensive, cyclical, commodity-linked, rate-sensitive)
- Run ONCE on training data only. Apply labels forward to testing period.

### Layer 2: Regime Clustering (rolling, primary signal driver)
**Purpose:** What macro environment are we in right now?

Features per rolling time window (3-month rolling window, 1-month lag per Raju 2025):
- SPY return (market direction)
- SPY realized vol: rolling_sd(log(P_t / P_{t-1}), window = 21) x sqrt(252)
- VIX level (implied vol proxy)
- Vol risk premium: VIX/100 minus realized vol (positive = calm, negative = stress)
- Yield curve slope: 10Y minus 2Y Treasury spread
- Rate of change of 10Y yield (monthly first difference — rising vs. high-but-stable)
- Cross-sectional return dispersion: sd() of 11 ETF returns in each period
- Average pairwise ETF correlation (rolling)
- Market momentum: SPY rolling 12-1 month return
- Credit spread: HYG minus IEI if API available

Dimensionality: ~10 features, hundreds of time periods — PCA is well-conditioned here.
- Normalize, PCA, cluster on top 2-3 PCs
- Parameter selection: Gap statistic + silhouette + seed stability (ARI) across k in {2, 3, 4}
- Majority voting to resolve overlapping window labels (Raju 2025, Section 3.4)
- Validate: regime timeline plot annotated with 2008 GFC / 2020 COVID / 2022 rate hikes
- Output: regime labels updated monthly

### Layer 3: Kalman Filter (smoothing layer)
**Purpose:** Reduce whipsawing in regime detection.

Apply Kalman filter to rolling macro factor estimates (yield trend, realized vol, momentum)
BEFORE feeding into regime clustering features. Prevents a 3-day spike from triggering a
regime shift; provides adaptive state estimation rather than raw noisy rolling estimates.

Must explain the Kalman Filter role explicitly in the writeup — earns marks only if justified.

### Signal Bridge (state these hypotheses BEFORE computing)

| Regime           | Macro Characteristics                        | Overweight     | Underweight    |
|------------------|----------------------------------------------|----------------|----------------|
| Bull / Growth    | Low VIX, positive momentum, normal curve     | XLK, XLY, XLI | Defensive      |
| Bear / Stress    | High VIX, inverted curve, wide credit spread | XLV, XLP, XLU | Cyclical       |
| Commodity / Infl | Rising WTI, steepening curve, high rates     | XLE, XLB, XLF | Rate-sensitive |
| Transition       | Mixed signals                                | Equal-weight   | —              |

**Architectural note:** WTI sensitivity is NOT a clustering feature for asset clustering — PCA
would suppress it since only XLE/XLB have meaningful WTI beta. WTI lives in the REGIME
features and routes to high-WTI-beta ETFs via the signal bridge. Deliberate design decision.

---

## SECTION 4: SIGNAL STATISTICAL SIGNIFICANCE (PRIMARY FIN 440 FAILURE)

This is what killed the FIN 440 QTP. Must address explicitly for every regime-conditional result.

Required for each regime x ETF cluster pairing:
- t-test on mean return difference between regimes
- Memmel z-test on Sharpe ratio differences (Raju 2025, Section 3.5)
- HAC-robust standard errors (Newey-West, 3-month truncation lag)
- Holm-Bonferroni correction for multiple comparisons
- Report: mean return by regime, Sharpe by regime, z-statistic, adjusted p-value
- State explicitly: "This signal is [statistically significant / not significant] at the X% level"

Benchmark from Raju (2025): bear regimes favor defensive factors (Sharpe differential 2-4 pts,
p<0.01 across all factor types). Use as external validation of your regime-conditional results.

Endogeneity warning (Raju 2025, Section 4.2.4): all 11 ETFs correlate 0.6-0.95 with SPY.
If SPY return is in your regime features, regime effects will be mechanically inflated for
high-beta ETFs. Explicitly test whether results hold after controlling for market beta.

---

## SECTION 5: RISK APPETITE ARTICULATION (2 pts — lost marks here in FIN 440)

Must explicitly state ALL of the following:
1. Maximum drawdown tolerance (e.g., strategy targets max drawdown < 20%)
2. Optimization objective: maximize Omega Ratio (threshold = 0), subject to Sharpe > 0.5
3. Position sizing rule: equal-weight within selected cluster, OR vol-weighted (1/sigma), OR conviction-weighted
4. Defensive trigger: define explicitly (e.g., if regime = stress AND avg pairwise corr > 0.85 → 100% defensive cluster)
5. Parameter optimization surface: grid search → heat map (geom_raster + scale_fill_gradient2)

Multi-criteria optimization (required — single criterion = -1 pt):
- Primary objective: Omega Ratio
- Secondary filter: Sharpe Ratio > 0.5
- Hard constraint: Max Drawdown < 20%
- Show 2D optimization surface over at minimum 2 parameters (e.g., lookback window x k clusters)

---

## SECTION 6: IMPLEMENTATION CHECKLIST

### Data and Preprocessing
- [ ] All data via tq_get() — no local files, no hardcoded paths
- [ ] Log returns: mutate(ret = log(adjusted / lag(adjusted)))
- [ ] Roll adjustment for any futures-based features: RTL::rolladjust()
- [ ] Date filters only — NO slice() with hardcoded indices
- [ ] Training/testing split: filter(date < "2025-01-01") vs filter(date >= "2025-01-01")

### Asset Clustering
- [ ] All features computed over training window only (zero look-ahead)
- [ ] scale() before PCA
- [ ] prcomp() — report variance explained, scree plot, biplot with ETF labels
- [ ] K-Means (kmeans()) AND Hierarchical (hclust() + cutree()) — compare dendrograms
- [ ] Validate: silhouette (cluster::silhouette()), WSS elbow plot
- [ ] Assign commercial cluster names — not "Cluster 1, 2, 3"

### Regime Clustering
- [ ] Rolling 3-month windows with 1-month implementation lag
- [ ] Majority voting for overlapping window label assignment
- [ ] Gap statistic (cluster::clusGap()) + ARI across k in {2, 3, 4}
- [ ] Kalman filter applied to key features BEFORE clustering
- [ ] Validate: regime timeline annotated with 2008 GFC / 2020 COVID / 2022 rate hikes

### Signal Generation
- [ ] Signal = regime cluster assignment, updated monthly
- [ ] Execute at Open(t+1) based on signal from Close(t) — verify lag
- [ ] position_change = signal(t) - signal(t-1) triggers trade
- [ ] Mermaid decision diagram documenting full signal logic

### Backtesting
- [ ] Transaction costs documented and applied
- [ ] retClCl for overnight; retOpCl for intraday
- [ ] Metrics: cumReturn, Sharpe, Omega, Sortino, maxDrawdown, Calmar, hit rate
- [ ] SPY buy-and-hold as benchmark throughout
- [ ] t-test on strategy excess returns vs benchmark
- [ ] Quarterly chart panels (required by rubric)

### Optimization
- [ ] Strategy wrapped in a clean function: strategy_fn(params) → metrics tibble
- [ ] purrr::pmap() or foreach/doParallel for parallel grid search
- [ ] Heat map of optimization surface (geom_raster + scale_fill_gradient2)
- [ ] Parameters selected from training period ONLY — no refitting on testing data

---

## SECTION 7: REQUIRED DOCUMENT STRUCTURE

Prof feedback on FIN 440 QTP: "flow requires back and forth between data and commentary."
Every chart must be immediately followed by a 3-5 sentence interpretation paragraph.

1. **Executive Summary** — 3 sentences: strategy thesis, training result, testing result
2. **Intuitive Rationale** — commercial thesis, paper citations, explicit failure conditions
3. **Data and Features** — gt::gt() table of all features used in each clustering layer
4. **Mental Model** — Mermaid diagram of the full three-layer pipeline
5. **Asset Clustering**
   - PCA: scree plot → biplot → [commentary: what do the PCs mean commercially?]
   - Clustering: dendrogram + K-Means plot → [commentary: name each cluster, justify]
   - Validation: silhouette width table → [commentary: are clusters well-separated?]
6. **Regime Detection**
   - Regime timeline with shaded regions → [commentary: does this match market history?]
   - Regime-conditional performance table (Sharpe by regime) with z-stats and p-values
   - [commentary: which signals are statistically significant? endogeneity check]
7. **Signal Logic and Execution Rules** — Mermaid decision tree + written rules
8. **Training Period Backtest**
   - Equity curve (strategy vs SPY) → [commentary]
   - Drawdown chart → [commentary]
   - Performance table gt::gt() → [commentary: interpret every metric]
9. **Optimization**
   - Heat map → [commentary: where is the stable region? why these parameters?]
   - Selected parameters with explicit risk appetite justification
10. **Testing Period Backtest** — same charts as training, honest assessment
11. **Risk Appetite Statement** — explicit, structured, quantified
12. **Lessons Learned** — MOST IMPORTANT SECTION (minimum 5 points, 3-5 sentences each)

### Document Hygiene
- [ ] No raw dataframe prints — all tables via gt::gt() with fmt_number / fmt_percent
- [ ] All chunks: message=FALSE, warning=FALSE, echo=FALSE unless showing code is intentional
- [ ] No commented-out code blocks in final render
- [ ] Every plot: title, subtitle, axis labels with units, caption citing data source
- [ ] Self-contained HTML: embed-resources: true in YAML header

---

## SECTION 8: PAPERS TO CITE

**Ajayi (2025)** — Detecting Market Instability with Regime Switching Models (ssrn-5366273)
- Use for: validating two-regime concept; Markov Switching as methodology comparison
- Key argument: parametric MRS identifies same GFC/COVID regimes as K-Means (convergent validity)
- Limitation vs your design: uses market returns only; yours adds WTI, rates, VIX (more exogenous)

**Raju (2025)** — Wasserstein k-Means for Bull-Bear Regimes: Evidence from Indian Equities (ssrn-5539698)
- Use for: rolling window spec (3-month, 1-month lag), majority voting, Gap statistic + ARI,
  regime-conditional Sharpe benchmarks, endogeneity warning
- Key finding: bear regimes favor defensive factors (Sharpe differential 2-4 pts, p<0.01)
- Difference from your design: Wasserstein distance vs Euclidean K-Means — justify the simpler choice

---

## SECTION 9: LESSONS FROM FIN 440 QTP (pre-seed for writeup)

1. Fixed parameters over a multi-year window failed — 2020/2022 dominated optimization;
   rolling window parameters and re-estimation periods are required this time
2. Signal statistical significance was never tested — this design uses Memmel z-test,
   HAC errors, and Holm-Bonferroni correction explicitly for every signal
3. Risk appetite articulation was vague — this time: explicit multi-criteria optimization
   (Omega primary, Sharpe constraint, drawdown hard cap) with 2D optimization surface
4. Outlier influence was unexamined — test robustness with and without 2020 COVID / 2022 Ukraine
5. Business document quality was an afterthought — structure is written first this time;
   every chart immediately followed by commercial interpretation paragraph
6. Single performance criterion (cumulative return only) — this submission uses Sharpe,
   Omega, Sortino, drawdown, Calmar, and hit rate at minimum

---

## SECTION 10: KNOWN BUG PATTERNS

**From global CLAUDE.md (Section 5):**
- NO slice() with hardcoded indices — always filter(date >= "YYYY-MM-DD")
- No hardcoded API credentials in committed code (.Renviron for keys)

**QTP-specific:**
- Signal lag: signal generated using data through Close(t), position enters at Open(t+1)
  Verify: overnight return uses lag(position), not current position
- Same-day trade bug: retClOp (intraday) must use PREVIOUS signal, not same-day signal
- Look-ahead bias: prcomp(), kmeans(), scale parameters ALL fit on training data only;
  apply to testing via predict(pca_model, newdata = test_features)
- Label-switching: K-Means cluster numbers are arbitrary — always re-label by economic
  characteristic each period (e.g., lowest-avg-vol cluster = "defensive")
- Roll adjustment: RTL::rolladjust() required for any continuous futures series

---

## SECTION 11: COMMERCIAL INTERPRETATION PROMPTS

After every result, answer these before moving to the next section:
- What does this cluster / regime tell a portfolio manager?
- Would a desk head act on this signal? Why or why not?
- What is the cost of a false signal?
- Does this result hold out-of-sample? If not, why not?

Target language: "Regime 1 (inverted yield curve, VIX > 25, negative momentum) captured
14 of the 17 months surrounding the 2008 GFC and all 4 months of the 2020 COVID shock.
A portfolio manager rotating into XLP, XLV, and XLU at the onset of each detected regime
would have reduced max drawdown by X% relative to SPY, with a Sharpe improvement of Y."

--- 

### Macro Data Sources (all via tq_get unless noted)
- VIX:              tq_get("^VIX", from = "2000-06-01")
- 10Y Treasury:     tq_get("DGS10", get = "economic.data")   # FRED
- 2Y Treasury:      tq_get("DGS2",  get = "economic.data")   # FRED
- Yield curve slope: DGS10 - DGS2 (computed)
- Credit spread:    HYG and IEI via tq_get() — spread = HYG return - IEI return
- WTI (regime use): tq_get("CL=F") or RTL::getPrices() with roll adjust
- Sector ETFs:      tq_get(c("SPY","XLB","XLC","XLE","XLF","XLI",
                             "XLK","XLP","XLRE","XLV","XLU","XLY"))
- Realized vol:     Computed from ETF log returns (rolling 21-day sd x sqrt(252))
- Vol risk premium: VIX/100 - realized vol (computed)

### Kalman Filter Implementation
- Package: dlm (preferred) or KFAS
- Apply to: yield slope, realized vol, and momentum BEFORE clustering
- Purpose: smooth noisy rolling estimates so a 3-day spike doesn't trigger regime shift
- Simple local level model per feature: dlmModPoly(order=1) + dlmFilter()
- Output: filtered state estimates replace raw rolling values as clustering inputs

