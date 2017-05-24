---
title: "Speeding up multi-linear regression"
excerpt: "An investigation into approaches in implementing multi-linear regression"
category: "gav+tim"
header:
  overlay_image: itallstarted/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: itallstarted/teaser.jpg
tags: [solving, algos, vectors]
---

# The context

> Regression - a measure of the relation between the mean value of one variable (e.g. output) and corresponding values of other variables (e.g. time and cost).

So what exactly is multi-linear regression when it's at home? As the definition above states, its about mapping the linear relationship between one output variable and several input variables. 

In mathematical terms, an output variable $$y$$ can be explained as a sum over input variables $$x$$, weighted by terms $$\beta$$:

$$y=\beta_{0}+\beta_{1}x_{1}+\beta_{2}x_{2}+...\beta_{n}x_{n}$$

$$y=\beta_{0}+\sum_{k=1}^n\beta_{k}x_{k}$$

When considering real data, for an observation $$i$$, we can write the relationship between the output variable $$y_{i}$$ and the inputs $$x_{ik}$$ by including an error term $$\epsilon_{i}$$:

$$y_{i}=\beta_{0}+\sum_{k=1}^n\beta_{k}x_{ik}+\epsilon_{i}$$

or in matrix form

$$y=\bf{X}\beta+\epsilon$$

where $$y$$, $$\beta$$ and $$\epsilon$$ are vectors and $$\bf{X}$$ is a matrix commonly refered to as the design matrix. Our task then is to find the vector $$\beta$$ which, for a given set of observations of both inputs and outputs $$y,\bf{X}$$, gives the smallest error vector $$\epsilon$$

# The setup

We're going to assume that the error terms $$\epsilon$$ are not correlated to the input variables $$\bf{X}$$.  This allows us to use the Ordinary Least Squares estimator which leads to a closed form solution for $$\beta$$:

$$\hat{\beta}=(\bf{X}^{T}\bf{X})^{-1}\bf{X}^{T}y$$

Our test will be to compare some implementations in widely-used .Net math libraries to our own version, implemented using some vectorized code for the matrix operations in the above formula.

# The test

...over to Tim??