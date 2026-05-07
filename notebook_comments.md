# Extracted Comments and Markdown from `asian_option_pricing.ipynb`

These were stripped out of the notebook so you can re-add explanations as you work through the code yourself. They are listed in original cell order.

## Cell 0 — markdown

# Pricing a European Floating-Strike Asian Call Option
**Stochastic Methods in Finance — Assignment, Spring Semester 2026**
*University of St. Gallen*

**Group members:** [FILL IN BEFORE SUBMISSION]

---

## Derivative
The payoff at maturity $T$ is:
$$A_T = \max\!\left(S_n - \bar{S}_n,\; 0\right), \qquad \bar{S}_n = \frac{1}{n+1}\sum_{t=0}^{n} S_t$$
where $S_t$ is the stock price at period $t$ and $\bar{S}_n$ is the arithmetic average over all $n+1$ prices. The strike is the average itself — hence *floating-strike*.

## Tasks covered
| Task | Description |
|------|-------------|
| (i) | Build the binomial tree for $S_t$, $t=0,\ldots,n$ |
| (ii) | Compute arithmetic averages $\bar{S}_n$ at terminal nodes |
| (iii) | Compute terminal payoffs |
| (iv) | Derive risk-neutral probabilities |
| (v) | Apply backward induction; explain path dependence |
| (vi) | Robustness check over $r$ and $\sigma$ |
| (vii) | Normal approximation for $D_n = S_n - \bar{S}_n$ |

**Data source:** Official dataset `apple stock data.xlsx` provided by Prof. De Giorgi on Canvas (28 April 2026) because Yahoo Finance now requires a subscription.

## Cell 1 — code

```python
# plotting
# numerical arrays
# tabular data / Excel loading
# standard normal CDF / PDF
```

## Cell 2 — markdown

## Section 1 — Calibration from AAPL Historical Data

### Why log returns?
The annualised volatility $\sigma$ is estimated from the sample standard deviation of daily **log-returns**:
$$r_t = \log\!\left(\frac{S_t}{S_{t-1}}\right)$$

Log returns are the natural choice because:
1. The binomial model specifies $\log(S_{t+\Delta t}/S_t)$ as the elementary random variable.
2. The assignment formulates the variance condition $\operatorname{Var}[\log(S_n/S_0)] = \sigma^2 T$ in log-return terms.
3. Log returns are **additive** over time: $\log(S_n/S_0) = \sum_{t=1}^n\log(S_t/S_{t-1})$, so variance scales linearly with the horizon.

### Annualisation factor $\sqrt{250}$
Assuming 250 trading days per year and **i.i.d.** daily log-returns:
$$\operatorname{Var}(R_{\text{year}}) = \sum_{t=1}^{250}\operatorname{Var}(r_t) = 250\,\sigma_{\text{daily}}^2$$
Taking the square root:
$$\boxed{\sigma = \sigma_{\text{daily}}\,\sqrt{250}}$$
The sample standard deviation uses Bessel's correction ($\text{ddof}=1$):
$$\hat{\sigma}_{\text{daily}} = \sqrt{\frac{1}{N-1}\sum_{t=1}^{N}(r_t - \bar{r})^2}$$

### Initial price $S_0$
The assignment specifies the closing price on **28 April 2026**. The official dataset ends on 27 April 2026 — the professor posted the file at 13:27 CET on 28 April, before the US market close. The latest available close (27 April 2026) is therefore used as $S_0$.

## Cell 3 — code

```python
# Load the official AAPL dataset (same directory as this notebook)
# Date-indexed closing price Series (split-adjusted, 2000-01-03 → 2026-04-27)
# Calibration window: 2020-01-01 → latest available (assignment: "2020 to 2026")
# Daily log-returns: r_t = log(S_t / S_{t-1})
# Sample std of daily log-returns (Bessel-corrected, ddof=1 by default)
# Annualised volatility: σ_annual = σ_daily · √250
# S_0: use 28 Apr 2026 if available, else latest (27 Apr 2026)
```

## Cell 5 — markdown

## Section 2 — Binomial Tree Setup

### Model parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| $T$ | 0.5 years | Maturity |
| $n$ | 25 | Number of periods |
| $r$ | 1% p.a. | Continuously-compounded risk-free rate |
| $\Delta t = T/n$ | 0.02 years | Step size |
| $u = e^{\sigma\sqrt{\Delta t}}$ | — | CRR up factor |
| $d = e^{-\sigma\sqrt{\Delta t}} = 1/u$ | — | CRR down factor |
| $p$ | 1/2 | Physical probability of an up-move |

