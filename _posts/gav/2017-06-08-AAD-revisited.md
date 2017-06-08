---
title: AAD for option greeks - Part II
excerpt: "AAD compared to other methods for computing option greeks"
category: "gav"
header:
  overlay_image:  traderChart.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: PerformanceCounts/teaser.jpg
tags: [performance, .net, options, aad]
---

![Atlantic](https://cetus.io/images/aadthrowdown/atlantic.jpg)

# Crossing the pond
##### *Where we got to last time*

In my previous post on the [topic](https://cetus.io/gav/AAD-throwdown/), it seemed like AAD was far from a magic bullet to aid in speedy greeks calculations - the AAD method was too slow to be useful.  Having read some more around the topic, I saw a few authors pointing out that dropping AAD types in like-for-like with vanilla .NET types is rarely optimal and instead the code should be re-cast as a vector problem.  As I'm always keen for some vectorization, I decided to revisit...

##### *Vectors,Vectors,Vectors*

In this itteration, I've added my own simple SIMD-accelerated/vectorized version of the trinomial grid to the analysis and extended the AAD portion to use a similar vectorized form.  I also switched to using single-precision (float) numbers in the grid calculation after reading about the performance improvements it can offer in DiffSharp.  It was definitely a worthwhile exercise, though not without its drawbacks:

Times for computing PV and delta 100 times on my i7-3750 workstation, compiled and run in release mode:

|Method|Time (seconds)|
|:---:|:---:|
|Black-76|0.00243|
|Trinomial|10.99686|
|Trinomial-Vectorized|0.64945|
|Trinomial-DiffSharp|89.01838|
|Trinomial-DiffSharp-Vectorized|1.54425|

So quite the improvement, but there is an inevitable catch.  The vectorized form using DiffSharp can compute a PV but the AAD fails to produce a number - this is likely because the vectorized grid calculation requires three overlapping vectors.  This is easy to do with copying arrays or constructing .NET vectors but there is no 'native' DiffSharp operation for this and I couldn't come up with a method to perform the same with what was on offer.

Also worth commenting on is the effect of moving from double to single precision numbers in the calculations.  The option is a 5y 150 strike call (forward = 100, zero discounting rate, volatility = 16%), priced using 4000 grid steps.  The equivalent Black-76 PV (computed under double precision) is shown as the benchmark:

|Method|PV|Delta|
|:---:|:---:|:---:|
|Black-76|2.78437|0.169935|
|Trinomial|2.78477|0.168147|
|Trinomial-Vectorized|2.78467|0.168180|
|Trinomial-DiffSharp|2.78455|0.168137|
|Trinomial-DiffSharp-Vectorized|2.78455|N/A|

##### Some conclusions

After my initial disappointment, I'm glad I revisited DiffSharp and AAD.  It seems that, much like regular SIMD vectorization, AAD can offer significant benefits but requires a potentially major refactoring of code and will only be of use in certain situations.  I can see a regular monte-carlo simulation, rather than trinomial grid, offering an easier target for improvement and maybe that is something for me to look at in the future.  Also it seems like the debate on speed vs. accuracy for double-vs.-single precision in financial calculations is worth continuing as halving the variable size will produce gains of up to 50% in vectorized code, whether it relates to AAD or SIMD.

