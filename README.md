# FIN 452 — Quantitative Trading Model
**Mathew Frame | University of Alberta | April 2026**

---

## Strategy Overview

A cluster-based sector rotation strategy applied to the S&P 500 Select Sector SPDR ETFs (SPY + XLB, XLC, XLE, XLF, XLI, XLK, XLP, XLRE, XLV, XLU, XLY).

The strategy classifies ETFs into **Cyclical** and **Defensive** clusters using K-Means and hierarchical clustering on training-period risk/return features. A **Kalman Filter** and 20-day SMA generate a spread signal that rotates capital toward the cluster with positive momentum, or steps aside into cash when both clusters deteriorate simultaneously.

- **Training period:** June 2000 – December 2024  
- **Testing period:** January 2025 – March 2026  
- **Rebalancing:** Monthly signal, daily execution at Open(t+1)  
- **Optimization:** Parallel grid search maximizing Omega Ratio, Calmar > 0.10 filter

---

## Files

| File | Description |
|------|-------------|
| `Mathew_Frame_QTP.qmd` | Main Quarto document — all analysis, code, and commentary |
| `Mathew_Frame_QTP.html` | Self-contained rendered HTML |
| `context.md` | Project context and CLAUDE.md instructions |
| `CLAUDE.md` | AI assistant context file |

---

## Key Packages

```r
tidyverse, tidyquant, RTL, cluster, factoextra,
dlm, TTR, slider, xts, timetk, gt, plotly,
ggrepel, patchwork, foreach, doParallel
