# Model Card: Bayesian Optimisation for Black-Box Functions

This model card describes the optimisation model used in my black-box optimisation (BBO) capstone project: a Gaussian process (GP) surrogate combined with an acquisition function, used to choose where to query a set of hidden functions under a strict query budget.

## Model Description

**Input:** A vector of continuous values within a bounded domain (the candidate query point). The challenge functions range from 2-dimensional to 8-dimensional inputs; each function has its own surrogate model. For modelling, inputs are normalised to the unit hypercube.

**Output:** Two things, depending on how the model is used. (1) The GP surrogate outputs a *predicted function value* and an *uncertainty estimate* (predictive mean and standard deviation) for any candidate input. (2) The full optimisation pipeline outputs the *next query point(s)* to submit — the candidates that maximise the acquisition function.

**Model Architecture:** A Gaussian process regression surrogate with a Matérn 5/2 kernel (RBF compared as an alternative), with kernel hyperparameters (length-scales, signal variance, noise level) fitted each round by maximising the marginal log-likelihood on all queries observed so far. On top of the surrogate, an **Expected Improvement (EI)** acquisition function selects the next query, with **Upper Confidence Bound (UCB)** used in some rounds for comparison; the acquisition function is maximised by multi-start local optimisation over the input domain. For the higher-dimensional functions, a random forest surrogate was evaluated as an alternative, since GPs become harder to fit well as dimensionality grows relative to the number of observations. Initial rounds (before enough observations existed) used space-filling random/Sobol sampling instead of the model.

## Performance

Because the true optima of the black-box functions are hidden, performance is measured by proxies available to me:

- **Best-observed value per round** — the primary metric. For each function, I tracked the running maximum of all returned outputs across submission rounds. The improvement curve rises steeply once Bayesian optimisation replaces random sampling and then flattens as the search concentrates around the discovered optimum region, which is the expected signature of a well-behaved BO run. *[Insert your per-function best values / improvement plot here — a simple line chart of best-so-far vs round number per function works well.]*
- **Surrogate quality** — evaluated by leave-one-out cross-validation on the accumulated queries: how well the GP predicts each held-out observation from the others. Prediction errors shrank over rounds as data accumulated, and observed values fell within the GP's predictive uncertainty bands at roughly the expected rate, indicating the uncertainty estimates were well calibrated.
- All performance analysis is done on my own query logs (see the data sheet); no external test set exists by the nature of the problem.

## Limitations

- **Tiny-data regime.** With only tens of observations per function, the surrogate's picture of the landscape is coarse; it can (and early on, did) miss narrow peaks that fall between queried points.
- **Curse of dimensionality.** For the 6–8 dimensional functions, the same query budget covers the space exponentially more thinly, so the model is far less certain about having found a good optimum there than in 2–3 dimensions.
- **Local optima of the acquisition function.** The acquisition surface itself is multimodal; multi-start optimisation mitigates but does not guarantee that the globally best next query is chosen.
- **Assumption of smoothness.** The GP kernel assumes a certain smoothness of the hidden function. If a true function is highly rugged or discontinuous, the surrogate will be systematically over-smooth and its uncertainty estimates optimistic.
- **No ground truth.** Since the true optima and evaluation metrics are hidden, I can never verify how close the best-found values are to the actual maxima — only that they improved over time.

## Trade-offs

- **Exploration vs exploitation.** Aggressive exploitation finds good values faster but risks getting stuck near a local peak; heavy exploration wastes scarce queries on low-value regions. I traded these off by starting with high-exploration settings (space-filling sampling, then large UCB κ) and shifting towards exploitation in later rounds — accepting slower early progress in exchange for lower risk of missing the best region entirely.
- **GP vs simpler/scalable surrogates.** The GP gives principled uncertainty estimates, which are essential for good acquisition decisions, but its cost and reliability degrade in higher dimensions with few points. The random forest alternative scales better and handles rough functions, but its uncertainty estimates are cruder — on the low-dimensional functions the GP clearly made better use of each query.
- **Per-round batch size.** Submitting several queries per round gathers information faster but means later queries in a batch cannot benefit from the results of earlier ones. Given the fixed round schedule, I prioritised making each batch diverse (spreading batch points apart) at the cost of some per-point optimality.
- **Compute vs query efficiency.** Refitting the GP and optimising the acquisition function each round costs computation, but computation is cheap relative to queries in this setting — the binding constraint is the query budget, so the trade-off strongly favours spending compute to make every query count.
