---
title: Stripping Americans
excerpt: "Stripping implied vols from American option PVs"
category: "gav"
header:
  overlay_image:  Caddy.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: strippingAmericans/Teaser.jpg
tags: [performance, .net, options]
---

# Spot the difference
##### *Who's worth more..?*

Under Black-type dynamics, with everything else equal, an American option has to be worth more than (or equal to) the equivalent European.  This makes logical sense - one could take an American and choose not to exercise it until the last possible moment, making it exactly equivalent to a European.  The extra optionality (i.e. the possibility to exercise early if it suits the holder) can only add to the value.  If we then look at the inverse implication of this, for a given option premium, the implied vol for an American option must be less than (or equal to) the equivalent Black/European implied vol.

##### *Life at the edge*

The edge cases here are worth considering - when are American and European options worth exactly the same?  The first, trivial, case is where the option is so far out-of-the-money that it is worth zero.  The second case is where the discounting rate is zero - to explain this we need to consider the circumstances under which one would actualy exercise an American option early.  This article will focus on options-on-futures and the decision to exercise early depends on the type of margining applied.  

In the more common margined-as-options case (such as used on most CME listed American contracts), the decision to exercise comes as the loss in remaining time-value of the option becomes worth less than the interest which could be earned by excerising and putting the profit on deposit.  This is because to realize the profit from an margined-as-option trade, one must either sell or exercise the position - keeping a deep in-the-money position and not exercising it means forgoing any interest which could be earned on the intrinsic value (remember that futures have no, or very little, cost of carry).  

In the less common margined-as-futures case (such as ICE Brent or JSE-listed FX options), any PnL from moves in the option value are instantly/overnight realized as cash.  This means the reason to exercise a deep ITM position early vanishes as the profit can be put on deposit while the position still exists.  As such, there is never a reason to early exercise and the option can, in practice, be priced using the Black formula with zero discount rate. 

![Search](https://cetus.io/images/strippingAmericans/Search.jpg)

##### *Searching for a volatility*

So the case we need to consider is the margined-as-options one (as the other is trivial) - how do we turn an option premium into an implied volatility?  Unfortunately, just like in pricing the options, there is no closed form solution here so we have to use a solver with a potentially slow/expensive grid pricing method.  So we need to be smart and we can do that by employing [Brent's method](https://en.wikipedia.org/wiki/Brent%27s_method) with some prior knowledge about the bounds.  Remember how, for a given premium the implied vol from an American has to be strictly less-than-or-equal-to the implied vol from a European?  That forms our upper bound for the Brent algorithm and can be cheaply calculated in closed form.  The lower bound is harder but we can employ some empirical analysis and pick a spread from the upper bound which is likely to work (say 5% vol) and always itterate lower if the value turns out to be too high.

The use case for all of this tends to be taking a set of settlement premiums for listed options from an exchange and turning them into implied vols for passing into a model of some sort.  Its worth keeping in mind that exchanges don't always have the best quality data and some massaging of points will likely be needed to get a reasonable smile (like using only the out-of-the-money options) and choice of discount rate is often more art than science. Rejecting points which are worth exactly (or less than) intrinsic value is also key as no real implied vol can be found from these.  But it can be done and the more liquid the market, the closer the exchange settlement implied vols tend to be to the tradable market. 