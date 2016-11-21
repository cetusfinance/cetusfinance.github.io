---
title: It all started in 2007
excerpt: "When the world become so much more complex"
category: "gav"
header:
  overlay_image: itallstarted/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: itallstarted/teaser.jpg
tags: [fx, ir, solving]
---

# The multi-curve interest rates model and how it came to be…

Prior to 2007, the world of quant finance relied upon an assumption which made everyone’s lives much easier – each currency has a single interest rate curve 
which could be used for forecasting and discounting any rates or cashflows in that currency. 
This wasn’t just some ill thought-out theory, it was validated by many years of practice and the problems contained in the
assumption only became obvious as 2007 drew to a close.

The one curve model assumes that you can have a single curve for a currency (say USD) and that rates for various tenors of index (say [LIBOR](https://en.wikipedia.org/wiki/Libor) 3-monh and LIBOR 6-month) 
can be queried from the same curve.  This means you can use a mixture of instruments referencing different tenors to build the curve and as there is only a single curve to fit, 
front-to-back bootstrapping can be used which is both well-behaved and very fast.

![Mind the gap](/images/itallstarted/gap.jpg)

## Mind the gap

What became obvious, however, as the financial crisis wore on, is that that a material basis (or spread) existed and persisted for rates of different tenors.  This meant that the market saw 
a different total outcome between two three month deposits running sequentially and a single six month deposit, or the average daily overnight rate for three months being different to the three month spot rate.
There are many possible explanations for this but at the time, with banks floundering and failing all around, the idea that a deposit with a 
longer maturity required a higher rate to compensate an investor due to increased credit and liquidity risk, seemed reasonable.

>When the music stops in terms of liquidity, things will get complicated. But as long as the music is playing, you've got to get up and dance. We're still dancing. - Chuck Prince, Citigroup

So what challenges did this introduce?  Firstly we needed to account for this basis with a more complex setup. 
A book of mixed USD interest rate derivatives now needed separate curves for the various LIBOR tenors (1m, 3m, 6m and 12m at least) and more than 
likely a curve for discounting under a standard [CSA](https://en.wikipedia.org/wiki/Credit_Support_Annex) (a curve for the Federal Funds effective rate). 

So that’s at least five times the work but really not so hard, right?
Well it would be if the market quoted and traded fixed-for-float swaps for each tenor/curve, but they don’t.
But that’s also relatively easy to overcome if you solve for the 3m curve first (where fixed-for-float swaps and other instruments which uniquely 
define the curve exist) then use the result to solve for the other curves where the benchmark instruments are basis swaps for given LIBOR tenor againt 3m LIBOR. 
So then we just solve for the 3m curve like we did before and it all falls into place?

![Cause and Effect](/images/itallstarted/balls.jpg)

## Multiple factors to consider

This brings us to the third challenge the new world of multi-curve rates introduced – the need to solve for more than one curve simultaneously.
This comes from correctly modeling an interest rate swap where the contract references 3m LIBOR (for example) for its floating leg but, due to
the trade being executed under a market standard agreement between two banks, any cash flows need to be discounted on a curve consistent with
the terms of a standard CSA (in the case of USD this means using a FedFunds curve).  

![Magnetic](/images/itallstarted/magnet.jpg)

## Weak links can be broken

The reason the link between the two curves cannot be broken or mitigated by solving in stages comes from the swaps against LIBOR 
3m needing the FedFunds curve for discounting and the FedFunds curve being constructed from float-float basis swaps against 
3m LIBOR, forming a circular reference.

Computationally, this has one significant side-effect – one can no longer use simple front-to-back bootstrapping to get from the 
quoted instrument prices to the yield curves we seek.

![Lines](/images/itallstarted/tackle.jpg)

## Tackling the problem

There are many ways to try and tackle this third problem without actually attempting to do simultaneous solving.
For instance, if your float-float 
swaps have the same payment schedule as your fixed-float swaps, one could reasonably transform into a pair of fixed-float swaps and make the problem much easier to solve. 
But this rarely works due to market conventions not complying (and for some reason the markets don’t seem to want to change to make quant’s lives easier) so it's a tool for a 
few edge cases at best.  

Another approach is to attempt iterative solving of a pair of curves – solve for one curve first, keeping the other constant, then switch and 
solve for the other curve keeping the first constant.  This is repeated until all instruments re-price to zero PV and under many circumstances will produce valid results. 
The two main drawbacks are speed (it doesn’t tend to converge quickly) and stability (not all sets of real market quotes will converge, especially when using non-local 
interpolation methods).

So we need a way to solve multiple curves (as many as 3 at the same time for GBP where SONIA, 3m LIBOR and 6m LIBOR are all inter-linked) at the same time. Though that 
may seem like a complex task, it turns out that applying some relatively simple solving techniques can produce stable and relatively performant results with relatively 
simple code.    
