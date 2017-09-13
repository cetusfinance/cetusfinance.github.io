---
title: Actual Analytic Jacobians
excerpt: "Implementing an analyic jacobian builder"
category: "gav"
header:
  overlay_image:  AnalyticSuperSpeed/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: AnalyticSuperSpeed/teaser.jpg
tags: [performance, .net, curves]
---
# The setup
##### *Context*

In a previoius [post](/gav/Jacobian-Juggling/), I covered a few ways to accelerate solving of multi-curve systems, one of which was to implement an analytic solution to generate the jacobian matrix used inside the solver (as opposed to using numerical estimation).  I this post, I will discuss the practical implementaion of this idea and give some hard benchmark numbers to demonstrate the impact.

##### *The problem*

Our curve engine solver fits the curves to the market instrument prices by using a [Gauss-Newton](https://en.wikipedia.org/wiki/Gauss%E2%80%93Newton_algorithm) solver - it tries to find the set of curves which give zero present value (PV) for each of the market instruments.  The algorithm itteratively improves the solution by taking the output from the current guess and applying a transform using the jacobian matrix and in the case of a truly linear problem can get to the answer in a single step.  In the case of our curve solving problem, it usually takes around five steps to get to the answer.

The slow part of the algorithm is generating the jacobian matrix.  If we use a numerical method to generate it, we will need to bump each point on the curve we are trying to solve and measure the change in each instrument we are trying to fit the curves to.  That, in turn, causes many (many, many) calls to a relativelty expensive interpolation function.

![chain](https://cetus.io/images/analyticsuperspeed/chain.jpg)
##### *The solution*

There is another way.  Rather than trying to numerically estimate the jacobian by bumping and calculating, we can turn our function to generate the PV of each instrument into a function to generate the sensitivity of the PV to bumps in our underlying curves. The key is to break the problem up and use the chain rule:

 $$\frac{dp}{dr}=\frac{dp}{dq}*\frac{dq}{dr}$$

 where
 
 $$p$$ is the PV of the instrument
 
 $$r$$ is a rate for a pillar date on the curve we are solving for 
 
 $$q$$ is a rate on a date which doesnt neccesarily fall on one of our curve pillars

 
So we split the problem up into expressing the sensitivity of the PV to the rates on the dates which are needed to price the swap along with the sensitivity of those rates to changes in the rates on the pillar dates of the curve we are trying to solve. The first part is a simple case of differentiating the PV with respect to the rate on each date needed to price the swap/FRA/depo/etc.  The second part comes from the interpolator used to turn the sparse set of rates-on-pillar-dates into a function to give a rate of any required date - for some simple interpolators, this can also be written down directly by differentiating the interpolation funciton but this part can be done numerically for more complex interpolation schemes.

If you look in the Qwack code base, you can see this in the Sensitivities method on the classes implementing the IFundingInstrument interface. On to the tangible stuff...

---
# Numbers 
##### Benchmarks 

The benchmark compares a simple curve build scenario - solving swap curves with OIS for two currencies - in three scenarios.  Firstly we just throw the solver at the problem.  Secondly we split up the problem into solve stages, where only curves which actually need to be solved simultaneously are done so. Both of the first two methods use a numerically-estimated jacobian in the solver.  The third test case uses an analytic jacobian as described above with the solve stages of the second test.

Tests are run on my i5-6200u laptop using .NET Core 2.0:

|Method|Mean(ms)|Error(ms)|StdDev(ms)|Scaled|Gen 0|Gen 1|Allocated|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|InitialOisAttempt|830.1|108.188|38.5820|1.00|1600.00|300.00|3.3 MB|
|Staged|244.3|16.410|  5.8522|   0.29 |   900.00 | 200.00 |   2.43 MB |
|StagedAnalyticJacobian| 106.3 |   2.030 |  0.7239 |   0.13 | 11500.00 | 800.00 |  19.62 MB |
 
So the using staging roughly cuts the time in three and the analytic jacobian basically halves it again.  I included the memory stats above to demonstrate the trade off - far more memory allocations and GC passes. 