### Derivation of the risk-neutral probability $q$ — Task (iv)

Under the risk-neutral measure $Q$, the discounted stock price $e^{-rt}S_t$ is a $Q$-martingale. For one step:
$$S_t = e^{-r\Delta t}\,\mathbb{E}^Q[S_{t+1} \mid \mathcal{F}_t]$$

Substituting the binomial dynamics $S_{t+1} = S_t u$ (prob $q$) or $S_t d$ (prob $1-q$):
$$1 = e^{-r\Delta t}\bigl[qu + (1-q)d\bigr]$$
$$e^{r\Delta t} = qu + (1-q)d = q(u-d) + d$$

Solving for $q$:
$$\boxed{q = \frac{e^{r\Delta t} - d}{u - d}}$$

**No-arbitrage condition:** $0 < q < 1$ requires $d < e^{r\Delta t} < u$, which holds whenever $\sigma\sqrt{\Delta t} > |r\Delta t|$, comfortably satisfied here.

## Cell 6 — code

```python
# maturity (years)
# number of binomial periods
# continuously-compounded risk-free rate (p.a.)
# step size
# CRR up/down factors — calibrated so Var[log(S_n/S_0)] = σ²T
# = 1/u (recombining tree)
# Risk-neutral probability (derived above)
# Per-step discount factor
# No-arbitrage check: 0 < q < 1
```

## Cell 7 — markdown

## Section 3 — Path-Dependent Backward Induction — Task (v)

### Why this option is path-dependent

A standard European call has payoff $\max(S_n - K, 0)$. In a recombining CRR tree, $S_n = S_0 u^j d^{n-j}$ depends only on the **count** $j$ of up-moves, not the order — same $(n,j)$ gives same $S_n$ gives same payoff. The tree recombines.

For the floating-strike Asian call:
$$A_T = \max(S_n - \bar{S}_n, 0), \qquad \bar{S}_n = \frac{C_n}{n+1}, \qquad C_n = \sum_{t=0}^n S_t$$

The cumulative sum $C_n$ depends on **every intermediate price**, not just on $j$. Two paths with the same $(n,j)$ — hence the same $S_n$ — can have different $C_n$ if the sequence of ups and downs differs:

> **Same final stock price $\;\not\!\!\!\Rightarrow\;$ same option value.**

#### Concrete $n=2$ example

Take $S_0=100$, $u=1.10$, $d=0.90$. Paths **UD** and **DU** both end at $S_2=99$ ($j=1$):

| Path | $C_2 = S_0+S_1+S_2$ | $\bar{S}_2 = C_2/3$ | Payoff $\max(S_2-\bar{S}_2,0)$ |
|------|---------------------|----------------------|----------------------------------|
| UD (up then down) | $100+110+99=309$ | $103.0$ | $\max(99-103,0)=0$ |
| DU (down then up) | $100+90+99=289$ | $96.\overline{3}$ | $\max(99-96.33,0)=2.67$ |

Same $S_2=99$, different payoffs — the option value at node $(t=2, j=1)$ is **not** a function of the stock price alone.

### Augmented state $(j, C_t)$

We track the pair $(j, C_t)$ at every node. The dynamics are:
$$\text{up:}\quad (j, C_t)\to(j+1,\; C_t + S_t \cdot u)$$
$$\text{down:}\quad (j, C_t)\to(j,\; C_t + S_t \cdot d)$$
making the system Markovian on the augmented state space. The backward-induction recursion is:
$$V_t(j, C) = e^{-r\Delta t}\bigl[q\,V_{t+1}(j+1,\,C+S_t u) + (1-q)\,V_{t+1}(j,\,C+S_t d)\bigr]

## Cell 8 — code

```python
    """
    Price a European floating-strike Asian call on a recombining binomial tree.

    The state at each node is augmented to (j, C_t) where:
      j   = number of up-moves so far
      C_t = sum of stock prices S_0 + S_1 + ... + S_t   (the running sum)

    We need C_t because the payoff at maturity depends on the average
    of all prices along the path, not just on S_n.
    """
    # --- model parameters for this call ---
    # --- FORWARD PASS: enumerate reachable (j, C_t) states  [Tasks (i)+(ii)] ---
    # states[t] is a dict where:
    #     key   = j (number of up-moves at time t)
    #     value = set of all reachable cumulative sums C_t with that j
# at t=0: 0 up-moves, only C_0 = S_0
# Task (i): stock price at node (t,j)
                # up-move: j increases by 1, C grows by S_now * u
                # down-move: j stays the same, C grows by S_now * d
    # --- TERMINAL PAYOFFS  [Task (iii)] ---
    # values[t][j] is a dict mapping C -> option value V(t, j, C)
            # floating-strike Asian call payoff: max(S_n - average, 0)
    # --- BACKWARD INDUCTION  [Task (v)] ---
    # Roll values back from t=n to t=0 using the risk-neutral pricing formula.
    # The unique state at t=0 has j=0 and C_0 = S_0
```

