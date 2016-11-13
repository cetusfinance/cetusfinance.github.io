---
title: Setting Up
excerpt: "Starting an opensource project - Qwack"
category: "tim"
header:
  overlay_image: start.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: startTeaser.png
tags: [setup, CI, testing, oss, .net, qwack]
---

# What?
An opensource quant library. Simple right? We want it to support both running on servers as well as directly in a number of UI's 
such as an excel plugin as well as a modern Web/Chromium UI. We want it to cover a broard range of instruments, and products and make
it accessible.

# Why?
Quant libraries have an interesting history. There are great math libraries out there, and some quant libraries but they are not really "all there".
When looking around none of them seemed to provide a whole package that would allow you to use the very same code from a trader or quants excel straight through
to the server, and be able to perform and be a proper type checked and tested solution. As a company we have this need for some of our ideas we have, so rather than
build it and have another closed source solution, why not be the first to make it opensource?

# How?
We are going to write the software in a feature by feature manor. Not adding "potential things we need", or as few as possible without a usecase. If you have a usecase you
are interested in please do add an issue, we will be looking for things to add. Even better feel free to write some code! If you don't want to do that we would still love you 
to add issues for usecases or features you would like to see. We will be using .Net Core and making our entire library cross platform.

So here we are looking to write an opensource quant library in .Net Core. 
Upfront we have some basic requirements for the project, without which it is just 
a learning excerise at best and at worst completely pointless.

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
