# Code Description
## asian_option_pricing.ipynb
**Stochastic Methods in Finance — Assignment, Spring Semester 2026**
*University of St. Gallen*

---

## Overview

The notebook prices a **European floating-strike Asian call option** on Apple Inc. stock using a multi-period binomial tree model. It implements all seven tasks from the assignment brief.

**Payoff:** $A_T = \max(S_n - \bar{S}_n, 0)$, where $\bar{S}_n$ is the arithmetic average of stock prices over the life of the option.

**Data:** `apple stock data.xlsx` — official AAPL closing prices provided by Prof. De Giorgi on Canvas (28 April 2026), covering 2000-01-03 to 2026-04-27.

---

## Section-by-Section Description

### Section 1 — Calibration (Task: data preparation)
Loads the official AAPL dataset and filters to the 2020–2026 window specified in the assignment. Computes daily log-returns, estimates $\sigma_{\text{daily}}$ via the sample standard deviation (Bessel-corrected), and annualises as $\sigma = \sigma_{\text{daily}} \cdot \sqrt{250}$. Sets $S_0$ to the latest available closing price (27 April 2026). Produces a two-panel plot of AAPL prices and log-returns.

### Section 2 — Binomial Tree Setup (Task iv)
Derives the risk-neutral probability $q$ from the martingale condition on discounted stock prices. Sets the CRR up/down factors $u = e^{\sigma\sqrt{\Delta t}}$, $d = 1/u$, computes $q = (e^{r\Delta t} - d)/(u-d)$, and asserts the no-arbitrage condition $0 < q < 1$.

### Section 3 — Path-Dependent Backward Induction (Tasks i, ii, iii, v)
Defines `price_floating_asian(S0, sigma, r, T, n)`, which:
- **Forward pass:** enumerates all reachable augmented states $(j, C_t)$ where $j$ = up-count and $C_t = \sum_{s=0}^t S_s$ is the running sum. The state container is a list of plain Python dicts: `states[t][j]` is a `set` of all reachable cumulative sums $C_t$, and new dict entries are created with explicit `if key not in d:` checks (no external libraries needed).
- **Terminal payoffs:** computes $\max(S_n - C_n/(n+1), 0)$ at each terminal state.
- **Backward pass:** rolls back option values using $V_t(j,C) = e^{-r\Delta t}[q\,V_{t+1}^{\text{up}} + (1-q)\,V_{t+1}^{\text{down}}]$.

Returns the unique price at $t=0$. Also explains *why* the option is path-dependent (same $S_n$ can yield different payoffs depending on the path) and provides a concrete $n=2$ worked example.

### Section 4 — Robustness Check (Task vi)
Re-prices the option on a $5\times5$ grid of $(\sigma, r)$ values — $\sigma$ at $\pm 10\%$/$\pm 20\%$ of the calibrated value, $r$ from $0\%$ to $2\%$. Uses two nested `for`-loops (one over $\sigma$, one over $r$) to fill in the result table cell by cell. Displays results as a table and a colour heatmap. Comments: price is monotone in $\sigma$; sensitivity to $r$ is mild because $r$ shifts both $S_n$ and $\bar{S}_n$ together.

### Section 5 — Monte Carlo Cross-Check (independent verification)
Simulates 200,000 GBM paths under $Q$ using the Itô-corrected drift $(r - \frac{1}{2}\sigma^2)\Delta t$. Computes discounted average payoffs and a 95% confidence interval. Serves as an independent sanity check on the binomial price.

### Section 6 — Normal Approximation (Task vii)
Approximates $D_n = S_n - \bar{S}_n \sim \mathcal{N}(\mu_D, \sigma_D^2)$ and prices the option using the truncated normal identity:
$$\mathbb{E}[\max(X,0)] = \mu\,\Phi(\mu/s) + s\,\phi(\mu/s)$$
Derives $\mu_D$ and $\sigma_D^2$ from first principles using one-period gross-return moments ($m_1$, $m_2$), marginal price moments, and an $(n+1)\times(n+1)$ covariance matrix filled in with two nested `for`-loops (one over $s$, one over $t$) — entry by entry, using the closed-form formula $\operatorname{Cov}(S_s, S_t) = S_0^2 (m_2^{\min(s,t)} m_1^{|s-t|} - m_1^{s+t})$. The variance of $D_n$ is then assembled by summing covariance entries with explicit loops. Compares the result to the exact binomial price and plots the tree-implied $Q$-distribution of $D_n$ against the fitted normal density.

### Section 7 — Summary
Prints a single table of all key outputs: $S_0$, $\sigma$, model parameters, binomial price, Monte Carlo estimate (with 95% CI), and normal-approximation price.

---

## Dependencies

| Library | Version | Purpose |
|---------|---------|---------|
| `numpy` | ≥1.24 | Arrays, `np.exp`, `np.sqrt`, `np.maximum`, `np.cumsum`, RNG |
| `pandas` | ≥2.0 | Excel loading, time-series handling, results table |
| `openpyxl` | ≥3.1 | Excel file backend for `pd.read_excel` |
| `scipy` | ≥1.10 | `norm.cdf`, `norm.pdf` for normal approximation |
| `matplotlib` | ≥3.7 | Plots (inline in notebook) |

Install with: `pip install numpy pandas openpyxl scipy matplotlib`

## How to run

1. Open `asian_option_pricing.ipynb` in Jupyter Lab or Jupyter Notebook from the `Project/` directory (so that `apple stock data.xlsx` is found at the relative path `./apple stock data.xlsx`).
2. Kernel → Restart & Run All.
3. All results and figures appear inline.

---

## Style notes

The code is intentionally written in a simple, beginner-friendly style:
- **Plain Python `dict`s** instead of `collections.defaultdict`. New keys are added with explicit `if key not in d:` checks, so the data flow is visible at every line.
- **Nested `for`-loops** instead of `itertools.product` for the $(\sigma, r)$ robustness grid.
- **Explicit `for`-loops** instead of `np.meshgrid` / `np.outer` to build the covariance matrix and to sum its entries — every index $(s, t)$ is visited by hand.
- **No list comprehensions** for non-trivial work; loops `append` results step by step.

The output numbers are unchanged; only the implementation is more transparent.