## Cell 10 — markdown

## Section 4 — Robustness Check — Task (vi)

We re-price the option on a $5\times 5$ grid: $\sigma$ ranges from $80\%$ to $120\%$ of the calibrated value, and $r$ ranges from $0\%$ to $2\%$ p.a. This tests sensitivity to modelling assumptions.

## Cell 11 — code

```python
# Empty DataFrame: rows = sigma values, columns = r values
# Loop over every (sigma, r) pair in the grid and store the price
```

## Cell 13 — markdown

### Comments on robustness

- **Monotone in $\sigma$:** The price increases strictly with volatility. Higher $\sigma$ widens the distribution of $S_t$, increasing the expected gap $S_n - \bar{S}_n$ and hence the call value.

- **Mild dependence on $r$:** Both $S_n$ and every term in $\bar{S}_n$ are shifted upward by a higher $r$ (via the $e^{r\Delta t}$ factor in the risk-neutral drift). Since the payoff depends on the *difference* $S_n - \bar{S}_n$, these shifts partially cancel — hence the small positive but limited sensitivity to $r$.

## Cell 14 — markdown

## Section 5 — Monte Carlo Cross-Check (Independent Sanity Check)

Monte Carlo is not required by the assignment, but provides an independent verification of the binomial price by simulating paths directly under the risk-neutral measure $Q$.

### GBM under $Q$
By Itô's lemma applied to $\log S_t$ (where $dS_t = rS_t\,dt + \sigma S_t\,dW_t$ under $Q$):
$$\log S_{t+\Delta t} - \log S_t = \underbrace{\left(r - \tfrac{1}{2}\sigma^2\right)\Delta t}_{\text{Itô-corrected drift}} + \sigma\sqrt{\Delta t}\,Z_t, \quad Z_t\overset{\text{iid}}{\sim}\mathcal{N}(0,1)$$

The $-\tfrac{1}{2}\sigma^2$ correction ensures $\mathbb{E}^Q[S_{t+\Delta t}] = S_t\,e^{r\Delta t}$, consistent with the binomial martingale condition.

The MC estimator is:
$$\hat{V} = e^{-rT}\,\frac{1}{M}\sum_{m=1}^M\max\!\left(S_n^{(m)} - \bar{S}_n^{(m)},\, 0\right)$$

## Cell 15 — code

```python
    """Monte Carlo price under GBM risk-neutral measure (independent verification)."""
    # (n_sims × n) matrix of standard normals
    # Log-increments under Q with Itô-corrected drift
    # Full price paths: shape (n_sims, n+1), includes t=0
# arithmetic average per path
# floating-strike payoff
# discounted average
```

## Cell 16 — markdown

## Section 6 — Normal Approximation — Task (vii)

We approximate $D_n = S_n - \bar{S}_n$ as normally distributed, then use the truncated normal expectation to compute the option price analytically.

### Key identity: truncated normal expectation

**Claim.** If $X\sim\mathcal{N}(\mu, s^2)$, then:
$$\mathbb{E}[\max(X,0)] = \mu\,\Phi\!\left(\frac{\mu}{s}\right) + s\,\phi\!\left(\frac{\mu}{s}\right)$$

**Derivation.** With density $f(x) = \frac{1}{s}\phi\!\left(\frac{x-\mu}{s}\right)$:
$$\mathbb{E}[\max(X,0)] = \int_0^\infty x\,\frac{1}{s}\phi\!\left(\frac{x-\mu}{s}\right)dx$$

Substitute $z=(x-\mu)/s$, lower limit $z=-\mu/s$:
$$= \int_{-\mu/s}^\infty(\mu+sz)\,\phi(z)\,dz = \mu\underbrace{\int_{-\mu/s}^\infty\phi(z)\,dz}_{=\Phi(\mu/s)} + s\underbrace{\int_{-\mu/s}^\infty z\,\phi(z)\,dz}_{=\phi(\mu/s)}$$

