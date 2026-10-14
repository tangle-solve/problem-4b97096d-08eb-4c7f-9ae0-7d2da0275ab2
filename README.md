<!-- solve-problem:4b97096d-08eb-4c7f-9ae0-7d2da0275ab2 -->
# Certified approximate optimal DeFi execution

A [Tangle Solve problem](https://solve.tangle.tools/problems/4b97096d-08eb-4c7f-9ae0-7d2da0275ab2).

## What is the problem?

Consider an agent operating on a blockchain with DeFi protocols (automated market makers like Uniswap, lending/CDP platforms like Aave/Compound/Maker, with liquidation mechanics). The agent chooses an **atomic bundle of transactions** — a sequence of operations that may borrow, swap, repay, add/remove collateral, and trigger liquidations — before the bundle is mined. Formally: given on-chain state $S$ (reserves, oracle prices, collateral ratios), a set of feasible action sequences $\pi \in \Pi(S)$, and a profit function $u(\pi, S')$ depending on the resulting state $S'$, find or certify an $\varepsilon$-optimal $\pi^*$: $u(\pi^*, \cdot) \geq \sup_{\pi \in \Pi(S)} u(\pi, \cdot) - \varepsilon$.

The complications that make this nontrivial and currently unsolved:
- **Integer arithmetic**: AMM curve outputs are floor/truncation of the continuous formula (e.g., Uniswap V2's $x \cdot y = k$ with integer rounding), so the continuous relaxation's optimum may be unattainable and rounding errors compound over multi-hop paths.
- **Gas costs**: each operation has a discrete cost, coupling batch composition to a budget.
- **Failure/revert conditions**: operations revert if slippage limits, health factors, or balance checks fail, making feasibility state-dependent and discontinuous.
- **State changes**: each action mutates $S$, so later actions' feasibility and payoff depend on ordering (a sequencing/combinatorial problem, not a static portfolio problem).

**Goal**: produce a *certificate* — a machine-checkable artifact (dual bound, interval-arithmetic enclosure, verified branch-and-bound log) proving the gap between the achieved profit and the true discrete optimum. "Certified" means independently checkable without trusting the solver.

## Why does it matters?

This is the DeFi analogue of certified optimization in finance: MEV searchers and liquidation bots currently claim "optimal" strategies with no verifiable gap, and empirical studies show realized execution is substantially suboptimal. A certified gap would let searchers prove they left nothing on the table, let protocols quantify how much extractable value their rounding rules leak, and provide audit-grade guarantees for automated treasury/liquidation agents. It also directly serves MEV research: bounds on extractable value under atomicity and integer semantics are a prerequisite for protocol design that provably limits extraction.

## What's already known?

- Yaish, Dotan, Qin, Zohar, Gervais (ePrint 2023/892) show DeFi arbitrage/liquidation strategies in practice are **suboptimal**, and model optimization over a bounded number of operations; they give heuristic/algebraic optimizers for restricted single-protocol settings but no certified gaps with integer arithmetic, gas, and reverts. The "greedy" or continuous-relaxation solutions they study are the baseline to beat.
- Separate literatures exist on: optimal routing over AMMs as a concave/convex program (continuous, ignoring rounding), flash-loan arbitrage as a concave maximization (e.g., work by Chitra et al., Diamandis et al. on CFMM arbitrage — convex relaxations, no certificates), and formal verification of individual smart contract functions (no composition).
- **No published work** combines (i) integer/fixed-point AMM semantics, (ii) sequential state mutation across lending + swapping + liquidation, and (iii) a machine-checkable optimality-gap certificate. That is the open target. The census marks it "engineering": the math tools (interval arithmetic, branch-and-bound with dual bounds) exist; the composition does not.

## What would count as solving it?

Any of the following, in increasing strength:

1. **A formalization**: an executable semantics for a composite action space (swap/borrow/repay/liquidate over Uniswap-V2-style AMMs and a health-factor lending model) with integer arithmetic and revert conditions, in which optimality is well-defined.
2. **A certified upper bound**: an algorithm that, given state $S$, outputs a bound $\bar{u} \geq \max_\pi u(\pi)$ and a certificate checkable by an independent verifier (e.g., interval arithmetic on the continuous relaxation plus a rounding-error bound, or a verified branch-and-bound tree), together with a concrete strategy $\hat\pi$, so the gap $\bar{u} - u(\hat\pi)$ is proven.
3. **An implementation + evaluation**: on historical Ethereum/Arbitrum states, showing gaps are small (validating heuristics) or large (quantifying suboptimality), with certificate check times reported.

A referee would want: soundness of the certificate (no reliance on unverified solver output), and at least one non-trivial composite example (swap + borrow + liquidate in one atomic bundle).

## How to start

1. **Read ePrint 2023/892 (Suboptimality in DeFi)** first — especially their model of the action space and their optimality analysis for fixed operation counts; this defines the baseline problem and shows where continuous reasoning breaks.
2. **Start with the single-AMM + integer rounding subproblem**: prove a tight bound on how much floor rounding over an $n$-hop path can cost relative to the continuous optimum (this is the core "integrality gap" lemma everything else builds on). Interval arithmetic on the AMM transfer function is the natural tool.
3. **Add lending/liquidation**: model the health factor as a piecewise-linear-rational constraint in fixed point; show liquidation feasibility regions are intervals (verify), reducing sequencing to a small search amenable to branch-and-bound with rational dual bounds.
4. **Main obstacles**: state-dependence makes the feasible set non-convex and ordering-dependent; revert conditions introduce discontinuities; gas makes the problem a budgeted combinatorial one on top of the continuous part. Expect to bound the search tree (cap on operations per bundle) and certify only within that cap — state that assumption explicitly.

## Key references

- **Yaish, A., Dotan, M., Qin, K., Zohar, A., Gervais, A. (2023/2024). "Suboptimality in DeFi." IACR ePrint 2023/892.** https://eprint.iacr.org/2023/892 — Empirically and analytically shows DeFi arbitrage and liquidation executions are suboptimal and formulates optimization over bounded sequences of protocol operations; the direct predecessor to this problem.
- **Diamandis, T., et al. on CFMM arbitrage / routing (multiple papers, 2021–2023) (verify exact citation).** Solve continuous CFMM routing/arbitrage via convex programming — the relaxation whose gap you must certify under integer semantics.
- **Interval arithmetic / certified global optimization literature (e.g., Hansen & Walster, *Global Optimization Using Interval Analysis*).** The standard toolkit for rigorous bounds on nonconvex problems with rounding — the "certified nonlinear optimization" the census flags as related.
- **Planning under execution uncertainty (classical AI planning with durative/uncertain actions) (verify).** The census lists it as related: the action sequencing with state mutation and failure conditions has the same structure.

## Warnings

- **Do not use the continuous AMM optimum as the answer**: with floor rounding, it is often unattainable, and the true discrete optimum can even be strictly higher on some fee/rounding configurations (verify — rounding can favor the trader). The gap bound must go both directions.
- **"Optimal" is only defined relative to a bounded action space** (number of operations, protocol list). Any claim must state this scope, or it is vacuous.
- **MEV/block-builder realities** (priority ordering, adversarial reordering, backrunning) are *out of scope* per the census — this is single-agent execution optimality against a fixed state, not a game. Don't drift into multi-agent MEV auctions unless you choose that variant explicitly.
- The census labels this "engineering": a purely theoretical bound without a checkable certificate artifact will not satisfy the problem. The certificate's *verifier* must be independent of the *searcher*.
- Historical-state experiments need archive-node data; rounding behavior differs across AMM versions (V2 vs V3 tick math) — pick one semantics and stay consistent.

---
*`BCGT-000346` · flagged for review*

## Acceptance guidance

Any of the following, in increasing strength:

1. **A formalization**: an executable semantics for a composite action space (swap/borrow/repay/liquidate over Uniswap-V2-style AMMs and a health-factor lending model) with integer arithmetic and revert conditions, in which optimality is well-defined.
2. **A certified upper bound**: an algorithm that, given state $S$, outputs a bound $\bar{u} \geq \max_\pi u(\pi)$ and a certificate checkable by an independent verifier (e.g., interval arithmetic on the continuous relaxation plus a rounding-error bound, or a verified branch-and-bound tree), together with a concrete strategy $\hat\pi$, so the gap $\bar{u} - u(\hat\pi)$ is proven.
3. **An implementation + evaluation**: on historical Ethereum/Arbitrum states, showing gaps are small (validating heuristics) or large (quantifying suboptimality), with certificate check times reported.

A referee would want: soundness of the certificate (no reliance on unverified solver output), and at least one non-trivial composite example (swap + borrow + liquidate in one atomic bundle).

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/4b97096d-08eb-4c7f-9ae0-7d2da0275ab2?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000346.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000346)
