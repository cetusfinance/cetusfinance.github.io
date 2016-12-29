---
title: Performance counts
excerpt: "Getting some useful code in there, but is it good enough?"
category: "tim"
header:
  overlay_image:  PerformanceCounts/header.jpeg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: PerformanceCounts/teaser.jpg
tags: [testing, interpolation, testing, vector, performance, .net]
---

We have finally come to crunch time. Today we are going to put together a basic 1d interpolator and talk through how
we are going to test it for performance (unit tests are a given and will be put in, you can go read them if you are interested).

The first thing to do is write the simplest naive method for producing a linear interpolator. We provide a constructor that takes
two arrays of doubles, the x and the y values. We also request a flag to denote if we can just reuse those arrays or if the caller
might still be using those arrays and doesn't want us to modify them. Finally is a flag to tell us if we should sort the two arrays
or if the data is presorted. This is really just a shortcut because we know that down the line we will have sorted data so why waste
the cycles checking/resorting.

Once we have copied (or not) the arrays and done any sorting we calculate the slopes between each of our x points with

``` csharp
var slopes = new double[_xValues.Length - 1];
for (int i = 0; i < slopes.Length; i++)
{
  slopes[i] = (y[i + 1] - y[i]) / (x[i + 1] - x[i]);
}
```

This precalculation is used under the assumption that we are going to do more interpolations than the number of pillars. I think
this is a pretty reasonable assumption considering most use cases in finance.

This leaves the actual interpolation. Because we have the slopes we need to find the index of the highest pillar that is less than
the value we are interpolating (unless that is < the smallest pillar in which case we just want that). Something like this will do

``` csharp
var index = Array.BinarySearch(_xValues, x);
if (index < 0)
{
  index = ~index - 1;
}
return Min(Max(index, 0), _xValues.Length - 2);
```

This uses the binary search function which is the standard procedure in this case. And finally we have the calculation of the value

``` csharp
public double Interpolate(double x)
{
  var k = LeftSegmentIndex(x);
  return _yValues[k] + (x - _xValues[k]) * _slopes[k];
}
```

So thats that right? Not yet! This method is pretty good in the general case. And for a generalized math library you might just stop
there. The code is simple, easy to follow and maintain and performs well in most situations. However we aren't writting a general 
math library but one focused on finance and therefore we can make certain assumptions about our use cases in the future. The 
assumptions I will be making are

* We will be interpolating in tight loops and it will be a hot path (vols during monte carlo, curve solving etc)
* Our number of pillars will be relatively low (from looking at our scenarios 10-20 is pretty "normal")
* The machines we will be running on are all made in the last 5-10 years, and we aren't running on phones :)

So with that in mind we can go about trying our hand at tuning this simple 1d linear interpolator, but before that we need to prove
our new methods are better/worse...

# Benchmarking!

