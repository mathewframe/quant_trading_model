# Plan: New QTP Document — Cluster-Based Relative Value Strategy (Long-Only)

## File
Create from scratch: `C:\Users\mattw\dev\quant_trading_model\Mathew_Frame_QTP.qmd`
The old file (MathewFrame_QTP_Fin452.qmd) is retired. New file, clean start.

---

## Architecture (Locked)

### Two-Frequency Structure
**Monthly:** Re-run hierarchical clustering on the 60-day rolling return correlation matrix
at each month-end. Produces ETF cluster assignments that hold for the coming month.
k is NOT fixed -- determined by elbow method (WSS) on training data.

**Daily (intra-month):** Within the fixed monthly clusters, compute z-scores of each ETF's
return relative to its cluster mean. Weight inversely proportional to z-score rank.
Kalman Filter hard gate applied before weighting.

### Clustering
- Input: 60-day rolling pairwise return correlation matrix
- Distance: d_ij = 1 - corr(i,j)
- Method: Hierarchical clustering, Ward.D2 linkage
- k: determined by elbow method (WSS) on training data -- parameterizable in optimization
- Output: cluster assignments per ETF, updated monthly

### Signal Generation
- For each ETF i in cluster C: rel_i = r_i - mean(r_j for j in C)
- Rolling 20-day std dev of rel_i: sigma_i
- z_i = rel_i / sigma_i
- Within cluster: rank ETFs by z-score ascending (lowest = most underperformed)
- Weight: inversely proportional to rank (1st rank = highest weight)
- Across clusters: equal weight (1/k per cluster)
- All weights normalized to sum to 1.0

### Kalman Filter (Hard Gate)
- Applied to each ETF's adjusted close price series
- Package: dlm, local level model (dlmModPoly order=1) + dlmFilter
- Output: smoothed price level + implied slope (trend direction)
- Rule: if Kalman slope < -threshold (ETF in confirmed downtrend), exclude that ETF
  entirely. Redistribute its cluster weight equally among remaining cluster members.
- Threshold: parameterizable in optimization
- Rationale: avoids catching a falling knife when underperformance is trend, not noise

### Execution Rules
- Signal computed at Close(t) using data through Close(t)
- Position entered at Open(t+1) -- never same-day
- Monthly cluster reassignment: computed at last Close of month, applied at first Open of new month
- Transaction costs: applied on |weight change| per ETF per day
- Benchmark: SPY buy-and-hold

### Long-Only Constraints
- All weights >= 0
- Weights sum to 1.0
- Max weight per ETF: 20% (hard cap)
- No shorts, no cash

---

## Document Structure

1. Executive Summary (written last)
2. Intuitive Rationale
   - Strategy thesis (humble: testing a hypothesis)
   - What clustering captures that directional forecasting cannot
   - When it works / when it fails (trend regimes, correlation collapse)
   - Paper citations: Ajayi 2025, Raju 2025
3. Data and Feature Selection
   - ETF universe + SPY benchmark
   - Data sources table (gt::gt)
   - Log returns
   - 60-day rolling correlation matrix as the clustering input
   - No macro features needed
4. Clustering
   - Elbow method (WSS) to select k
   - Hierarchical clustering on distance matrix
   - Silhouette validation
   - Sample cluster visualization (one month snapshot)
   - Commercial interpretation: what do the clusters represent?
5. Kalman Filter
   - Explain: smoothed price level + slope = trend direction
   - Hard gate rule
   - Visualize: one ETF with Kalman level + slope overlaid
   - How often does the gate fire? (% of ETF-days excluded)
6. Signal Generation
   - Within-cluster z-scores
   - Inverse rank weighting
   - Cross-cluster equal weight
   - Full weight calculation example (one month, one cluster)
7. Execution and Implementation + Mental Model
   - Mermaid diagram: full pipeline
   - Execution rules (signal lag, transaction costs)
   - Position sizing math
8. Training Period Backtest (Jun 2000 -- Dec 2024)
   - Equity curve vs SPY
   - Drawdown chart
   - Performance table: Sharpe, Omega, Sortino, maxDD, Calmar, hit rate
9. Risk Appetite
   - Max drawdown tolerance
   - Optimization objective: Omega primary, Sharpe > 0.5 filter, maxDD < 20% hard cap
   - Defensive trigger: Kalman gate fires on >50% of ETFs -> go to equal weight
10. Optimization (training data only)
    - Parameters: lookback window (40-80d), k (2-4), z-score rank weight steepness,
      Kalman threshold, vol window (10-30d)
    - purrr::pmap or doParallel grid search
    - Heat map: geom_raster + scale_fill_gradient2
    - Selected parameters justified
11. Testing Period Results (Jan 2025 -- Mar 2026)
    - Same charts as training, honest assessment
    - No refitting
12. Lessons Learned
    - Min 5 points x 3-5 sentences each
    - Reference FIN 440 QTP failures explicitly

---

## Build Order (Section by Section)

Quiz user before each section. Never build more than one at a time.

1. YAML + setup chunk + data pull (Section 3 skeleton)
2. Section 2: Intuitive Rationale (prose only)
3. Section 3: Data table + log returns + rolling correlation setup
4. Section 4: Clustering (elbow, hierarchical, silhouette, visualization)
5. Section 5: Kalman Filter (dlm, hard gate, visualization)
6. Section 6: Signal generation (z-scores, ranking, weights)
7. Section 7: Mermaid + execution rules
8. Section 8: Training backtest
9. Section 9: Risk appetite
10. Section 10: Optimization
11. Section 11: Testing period
12. Section 12: Lessons learned
13. Section 1: Executive summary (last)

---

## Section Workflow (CRITICAL)
For every section:
1. Quiz Mathew on what the section will implement and why
2. Build only after he has reviewed, understood, and approved
3. After rendering, review output together -- insights from one section inform the next
   (e.g., elbow method chart determines k before clustering code is written)
4. Never move to the next section until Mathew explicitly approves the current one

## Key Rules
- No em dashes in prose
- Every chart immediately followed by 3-5 sentence interpretation
- All tables via gt::gt()
- message=FALSE, warning=FALSE, echo=FALSE on all chunks
- Date filters only -- no slice() with hardcoded indices
- Signal at Close(t), execute at Open(t+1)
- Training/testing split: filter(date <= "2024-12-31") vs filter(date >= "2025-01-01")
- All parameters fit on training data only
