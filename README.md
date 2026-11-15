# Portfolio Optimization with QAOA: From Penalty Constraints to XY Mixers

**BasQ Qiskit Fall Fest 2026 · Challenge 2 — Optimization / QAOA**

🏆 **First prize, BasQ Qiskit Fall Fest 2026 (Challenge 2)** — Team Cosine

Pick **exactly 3 of 8 assets** so that expected return is high and risk is low, using the Quantum Approximate Optimization Algorithm (QAOA). We start with the textbook penalty encoding, replace it with a constraint-preserving **XY mixer** that cannot produce invalid portfolios, and then test both against classical solvers, barren plateaus and a simulated noisy IBM Heron device.

> **Main finding.** On an ideal simulator, XY-QAOA returns a valid portfolio **100%** of the time and samples the exact optimum with probability **13.0%** (33× random), versus 61.9% valid and 6.0% optimal for the best penalty-QAOA setting (both at depth $p=5$). Classical methods still win easily at this size, and under simulated device noise the XY circuit's extra gates cancel its advantage.

## Contents

- [Team](#team)
- [The problem](#the-problem)
- [Method](#method)
- [Results](#results)
- [How to run](#how-to-run)
- [Repository layout](#repository-layout)
- [Evaluation methodology](#evaluation-methodology)
- [Next steps](#next-steps)
- [Acknowledgments and references](#acknowledgments-and-references)

## Team

| Member | Affiliation | Role |
|---|---|---|
| Shikha Chandel | North Carolina A&T State University | Code and experiments; appendix implementation |
| Dushyant Singh Pawar | University of North Carolina at Charlotte | Code and experiments |
| Yadav Ghimire | North Carolina A&T State University | Report and presentation |
| Bashu Dev Ghimire | North Carolina A&T State University | Report and presentation |

## The problem

We generate a synthetic, reproducible market with $n = 8$ assets: expected returns $\mu$ and a correlated covariance matrix $\Sigma$ from a two-factor model. Each portfolio is a bitstring $x \in \lbrace 0,1 \rbrace^n$, where $x_i = 1$ means asset $i$ is held. We minimize the mean–variance cost

$$
f(x) = q \, x^{\top} \Sigma \, x \;-\; \mu^{\top} x
$$

subject to the budget constraint

$$
\sum_{i=1}^{n} x_i = K, \qquad K = 3 .
$$

The risk aversion is $q = 1$ and the random seed is fixed. "Budget" here means an exact number of assets, not a dollar limit. Only $\binom{8}{3} = 56$ of the $2^8 = 256$ bitstrings are valid portfolios.

## Method

### QAOA in one line

QAOA prepares the state

$$
\lvert \psi(\gamma, \beta) \rangle = \prod_{k=1}^{p} e^{-i \beta_k H_M} \, e^{-i \gamma_k H_C} \; \lvert \psi_0 \rangle
$$

where $H_C$ is the cost Hamiltonian (diagonal, with $H_C \lvert x \rangle = C(x) \lvert x \rangle$), $H_M$ is the mixer, $p$ is the **depth** (number of layers), and the $2p$ angles $\gamma, \beta$ are tuned by a classical optimizer to minimize $\langle \psi \rvert H_C \lvert \psi \rangle$.

### Beginner: penalty QAOA

The constraint is moved into the cost as a penalty:

$$
C_{\lambda}(x) = f(x) + \lambda \left( \sum_{i} x_i - K \right)^{2}
$$

Substituting $x_i = (1 - Z_i)/2$ turns this QUBO into an Ising Hamiltonian with Pauli $Z_i$ and $Z_i Z_j$ terms. The circuit starts in $\lvert + \rangle^{\otimes n}$ (all 256 bitstrings) and uses the standard mixer $H_M = \sum_i X_i$.

**How large should $\lambda$ be?** The lowest-energy state is a valid portfolio exactly when

$$
\lambda \;\ge\; \lambda_{\mathrm{crit}} = \max_{x \text{ invalid}} \frac{f_{\mathrm{opt}} - f(x)}{\left( \lvert x \rvert - K \right)^{2}}
$$

Below $\lambda_{\mathrm{crit}}$, even a perfect solver returns an invalid portfolio. Far above it, the penalty drowns out the return/risk differences QAOA needs to see. We therefore sweep $\lambda$ in units of $\lambda_{\mathrm{crit}}$.

### Intermediate: constraint-preserving XY-QAOA

Instead of penalizing invalid portfolios, we make them impossible (Hadfield et al., 2019):

| | Penalty QAOA | XY-QAOA |
|---|---|---|
| Initial state | $\lvert + \rangle^{\otimes n}$: all 256 bitstrings | Dicke state $\lvert D^{n}_{K} \rangle$: equal superposition of the 56 valid portfolios |
| Mixer | $\sum_i X_i$ (changes the number of assets) | $\sum_{(i,j) \in \text{ring}} \tfrac{1}{2} \left( X_i X_j + Y_i Y_j \right)$ (swaps one asset in for one out) |
| Cost | $f(x) + \lambda \left( \sum_i x_i - K \right)^2$ | $f(x)$, so there is no $\lambda$ to tune |

The XY term only exchanges $\lvert 01 \rangle \leftrightarrow \lvert 10 \rangle$, so the number of held assets never changes. Together with a diagonal cost layer, this gives **P(valid) = 1 by construction** on an ideal device. The Dicke state is prepared with the deterministic $O(nK)$ circuit of Bärtschi & Eidenbenz (2019).

We also try a **CVaR** objective (Barkoutsos et al., 2020), which minimizes the mean of the best $\alpha = 10\%$ of outcomes instead of the full average.

### Advanced: trainability and noise

- **Barren plateaus.** For $n = 4, 6, \dots, 14$ with $K = n/2$, we measure the gradient variance $\mathrm{Var}_{\theta} \left[ \partial \langle C \rangle / \partial \gamma_1 \right]$ over 100 random angle settings, for both mixers at shallow ($p = 2$) and deep ($p = n$) depth.
- **Hardware noise.** Circuits are transpiled to `FakeFez`, a snapshot of a 156-qubit IBM Heron device (heavy-hex connectivity, measured error rates), and run with Qiskit Aer. We then apply readout mitigation and post-selection on valid portfolios.

## Results

The classical ground truth comes from enumerating all 256 bitstrings.

- **Exact optimal portfolio:** BETA, GAMA, THET
- **Optimal mean–variance cost:** ≈ −0.3323 (lower is better)

### Penalty QAOA: a trade-off, not a fix

![Penalty sweep: P(valid), P(optimal) and quality over depth and penalty strength](docs/images/heatmaps.png)

A larger $\lambda$ gives more valid portfolios but worse ones. The best P(optimal) sits near $\lambda \approx \lambda_{\mathrm{crit}}$, and even there about 40% of the probability is wasted on invalid portfolios.

### Penalty vs XY-QAOA (ideal simulator)

![P(valid), P(optimal) and quality vs depth for penalty QAOA, XY-QAOA and CVaR](docs/images/comparison.png)

Best depth by optimal-solution probability for each method:

| Method | Depth $p$ | P(valid) | P(optimal) | Quality among valid |
|---|---|---|---|---|
| Penalty QAOA ($\lambda = \lambda_{\mathrm{crit}}$) | 5 | 61.9% | 6.0% | 0.827 |
| Penalty QAOA ($\lambda = 10 \lambda_{\mathrm{crit}}$) | 5 | 87.5% | 1.8% | 0.644 |
| **XY-QAOA (Dicke + XY mixer)** | 5 | **100.0%** | **13.0%** | **0.855** |
| XY-QAOA + CVaR(0.1) | 3 | 100.0% | 10.1% | 0.765 |

For reference, a random 8-bit string matches the optimum with probability 1/256 ≈ 0.39%, and a random valid portfolio with 1/56 ≈ 1.79%.

### Classical baselines

| Method | Result |
|---|---|
| Exact enumeration | Finds the optimum in about 2.4 ms |
| Greedy | Also finds the optimum on this instance |
| Simulated annealing (50 swap steps) | Finds the optimum in 95.8% of 500 seeded runs |

There is **no quantum advantage** here, and we don't claim one. The contribution is an empirical comparison of constraint encodings, solution quality, circuit cost and noise sensitivity. Interestingly, annealing's best move (swap one asset in, one out) is the classical twin of the XY mixer.

### Gate cost

The XY design is not free. After decomposition to CX gates, XY-QAOA needs **210 CX** at $p = 1$ versus **56 CX** for penalty QAOA, and 498 versus 280 at $p = 5$. The Dicke state alone costs 138 CX. Even at equal gate budget XY still wins on the ideal simulator: XY at $p = 1$ (210 CX) roughly matches penalty QAOA at $p = 5$ (280 CX) in P(optimal), with every sample valid.

### Barren plateaus

![Gradient variance vs number of qubits for X and XY mixers at shallow and deep depth](docs/images/barren_plateaus.png)

Deep circuits ($p = n$) of **both** kinds show exponentially vanishing gradients: the variance shrinks about 1.6× (X mixer) and 1.9× (XY mixer) per extra qubit. Staying inside the valid subspace does not help, because $\binom{n}{n/2}$ still grows exponentially. Shallow circuits with smart initialization (ramp + INTERP) keep usable gradients.

### Noise on a simulated IBM Heron device

![P(valid) and P(optimal | valid) under FakeFez noise, and P(optimal | valid) vs CZ gate count](docs/images/noise.png)

- **Routing roughly doubles the cost:** heavy-hex connectivity takes penalty QAOA from 56 to 92 two-qubit gates at $p = 1$, and XY-QAOA from 210 to 345 (about 1000 at $p = 5$).
- **Noise breaks XY's guarantee:** P(valid) falls from 1.00 to 0.47 at $p = 1$ and to about 0.28 at $p = 5$. Every invalid sample is a *detected* error, so post-selection removes it, at the cost of discarding up to ~70% of shots.
- **The ideal winner is not the hardware winner:** after post-selection the best noisy results are a tie at about 5.3% optimal: penalty QAOA at $p = 5$ versus XY-QAOA at only $p = 2$. Deeper XY circuits get worse under noise.
- **Readout mitigation barely helps** (≤ 1 percentage point): gate errors dominate.

**On hardware, the currency is two-qubit gates, not layers.** This runs on a simulator with a device-derived noise model, not on a real quantum processor.

### Appendix: independent cross-check

A separate implementation on a different instance (5 assets, pick 3; hand-built gates, no shared code) reproduces the main finding: XY-QAOA is 100% valid and samples the optimum 3–3.5× more often than a random valid portfolio, while penalty QAOA with a too-large penalty is valid but no better than random.

## How to run

Python 3.12 was used for the saved run, with Qiskit 2.x.

1. Clone this repository and open `challenge_2_portfolio_final.ipynb` in JupyterLab, VS Code or qBraid.
2. Install the dependencies in the same environment as your notebook kernel:
   ```bash
   python -m pip install -r requirements.txt
   ```
   Or uncomment and run the notebook's first `%pip install` cell.
3. Restart the kernel and **Run All** in order. Later sections reuse variables from earlier ones, so running cells out of order can cause `NameError` or stale results.
4. Plots and tables are written to `figures/` and `results/`.

A full run takes about 10 minutes on qBraid; the penalty sweep and the barren-plateau experiment take the longest.

## Repository layout

```
.
├── challenge_2_portfolio_final.ipynb  # Main notebook, as submitted (Beginner → Advanced + appendix)
├── appendix_5asset_qaoa_shikha.ipynb  # BasQ Max-Cut QAOA tutorial + Shikha's independent 5-asset implementation
├── TeamCosine_Final_No_Notebook.pdf   # Written report (as submitted)
├── docs/
│   ├── presentation.pptx              # Presentation deck
│   ├── study_deck.pdf                 # Team study / Q&A prep deck
│   └── images/                        # Plots used in this README (from the saved notebook run)
├── requirements.txt
├── LICENSE
└── README.md
```

Running the notebook also creates `figures/` and `results/` (CSV tables such as `results/intermediate_summary.csv` and `results/classical_baselines.csv`). These are generated outputs and are not checked in.

## Evaluation methodology

All metrics use the full probability distribution of the final QAOA state, not just the best sample seen. With 256 bitstrings and thousands of shots, "best sample found" would read 100% even for a bad circuit.

- **P(valid):** probability of measuring exactly $K = 3$ assets.
- **P(optimal):** probability of measuring the globally optimal portfolio.
- **Quality:** among valid outcomes, how close the average portfolio is to the best one (1 = always optimal, 0 = always worst).
- **Correctness checks:** the Hamiltonian diagonal matches the brute-force cost for all 256 bitstrings; the fast simulators match Qiskit's statevector to about $10^{-16}$; the Dicke state has fidelity 1.0; sampled frequencies match predicted probabilities.

The asset data are synthetic and for education only. They are not investment advice.

## Next steps

- A hardware-native XY ring that follows physically connected qubits, so the mixer needs no SWAPs.
- A cheaper Dicke-state preparation, or a warm start from a single good valid portfolio.
- Choosing $p$ by the noise budget rather than by the ideal simulator.
- Larger instances against stronger classical heuristics, and mitigation tested on real hardware.

## Acknowledgments and references

This project builds on B. Tirado's *Combinatorial Optimization with QAOA: Max-Cut* tutorial for BasQ Qiskit Fall Fest 2026 (tutorial text CC BY 4.0, code Apache 2.0).

**References**
- H. Markowitz, *Portfolio Selection*, J. Finance (1952).
- E. Farhi, J. Goldstone, S. Gutmann, *A Quantum Approximate Optimization Algorithm*, [arXiv:1411.4028](https://arxiv.org/abs/1411.4028) (2014).
- S. Hadfield et al., *From the Quantum Approximate Optimization Algorithm to a Quantum Alternating Operator Ansatz*, Algorithms 12, 34 (2019).
- A. Bärtschi, S. Eidenbenz, *Deterministic Preparation of Dicke States*, FCT (2019).
- L. Zhou et al., *Quantum Approximate Optimization Algorithm: Performance, Mechanism, and Implementation on Near-Term Devices*, Phys. Rev. X 10, 021067 (2020).
- P. Barkoutsos et al., *Improving Variational Quantum Optimization using CVaR*, Quantum 4, 256 (2020).
- J. McClean et al., *Barren plateaus in quantum neural network training landscapes*, Nat. Commun. 9, 4812 (2018).
- M. Larocca et al., *Diagnosing Barren Plateaus*, Quantum 6, 824 (2022); M. Ragone et al., *A Lie algebraic theory of barren plateaus*, Nat. Commun. 15, 7172 (2024).

---

*Educational quantum optimization experiment · BasQ Qiskit Fall Fest 2026, Challenge 2.*
