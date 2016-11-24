---
title: "Instruments Galore"
excerpt: "Finally we can calculate some numbers!"
category: "tim"
header:
  overlay_image: https://cetus.io/images/instrumentsgalore/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: https://cetus.io/images/instrumentsgalore/teaser.jpg
tags: [fx, ir, solving, qwack]
---

# The building blocks for rates

If you haven't read them yet the following articles are worth reading before this,

1. [It all Started in 2007](/gav/It-all-started-in-2007/)
2. [Solving the Problem](/gav/Solving-the-problem/)
3. [Time ticks on](/tim/Time-ticks-on/) - not 100% required reading

Firstly we need a curve to solve, we only have one right now but more will definitely be added
so it calls for an interface

``` csharp
public interface ICurve
{
  DateTime BuildDate { get; }
  double GetDf(DateTime startDate, DateTime endDate);
  double GetForwardRate(DateTime startDate, DateTime endDate, RateType rateType, DayCountBasis basis);
  string Name { get; }
  ICurve SetRate(int pillarIx, double rate, bool mutate);
  int NumberOfPillars { get; }
  double GetRate(int pillarIx);
  IrCurve BumpRate(int pillarIx, double delta, bool mutate);
}
```

That should about cover it. The important part really is the mutate flag on the BumpRate and SetRate. 
For performance reasons during solving we will assume that we own the curves and that we can change them
as we need. Otherwise a lot of array copying would be required per guess while looping around. There might be
a need for copying the underlying data and then changing it at a later date so we have added a flag.

Now we can make a basic IrCurve object the constructor will take a list of pillars and rates as well as
a build date that should relate to the valuation date of our model. The curve also needs a name and a type.

``` csharp
public IrCurve(DateTime[] pillars, double[] rates, DateTime buildDate, string name, Interpolator1DType interpKind)
```

As explained in the earlier [article](/gav/Solving-the-problem/) we are only going to implement Linear interpolators,
then we will put in Log Linear, Cubic etc as we need them.

![Factory](https://cetus.io/images/instrumentsgalore/factory.jpg)

## Yes a factory

The Interpolators (all 1D at this point) are going to be put behind a factory. Normally I am the first to remove factories
from code. I find the pattern is often overused. In this case I think it's a good fit however. The reason, the implementations
of the interpolators will be heavily optimized based on the interpolator type, data set size and possibly even machine specifications.
But as always we want to give the user the ability to change this behavior. Initially it will be a static factory but I will change this
once we have settled down a little.

So what does our first cut of the factory look like?

``` csharp
public static IInterpolator1D GetInterpolator(double[] x, double[] y, Interpolator1DType kind, bool noCopy = false, bool isSorted = false)
{
    switch (kind)
    {
        case Interpolator1DType.LinearFlatExtrap:
            return new LinearInterpolatorFlatExtrap(x, y, noCopy, isSorted);
        default:
            throw new NotImplementedException();
    }
}
```

Once again you can see we have provided a toggle to allow us to reuse the input data in a potentially destructive/mutable fashion. The other useful
toggle is if the data is presorted. Often we will be processing something like curve data that is already in date order so there is no point spending time
trying to sort it again.

After a couple of versions of interpolations I quickly realised that we could move the copy/sorting up into the factory. It could have also been done in a base
class but I am avoiding virtual methods and this code is common. If the user wants to change behavior they can switch out the factory (once it isn't static).
So now our factory looks more like

``` csharp
if (!noCopy)
{
    var newx = new double[x.Length];
    var newy = new double[y.Length];
    Buffer.BlockCopy(x, 0, newx, 0, x.Length * 8);
    Buffer.BlockCopy(y, 0, newy, 0, y.Length * 8);
    x = newx;
    y = newy;
}
if (!isSorted)
{
    Array.Sort(x, y);
}
switch (kind)
{
    case Interpolator1DType.LinearFlatExtrap:
        return new LinearInterpolatorFlatExtrap(x, y);
}
```

Now we add a currency that contains the day count and settlement calendar. A float rate index which also is really just a container for settings with currencies,
day counts, roll types etc.

![Money](https://cetus.io/images/instrumentsgalore/funding.jpg)

## Show me the .. funding?

The basis of all financial products is a series of "flows". The most common of these are cashflows money being exchanged
at some point in time. Other types of flows can include things like other assets (physically delivered) such as stocks, bonds
or physical assets. Flows can also be be dependent on events or decisions such as barriers and options. In our current scenario
we only need simple cashflows so we will create our cashflow objects

``` csharp
public class CashFlow
{
    public DateTime AccrualPeriodStart { get; set; }
    public DateTime AccrualPeriodEnd { get; set; }
    public DateTime SettleDate { get; set; }
    public DateTime ResetDateStart { get; set; }
    public DateTime ResetDateEnd { get; set; }
    public DateTime FixingDateStart { get; set; }
    public DateTime FixingDateEnd { get; set; }
    public double Fv { get; set; }
    public double Pv { get; set; }
    public double Notional { get; set; }
    public double NotionalByYearFraction { get; set; }
    public Currency Currency { get; set; }
    public double FixedRateOrMargin { get; set; }
    public FlowType FlowType { get; set; }
}
```

Then we need a cashflow schedule, which is just a series of flows that will make up the legs of our instruments

``` csharp
public class CashFlowSchedule
{
  public List<CashFlow> Flows { get; set; }
  public DayCountBasis DayCountBasis { get; set; }
  public ResetType ResetType { get; set; }
  public AverageType AverageType { get; set; }
}
```

For the actual instruments we will define an interface they will all implement so that things like the solving engine
can be instrument agnostic

``` csharp
public interface IFundingInstrument
{
  double Pv(FundingModel model, bool updateState);
  CashFlowSchedule ExpectedCashFlows(FundingModel model);
}
```

