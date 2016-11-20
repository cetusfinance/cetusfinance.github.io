---
title: Not your grandad's .net - Pipes Part 1
excerpt: "How the world really has changed for .net developers."
category: "tim"
header:
  overlay_image: granddadsnet/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: granddadsnet/teaser.jpg
tags: [microsoft, asp.net, core, .net]
---

# C#, it's what you use for fat clients on windows right? Wrong!

One of the reasons I have stuck with .net and made c# the primary
language I focus on has always been, well the c# language. I just love
the syntax and the pushing forward in the language development, yet the sense
that someone is watching over it guiding it and stopping it from fragmenting into
a million pieces. 

It's a great C style language but much clearer and safer than
C or C++ as well as having the second mover advantage on java (getters and setter always come to mind).
Over the past few years it has been harder and harder to justify. .Net was finding it's home in 
corporations, line of business applications and windows GUI's. 

I was seeing a pattern emerge, the ".net"
teams were writing thick client apps and the server code was on java or c++ or Go. As the OSS community
really started to gain traction I found more and more I was diving into java, then Go and even sometimes
enjoying myself. The .net ecosystem certainly wasn't dieing but it was moving at the pace of the rest
of the world.

[Transform](/images/granddadsnet/transform.jpg)

## But then ..

But all that started to change, Microsoft has made a massive shift that has often gone unnoticed
both by people from other languages/linux patforms etc. But even more so from corporate developers
who use .net on a daily basis. 

Microsoft these days is looking like one of the more intouch technology
companies, their engineers, and architects are available and open to discussion, rapid feedback cycles,
code openly being discussed on GitHub, real Q&A and community stand-ups and a willingness to listen.

Both Microsoft and the community have benedited enormously from this with the ultimate winner being the .net
platform itself. Last week microsft held a 3 day event live streamed called Connect(). The announcements were
both surprising and very welcome at the same time. Some of the key points that took from it were

1. ~60% of the .net core code base has been contributed from outside Microsoft
2. [Techempower's round 13](https://www.techempower.com/blog/2016/11/16/framework-benchmarks-round-13/) results show asp.net on LINUX to be number 10 on plain text, with those above been mostly very thin libraries and not full frameworks
3. Google has joined the .Net Foundation... yes google!
4. Microsoft has joined the [Linux foundation](https://techcrunch.com/2016/11/16/microsoft-joins-the-linux-foundation/) as a platinum partner.... say what?

So what now are they done? Not on your life, [.net core 1.1 is RTM](https://blogs.msdn.microsoft.com/webdev/2016/11/16/announcing-asp-net-core-1-1/) 
and that is great but Microsoft and the community have
been cooking up even more for .net core 1.2.

![salmon](/images/granddadsnet/salmon.jpg)

## The journey begins

So this brings me to pipelines. About a month ago I was hanging out in a chat room about Microsoft Orleans. It's a very
cool distributed actor model based platform (also opensource). I was digging and playing with the code mostly to see if I 
could use the system to provide a kind of "pricing session" cache for shifting objects out of process locally or potentially
out to a farm of servers. Excel 32bit is prevalent in the world still and that is just not enough address space to do 
some of the things I want.

It turns out that while Orleans is very cool in the performance and latency sensitive tasks I had in mind it wasn't 
suited to what I wanted, I was too chatty and sometimes in math calculations mutable state is the only way to get things
done quickly. 

However while I was in the chat there was some discussion about replacing the networking layer with an 
external library so that Orleans could focus on what it was designed for and not low level concerns. Some random person
popped into the chat and mentioned he was working on a new model to replace .net streams with something that would reduce
memory allocations, and copying by reversing the control of buffers. This sounded interesting...

![crew](/images/granddadsnet/crew.jpg)

## Meet the high performance crew

Turns out people in the community know [David Fowler](https://twitter.com/davidfowl), I hadn't heard of him, [Ben Adams](https://twitter.com/ben_a_adams)
or [Marc Gravell](https://twitter.com/marcgravell) (but had been using Marc's libraries 
for years). It just goes to show you the corporate bubbles I had lived in. Anyway I rocked along to the chat they were in,
rudely interrupted and asked a bunch of sometimes I imagine boring questions.

They were all really nice even as I criticized the API, most of my feedback in other circles might have been considered moaning,
however they constantly encouraged it as useful to hear. They took it as it was meant constructive criticism.

So I made an example project, pushing rapid messages serialized in various formats, mostly binary from things like CME data
feeds and others and found the out of the box performance was nothing short of amazing. 

I wanted to help get this into the 
core framework so others could use it, so .net would be faster and better and well their enthusiasm was kind of infectious.

So the question was posed "Is there anything I can help with?" the resounding response was "SSL/TLS". Seems everyone wanted
it but no one wanted to tackle it. That is where my journey began, it has been long and isn't over yet!

In the next post, I will try to discuss TLS, the security landscape of the world and where I have got to with TLS in 
Pipelines.


