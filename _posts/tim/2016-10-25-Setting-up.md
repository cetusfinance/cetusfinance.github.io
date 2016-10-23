---
title: Setting Up
excerpt: "Starting an opensource project."
category: "tim"
header:
  overlay_image: start.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: startTeaser.png
tags: [setup, CI, testing, oss, .net]
---

So here we are looking to write an opensource quant library in .Net Core. 
Upfront we have some basic requirements for the project, without which it is just a learning excerise at best and at worst completely pointless.

* Performance matters and needs to be baked in from day one
* Xplat or cross platform is becoming extremely important
* Easy deployment and the ability to plug into diverse datasources
* Code quality needs to be high from the start and needs to be maintained
* A community and a policy of openness from day one
* Automate as much of the mundane as possible

Building out and setting up all of this is no small task. Even though Microsoft is moving the cheese on the project file layout for .net core soon we will push forward.
We will roughly try to use the following set of tools to kick things off, but are open to other ideas or help if anyone has some?

* Gitter for an open chat room around the project
* BenchmarkDotNet for measuring performance from day one
* Coveralls.io + OpenCover for code cover of tests
* Xunit for writing our tests both integration and unit based
* Travis + Appveyor for Ubuntu, Osx and Windows CI builds and tests
* MyGet for CI Nuget package builds
* Nuget for when we actually get to a proper release

Later down the line we migh add other tools like Chocolaty, and maybe even a JS framework for making a spiffy UI but for now we just want to get some meat on the bones

If you head over to [qwack](https://github.com/cetusfinance/qwack) you can see the basic framework setup and building and some nice badges on the readme. If you are reading this months down the line, then maybe you 
want to look at just the first commits if you just want to see how we started out setting things up.

So now what? Look out for the next post where hopefully I will get my first code past a code review and discuss Interpolation (and vectors :thumbsup:)

If you are interested in the nitty gritty of setting it up look for a future blog post on the subject.