The second integral uses $z\phi(z)=-\phi'(z)$, so $\int_{-\mu/s}^\infty z\phi(z)\,dz = \phi(\mu/s)$.

The option price is therefore:
$$\text{Price}_{\text{normal}} \approx e^{-rT}\bigl[\mu_D\,\Phi(\mu_D/\sigma_D) + \sigma_D\,\phi(\mu_D/\sigma_D)\bigr]$$

> **Simplifying assumption:** $D_n$ is treated as *exactly* normal — it is only approximately so by the CLT. The approximation improves as $n$ grows.

### Moment derivations under $Q$

**Step 1 — One-period gross-return moments.** Let $R_t = S_t/S_{t-1}$; under $Q$ each $R_t$ is i.i.d.:
$$m_1 = \mathbb{E}^Q[R] = qu + (1-q)d = e^{r\Delta t}, \qquad m_2 = \mathbb{E}^Q[R^2] = qu^2 + (1-q)d^2$$

**Step 2 — Marginal price moments.** By independence of returns:
$$\mathbb{E}^Q[S_t] = S_0\,m_1^t, \qquad \mathbb{E}^Q[S_t^2] = S_0^2\,m_2^t$$

**Step 3 — Cross-moment via tower property** ($s\le t$):
$$\mathbb{E}^Q[S_s S_t] = \mathbb{E}^Q\!\left[S_s\,\mathbb{E}^Q[S_t\mid\mathcal{F}_s]\right] = \mathbb{E}^Q\!\left[S_s^2\,m_1^{t-s}\right] = S_0^2\,m_2^s\,m_1^{t-s}$$
By symmetry: $\mathbb{E}^Q[S_s S_t] = S_0^2\,m_2^{\min(s,t)}\,m_1^{|t-s|}$.

**Step 4 — Covariance:**
$$\operatorname{Cov}(S_s, S_t) = S_0^2\!\left(m_2^{\min(s,t)}\,m_1^{|t-s|} - m_1^{s+t}\right)$$

**Step 5 — Mean of $D_n$:**
$$\mu_D = \mathbb{E}^Q[D_n] = \mathbb{E}^Q[S_n] - \frac{1}{n+1}\sum_{t=0}^n\mathbb{E}^Q[S_t] = S_0\,m_1^n - \frac{S_0}{n+1}\sum_{t=0}^n m_1^t$$

**Step 6 — Variance of $D_n$.** Writing $D_n = S_n - \tfrac{1}{n+1}\sum_t S_t$ and applying bilinearity of covariance:
$$\sigma_D^2 = \operatorname{Var}(S_n) - \frac{2}{n+1}\sum_{t=0}^n\operatorname{Cov}(S_n, S_t) + \frac{1}{(n+1)^2}\sum_{s=0}^n\sum_{t=0}^n\operatorname{Cov}(S_s, S_t)$$
The double sum is computed via a vectorised $(n+1)\times(n+1)$ covariance matrix using `np.meshgrid`.

## Cell 17 — code

```python
# --- One-period gross-return moments under Q ---
# E^Q[R]   = e^{r*dt}
# E^Q[R^2] = q*u^2 + (1-q)*d^2
# --- Marginal price moments E[S_t] for t = 0..n ---
# --- Build the (n+1) x (n+1) covariance matrix Cov(S_s, S_t) ---
# We use the formula  E[S_s * S_t] = S0^2 * m2^min(s,t) * m1^|s-t|
# and then  Cov(S_s, S_t) = E[S_s * S_t] - E[S_s] * E[S_t].
# --- Mean of D_n  =  E[S_n] - (1/(n+1)) * sum_t E[S_t] ---
# --- Variance of D_n via bilinearity of Cov ---
# sigma_D^2 = Var(S_n) - 2/(n+1) * sum_t Cov(S_n, S_t)
#                       + 1/(n+1)^2 * sum_s sum_t Cov(S_s, S_t)
# --- Closed-form normal-approximation price using the truncated normal identity ---
# E[max(X, 0)] = mu * Phi(mu/s) + s * phi(mu/s)   for X ~ N(mu, s^2)
```

## Cell 18 — code

```python
# Forward pass tracking the Q-probability of every reachable (j, C_t) state.
# prob_states[t][j] is a dict mapping C -> Q-probability of that state.
            # up-move
            # down-move
# Collect all terminal D_n values together with their Q-probabilities
# Sanity check: Q-probabilities must sum to 1
```

## Cell 19 — markdown

## Section 7 — Summary of Results
