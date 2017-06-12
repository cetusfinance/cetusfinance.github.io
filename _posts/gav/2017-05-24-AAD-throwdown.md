---
title: AAD for option greeks
excerpt: "AAD compared to other methods for computing option greeks"
category: "gav"
header:
  overlay_image:  traderChart.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: AADThrowdown/greece.jpg
tags: [performance, .net, options, aad]
---

![Atlantic](https://cetus.io/images/aadthrowdown/atlantic.jpg)

# Crossing the pond
##### *A quick intro to American vs. European options*

To narrow the scope of this article, we're going to focus on options-on-futures of the kind one can trade on derivative exchanges around the world ([CME](https://www.cmegroup.com), [ICE](https://www.theice.com), etc).  The two predominantly traded option types are known as American and European options - the names have nothing to do with the geographic location of the exchange or the underlying contract but rather distinguish when the holder of an option is allowed to exercise it.  European options only allow exercise on the expiry date of the contract, American options allow exercise at any point up to (and including) the expiry date. 

##### *Mr Black*

The big difference between European and American options however comes in how you can price and risk them - values for European options-on-futures can be found using the closed-form [Black-76](https://en.wikipedia.org/wiki/Black_model) formula (including greeks) where as American options require the use of a grid or PDE method for pricing (with greeks computed via bump-and-reval). The practical distinction here is one of speed - for options with several years to expiry, the Black formula can be several orders of magnitude quicker in producing prices and greeks compared to a grid method.

---
# The setup

Grid pricing isn't only for American options - one can price a European option in the same way if you have time to kill (or are trying to produce a test such as this one!).  The test here will be to price and produce greeks (delta and vega) for a European option, comparing the Black formula for a benchmark (and to verify values produced) to a regular trinomial grid model and the same grid model implemented using an AAD library.

I've chosen diffSharp as the AAD library for ease of implementation.  There are others we could look at but as a first step I'm attempting to get a ball-park comparison as to how AAD compares to other methods.  The code for the trinomial grid pricing algorithim can be found in the Qwack library, as can our implementation of Black-76.

![Raw meat](https://cetus.io/images/aadthrowdown/raw.jpg)
##### Raw results

Times for computing PV 100 times on my i5-6200u laptop, compiled and run in release mode:

|Method|Time (seconds)|
|:---:|:---:|
|Black-76|0.00131|
|Trinomial|5.99087|
|Trinomial-DiffSharp|40.68622|

As expected, the closed-form Black formula is orders of magnitude quicker than the grid methods.  The take-away here though is that the diff-sharp solution is dramatically slower than the vanilla trinomial grid implementation, so much so that even with zero overhead in computing greeks, the AAD solution would still be slower than bump-and-reval.

For a combination of PV, delta and vega (now only 10 simulations):

|Method|Time (seconds)|
|:---:|:---:|
|Black-76|0.00689|
|Trinomial|2.32571|
|Trinomial-DiffSharp|36.5152|

The vanilla trinomial result above scales as expected - the PV plus two greeks require four calculations of PV. What is interesting to me is that the AAD verison performs even worse under the test where I expected to see the benfits - and to make it worse, the value produced by my attempt at an AAD vega calculation is clearly nonsense (thought delta and PV match closely to Black).  I attempted a few other approaches with DiffSharp, such as using the gradient (vector-to-scalar) differentiation method rather than the simple scalar-to-scalar form but nothing produced any quicker results.

##### Some conclusions

My first foray into AAD was disappointing to say the least.  It could be my implementation but I doubt that could account for such a massive speed discrepancy.  From here I see two other tests I want to make - trying another .NET open-source AAD library and stacking that up against a vectorized version of the trinomial grid code.  Watch this space for more on using vectors to speed up our opensource finance library.... 

Test code can be found on a branch of Qwack, [HERE](https://github.com/cetusfinance/qwack/tree/AmericanAAD)