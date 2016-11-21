---
title: Solving the problem
excerpt: "Post-2007 curve solving"
category: "gav"
header:
  overlay_image: solvingtheproblem/solvingtheproblem.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: solvingtheproblem/solvingtheproblem.jpg
tags: [fx, ir, solving]
---

## The problem

The problem we need to solve is to find a set of curves which reprice a set of market-quoted instruments, which means they all price to a PV of zero - the market price is seen as "fair" value and as such neither buyer nor seller would make or lose money when trading at that price.  The curves themselves are just wrapped interpolators where the x-axis is time (usually expressed as a year fraction under a specified day-count convention), the y-axis is some measure of interest rate and the type of interpolation is also a free choice.  As we will use a rate converter to facilitate any requests for discount factors or rates of different types from the curves, we are relatively agnostic to exactly what specification is chosen here so we will use log-linear interpolation in discount factor space. 

So we have a set of inputs, each of which serves as a constraint in our solving.  And we have a set of outputs - the values we want to solve for at points along each curve.  For our example, we will stick to the case of the number of inputs/constraints being exactly equal to the number of outputs, so it makes sense to assign one point on a curve to each input instrument.  A sensible choice for this is to use the expiry/maturity date of the instrument for this purpose.
There are many different types of solvers one could apply here so we will go with one of the oldest and simplest - Newton Raphson.  [Wikipedia] (https://en.wikipedia.org/wiki/Newton's_method) has a solid set of articles on the topic which I wont repeat here so I will focus on the version of the algorithm we will implement:

```
Guess = InitialGuess
PVs = PV(inputInstruments,Guess)
While(PVs.Abs.All(>tollerance)
{
    Jacobian = ComputeJacobian(Guess)
    Guess = UpdateGuess(Guess,Jacobian)
    PVs = PV(inputInstruments,Guess)
}
```

For the sake of keeping things simple in this example, we will compute the Jacobian numerically - that is, we will bump each input and measure the relative change in each output as so:

```
pvDiff = PV(inputInstruments[i],bumpedCurve[j]) - PV(inputInstruments[i],nonBumpedCurve)
Jacobian[i,j] = pvDiff / bumpSize
```

Where bumpedCurve[j] is the set of curves with only point j being bumped.

The final piece of the puzzel is the UpdateGuess step. This is the Newton Raphson part:

```
currentPVs = PV(inputInstruments,guess)
changeInGuess = Jacobian.Inverse() * currentPVs
guess = guess - changeInGuess 
```

So we run through the steps above, each time getting closer to our answer.  As with all solvers, a well posed problem and a well chosen starting point can help improve your chances of getting to a soloution quickly.  For rates, zero might be a simple enough point to start at but as we want to have something to discount on our floating rate legs then picking an arbitary non-zero value (say 1%) might reduce your itterations needed to solve.  As for a well-posed problem, just remember that picking randomized/arbitary values for your input instrument par-rates wont help your cause - try and work with real (or at least realistic) market data.

We now have a recipie for how to solve a set of curves so we need some objects to put this into practice... over to Tim...