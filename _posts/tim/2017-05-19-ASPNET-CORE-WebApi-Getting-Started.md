---
title: "ASPNET CORE WebApi Getting Started"
excerpt: "The first in a series of short video tutorials that take you past hello world Web APIs"
category: "tim"
header:
  overlay_image: ASPNETCOREWebApiGettingStarted/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: ASPNETCOREWebApiGettingStarted/teaser.jpg
tags: [aspnet, webapi, aspnetcore, visual studio, vs code]
---

## The background

This is a series of short videos that will take us through starting a basic WebAPI. It will start at the very basics, laying out a solution and projects. We will use MVC and ASPNETCORE to write the WebAPI 
based around having users and repos, with information about them. We are going to use GITPitch to 
provide the tutorial in a presentation layout. 

If you would just like links to the videos you can find them at the bottom of this article.

## Setup of the project

[Presentation](https://gitpitch.com/drawaes/CodePersuit/Tutorial1)

So in this presentation we setup a basic project and showed that we could access it via a browser. If you
have written a WebAPI before you can skip it.

## A little bit of data

[Presentation](https://gitpitch.com/drawaes/CodePersuit/Tutorial2)

This tutorial takes us a little further. We use Sql Server on Linux running in docker. We script the 
database from clean (creating the database, the tables, and the data). We then use Dapper to get the 
data out of the database. Finally we clean up the solution a little.

## An API with some Swagger

[Presentation](https://gitpitch.com/drawaes/CodePersuit/Tutorial3)

This is the last piece of this first tutorial set. Here we create a swagger document from out WebApi.
We then use this document to make a client sdk, and finally show how to make the sdk a little cleaner
for consumers.


## Finishing up

Thats the end of this first set of tutorials, I hope you enjoyed them. For the next set we will ramp
up the complexity. We are going to change scenarios to a project I am actually working on and show how
to get some reliability and discoverability while cutting down on manual config for backend services.

## The raw videos

In case you don't want to go through the presentations

1. [Tutorial 1 Playlist](https://www.youtube.com/playlist?list=PLmH6QaxzgYQsvrvJaDZOkZ8mMHK38OKQW)
1. [Tutorial 2 Playlist](https://www.youtube.com/playlist?list=PLmH6QaxzgYQt2hWs7SulTZ234iynqB-_2)
1. [Tutorial 3 Playlist](https://www.youtube.com/playlist?list=PLmH6QaxzgYQsdUh8dcXFBn5zlwIRK5nIA)
