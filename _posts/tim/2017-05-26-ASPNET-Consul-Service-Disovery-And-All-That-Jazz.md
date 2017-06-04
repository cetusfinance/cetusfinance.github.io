---
title: "ASPNET-Consul-Service Disovery and All That Jazz"
excerpt: "The second in a series of short video tutorials that take you past hello world Web APIs"
category: "tim"
header:
  overlay_image: jazz.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: ASPNETCOREWebApiGettingStarted/teaser.jpg
tags: [aspnet, webapi, aspnetcore, visual studio, vs code, consul]
---

## The background

This second part of the series will assume you know how to make a basic WebAPI project and how to lay it out. If not I would recommend looking at the first part at

[ASPNET-CORE-WebApi-Getting-Started](https://cetus.io/tim/ASPNET-CORE-WebApi-Getting-Started/)

If you would just like links to the videos you can find them at the bottom of this article.

## Who this is for?

Anyone that wants learn in two quick (~30mins each) sessions via video how to do service discovery and dynamic configuration. The tool we will use to do this is
[consul.io](http://consul.io). It has been chosen because it will work just as well on prem as any of the cloud providers. Obviously all the providers have fully
integrated methods for doing this as well.

## The Scenario

We have changed to a new scenario of a API that takes in swagger documents, saves them to a SQL Database in Linux and eventually will do some validation on them.
The scenario itself is not overly important, more that we have a webapi and we want to be able to discover instances of it at runtime and have some configuration options.

## Zero Config Service Discovery

[GitPitch Presentation](https://gitpitch.com/Drawaes/Condenser.ApiFirst/Tutorial1)

So in this presentation we see how to have our service registry with consul, including multiple instances on dynamic ports. We will look at how to implement a health check,
how to lookup these services and finally how to keep our registry clean.

## Configuration and Logging

[GitPitch Presentation](https://gitpitch.com/Drawaes/Condenser.ApiFirst/Tutorial2)

We see how to avoid configuration in most instances completely. For the instances where we actually need configuration we see how to use the inbuilt ASPNETCORE mechanisms and
then how to make that dynamic from consul.

Finally we add serilog and see how to have the configuration of serilog trigger dynamically also.

## Finishing up

Thats the end of the second set of tutorials, I hope you enjoyed them. I would love to hear any feedback anyone has. I have moved the links in the presentations to the start for this series
as a couple of people noted that if they wanted to follow along they had to go to the end to get the links. I have also increased the font size a lot to help with clarity. Anymore feedback or anything
you would like to see? If so please to flick me a note on twitter or a comment here. 

The topics here were a step above the previous set, in the next tutorial set we will look at moving to ASPNETCORE 2.0 Preview 1, how that changes our configuration and host setup and some advanced
setup and docker use (if I can get it all working!).

## The raw videos

In case you don't want to go through the presentations

1. [Tutorial 4 Playlist](https://www.youtube.com/playlist?list=PLmH6QaxzgYQuAIJ310g5LNaX2Iui9zSrX)
1. [Tutorial 5 Playlist](https://www.youtube.com/playlist?list=PLmH6QaxzgYQtW5lv51IiqroNlHH38m4Ue)