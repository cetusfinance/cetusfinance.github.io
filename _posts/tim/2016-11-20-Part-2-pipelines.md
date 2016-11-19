---
title: Pipelines formally known as channels
excerpt: "A faster lower allocation stream stack for the future"
category: "tim"
header:
  overlay_image: pipelines/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: pipelines/teaser.jpg
tags: [asp.net, .net, channels, pipelines, networking, tls, ssl]
---

# Channels... not any more

So the pipelines library started life as "channels" a repo on github under David Fowlers personal account. By the time
I came to look at it, it was up and running, buggy sure, API unbaked but it was functional. To look at how it was and a 
good kick starter (plus a blog always worth reading) take a look at Marc Gravell's early post on the subject.

[http://blog.marcgravell.com/](Marc's post on channels)

His final comment about what to call it has now been answered by Microsoft, and that is Pipelines.

If you want an overview of how it is hoped it will improve asp.net you can take a look at this performance
overview [https://github.com/dotnet/corefxlab/blob/master/docs/roadmap.md](here) it outlines what is going on to make
things faster. You might say so what? I don't care about the web, or http I use raw sockets and some custom binary protocol
well don't worry, pipelines is down at the "stream" level of the stack, it's in no way tied to asp.net and it could be used
for all sorts of scenarios, sockets, files you name it.

Onto the SSL/TLS story, when I first started to look at this I probably knew as much as the average developer. A server
has a certificate that is certified by a known certificate authority. It has a public/private key and it is used by the client
to make sure the server is who they say they are. This is also then used to perform some sort of "handshake" to agree on a
key (well actually a set of keys) used to then do all communication from then on. This enables authentication of the server
(and the client if using client certs although that is less common) and to secure further communications... easy right?

Well yes and no, my basic assumptions were correct, but I figured lets get some background before I just off and port SslStream.
I thought SslStream has been around for sometime, had Xplat added to it and made in another time. Ben Adams seemed to hint (well more than that)
at the "allocaty" nature and maybe a fresh look was needed. So then where to start? I always find that it's best if you want
to deep-dive something then you go to the source and work up, that way you are less likely to build assumptions into your 
thinking before you have some grounding in the topic. That left one source.... IETF specs

1. [https://tools.ietf.org/html/rfc6101](SSL3) - mostly for some historical background I figured I didn't need to go back further than that
2. [https://www.ietf.org/rfc/rfc2246.txt](TLS 1.0) - now we are starting to get somewhere
3. [https://www.ietf.org/rfc/rfc5246.txt](TLS 1.2) - the good stuff

Let's just say they don't make for the most exciting reading, but I was armed with just enough knowledge to be dangerous,
time to getting started writing a security library how hard could it be?

## SSPI

Pretty quickly I realised why no one wanted to touch this stuff. SSPI was my starting point, and unlike say .net core is 
certainly isn't open source and the API documentation that is on MSDN is very very old which was a bit of an issue. The
upside was that SslStream is opensource so I decided that it was time to take a look. What I found (and is the pattern for
OpenSsl as well) is that you have a set of security credentials or a context and this contains information like a certificate
and your settings. This is then used to generate a set of session keys etc and then that session is essentially detached and not
reliant on the initial credentials.

This is important and one of the reasons that I still believe that SslStream is not the way forward for server side TLS (I am going to 
stop refering to SSL from now on, it's broken). Sslstream has the concept of making a stream, passing in the upchannel or wrapped stream
and the certificate etc and carrying on. This leads to a major inefficency, making a authorization context for every conenction.
Even if you could pool and reset the SslStream you would still have the authorization per connection. There is really no way around
this but changing the API heavily. As SslStream is used in a lot of places and needs to maintain backwards compatiblity these changes
to the API really aren't feasible. So in summary this is how things are, and where I thought they should be

![Diagram of SslStream vs the proposed](/images/pipelines/concept.png)

