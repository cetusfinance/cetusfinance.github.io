---
title: Off the starting line?
excerpt: "What/When/Why/Who..."
category: "tim"
header:
  overlay_image: start.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: startTeaser.png
tags: [setup, CI, testing, oss, .net, qwack]
---

# Who?
I am Tim, and the other person you will see writing around here is Gav. We both meet in the city of London a number of years ago and have 
coded together on and off just for fun for a number of years. We have had some great ideas (and not so great!). We thought that people out
there might have some interest in these many and varied late night coding topics and discussions and we thought that the best format to bring
this to you was in blog format. It will be a mix of mathematics and coding related topics and will revolve heavily around writing C# and high
performance code to solve complex real world problems, in the simplest matter possible. 


# What?
An opensource set of code and accompanying articles discussing everything from making a high performance set of services
to math functions in a managed library. Simple right? We want it to support both running on servers as well as directly in a 
number of UI's such as an excel plug-in (everyone loves a good excel plug-in) as well as a modern Chromium UI. 
We want it to cover a broad range of topics, theories and make it accessible, but also cover a broad range of topics from vectors/gpu's, communications
you name it we like fiddling

# Why?
Quant libraries have an interesting history. There are great math libraries out there, and some quant libraries but they are not really "all there".
When looking around none of them seemed to provide a whole package, explanation and documentation is in short supply. They are complex and require a lot of intervention and research
from implementors.  Often higher level libraries are made in scripting languages and the lower level math functions in low level languages, there are very few examples that provide both
in a middle ground language that can provide clarity and ease of use and be able to be used on the desktop right through to the server, and be able to perform and be a proper type checked and tested solution. 
Maybe it's impossible to do, maybe not.. but why not try it might be fun!

# How?
We are going to write the software in a feature by feature manor. Not adding "potential things we need", or as few as possible without a a purpose. If you have a use case you
are interested in please do add an issue, we will be looking for things to add. Even better feel free to write some code! If you don't want to do that we would still love you 
to add issues for ideas, bugs, improvements or features you would like to see. We will be using .Net Core and making our entire library cross platform.

Upfront we have some basic requirements for the project, without which it is just 
a learning exercise at best and at worst completely pointless.

* Performance matters and needs to be baked in from day one
* X-Plat or cross platform is becoming extremely important
* Easy deployment and the ability to plug into diverse data sources
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
* ExcelDna for excel integration

Later down the line we might add other tools like Chocolaty, and maybe even a JS framework for making a spiffy UI but for now we just want to get some meat on the bones

If you head over to [qwack](https://github.com/cetusfinance/qwack) you can see the basic framework setup and building and some nice badges on the read me. If you are reading this months down the line, then maybe you 
want to look at just the first commits if you just want to see how we started out setting things up.

So now what? Look out for the next post where hopefully I will get my first code past a code review and discuss Interpolation (and vectors :thumbsup:)

If you are interested in the details of setting it up look for a future blog post on the subject.