There is a famous quote that does the rounds that goes something like
> We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil
This might very well be true (I don't think so), but even if it is, then this counts as one of the 3% of the time!

We are going to solve this by setting up a benchmark project, along side our normal unit tests and make it a first class citizen of our project.
I have to thank the makers of [BenchmarkDotNet](https://github.com/PerfDotNet/BenchmarkDotNet) who have enabled a proper testing
framework with actual output. Warmups, and all that best practice wrapped up in a nice little nuget package for you.

So here is our solution layout under the test folder now
![solution layout](https://cetus.io/images/PerformanceCounts/Testfolder.png)

Now we will setup the framework in the console application to run the tests. First the test configuration which looks like this

``` json
"dependencies": {
  "BenchmarkDotNet": "0.9.9"
},
"frameworks": {
  "netcoreapp1.0": {
    "dependencies": {
      "Microsoft.NETCore.App": {
        "type": "platform",
        "version": "1.0.0"
      }
    }
  },
  "net461": {
    "dependencies": {
      "BenchmarkDotNet.Diagnostics.Windows": "0.9.9"
    }
  }
}
```

To break this down, first we pull in the benchmarkdotnet package which does what it says on the tin. Next we will target both 
dotnet core and 461. In the setup post I explained why we are targeting this version of the full CLR. The reason we don't just 
target dotnet core? Well as you can see from the dependencies there is a set of Diagnostics that only run in the full CLR currently.
They will provide us vital information about memory allocations and JIT inlining which we will see shortly.

Next we setup a basic parameter that we can pass in the console arguments that allows us to run a single test or all in one go. This means
we don't need a new project for each benchmark but also allows us to run a single test (performance tests can take a while).

``` csharp
private static readonly Dictionary<string, Type> _benchmarks = new Dictionary<string, Type>(StringComparer.OrdinalIgnoreCase)
{
  ["LinearInterp"] = typeof(LinearInterpolationBenchmarks),
};
```

This currently has a single benchmark in it, however we will be adding more as we go along. Next is the code to actually run the benchmark

``` csharp
public static void Main(string[] args)
{
  if (args.Length > 0 && args[0].Equals("all", StringComparison.OrdinalIgnoreCase))
  {
    Console.WriteLine("Running full benchmarks suite");
    _benchmarks.Select(pair => pair.Value).ToList().ForEach(action => BenchmarkRunner.Run(action));
    return;
  }
  if (args.Length == 0 || !_benchmarks.ContainsKey(args[0]))
  {
    Console.WriteLine("Benchmarks");
    foreach (var kv in _benchmarks)
    {
      Console.WriteLine(kv.Key);
    }
    Console.WriteLine("All");
    return;
  }
  BenchmarkRunner.Run(_benchmarks[args[0]]);
}
```

The key to the above code is calling the BenchmarkRunner.Run passing in the type of the benchmark we want to run. All pretty simple
so lets jump over to the actual benchmark.

As we currently only have a single interpolator type we will start with that, first we put in the correct attribute for setup

``` csharp
[Config(typeof(ConfigSetup))]
public class LinearInterpolationBenchmarks
```

We will come to the ConfigSetup class shortly but for now just know this requires the benchmark to be run with the settings in that
configuration class

``` csharp
public const int NumberOfInterpolatorCreates = 100;
private double[] xValues;
private double[] yvalues;
```

The const is the number of times we will loop internally to the benchmark method. We will also tell the benchmark this and it will
divide our result by this number to give us a result per iteration of the loop. The next two members are to store the input values
we are going to test. We want to set these up outside the actual benchmark as this is not part of our test. To do this we make a 
setup method like so

``` csharp
[Setup]
public void Setup()
{
  Random rnd = new Random(777);
  xValues = new double[Pillars];
  yvalues = new double[Pillars];
  var stepPerPillar = 1.0 / Pillars;
  for (int i = 0; i < Pillars; i++)         {
  {
      xValues[i] = i * stepPerPillar;
      yvalues[i] = rnd.NextDouble();
  }
}
```
We once again use an attribute to tell the benchmark to run this before each benchmark. We setup the y values based on some random 
numbers but because of the fixed seed they should be the same "random" numbers each time to take out any difference that might change
any results between runs. The pillar parameter is setup here

``` csharp
[Params(1000, 5000)]
public int Interpolations { get; set; }
[Params(10, 20)]
public int Pillars { get; set; }
```

By using another attribute the benchmark will run all combinations of these and output the results in a table for us to get an idea
of our performance across a couple of different scenarios.

Now onto the actual benchmark, it's pretty simple really 

``` csharp
[Benchmark(Baseline = true, OperationsPerInvoke = NumberOfInterpolatorCreates)]
public void StandardBinarySearch()
{
  var reps = Interpolations;
  Random rnd = new Random(777);
  for (int x = 0; x < NumberOfInterpolatorCreates; x++)
  {
    var interp = new Qwack.Math.Interpolation.LinearInterpolator(xValues, yvalues, true, true);
    double value;
    for (int i = 0; i < reps; i++)
    {
      value = interp.Interpolate(rnd.NextDouble());
    }
  }
}
```

The attribute tells the benchmark the number of iterations as explained above and that this is our "baseline" which means any further
added test will be given statics in relation to this test. The rest is pretty easy to follow, now we build in release mode and run from
the console (after plugging my laptop into the power and turning everything up to 11!).

     