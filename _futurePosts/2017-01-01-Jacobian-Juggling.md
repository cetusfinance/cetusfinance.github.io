---
title: Shortcuts to a faster curve engine build
excerpt: "Juggling Jacobians to speed things up"
category: "gav"
header:
  overlay_image:  PerformanceCounts/header.jpeg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: PerformanceCounts/teaser.jpg
tags: [performance, .net, curves]
---
# Divide and Conquer
##### *Solving smaller problems speeds things up*


Let's say we want to setup some curves to price USD/ZAR FX trades. That will require us to fit to cross-currency swaps which will depend on USD LIBOR and ZAR JIBAR indices. And forecasts for those indices will be fitted to vanilla IR swaps which rely on OIS discount curves for their pricing. So we need to solve five curves - two for each currency separately (forecast and OIS discounting) and one for the cross-currency basis. Now let's say each curve has 20 benchmark instruments on it which leads to 100 benchmarks in total. Computing a 100x100 Jacobian matrix will be slow.  Thankfully we don't actually have to do that - we can solve the USD and ZAR problems first (and separately) which are both 40x40 matrices followed by using the output of the first stage to solve for the cross currency curve as a final step (merely a 20x20 matrix there). Not only is it much quicker due to removing unnecessary Jacobian calculations (10k vs. 3.6k elements per pass) but solving in stages also should improve convergence speed for the cases where a curve depends on another which can be pre-solved.

---
# Be choosy 
##### Selective update of discount or forecast curves 

This optimization can be applied when computing the Jacobian as we know which curve we are bumping at any stage and we can make the instrument store it's latest "state" internally. The idea is to re-use the discount factors (implied from PV divided by FV for each flow) if we are only bumping the forecast curve or simply re-discount the FV into PV if we are bumping only the discount factor. This saves a lot of unnecessary interpolation calls and can have a relatively dramatic effect for a simple code change (if you can live with the immutability of the objects being violated)

Another idea is to only update points you see sensitivity to - Starting point is everything here and the only thing to remember is not to start with rate exactly equal to zero as a flow with a forecast rate of exactly a zero will have no discounting sensitivity. This ensures any point with real sensitivity should have a non-zero value computed on the first pass. The trick is then on subsequent passes to only re-compute Jacobian elements where the values is not exactly equal to zero, which should stop you bumping a 30y swap for the 3m deposit. Another idea might be to enforce a locality constraint and only bump points within a certain distance on the maturity axis from your instrument expiry - results may vary depending on the kind of interpolator chosen.

---

# Juggling that Jacobian
##### Parallelization as a start, analytic solutions even better

Solving the Jacobian can be quite easily parallelized if one can clone and bump curves such that all bumped scenarios can be computed at the same time. One good way of doing this is to clone the whole curve set as a shallow copy and then just replace the curve containing the bumped point with a new, deep copied object. 

A better way to optimize this is to have an analytic Jacobian calculation. It should be relatively easy to write down the analytic derivative of a swap PV to a change in the rates to each reset and payment date. The problem is that we need sensitivity to bumping the pillars on the curves we are fitting rather than sensitivity to all the dates in the swap. Thankfully we can just apply the chain rule to solve this problem and use the interpolator definition to either write down the derivative for rates at each date in the swap with respect to bumping each pillar or, in the case of a more complex interpolator, compute them numerically.
