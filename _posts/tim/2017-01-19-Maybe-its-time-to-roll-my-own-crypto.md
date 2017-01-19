---
title: "Maybe it's time to roll my own crypto, someone has to do it?"
excerpt: "Rewind and Restart - Pipes Part Zer0"
category: "tim"
header:
  overlay_image: rollyourown/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: rollyourown/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl,nss, sslstream]
---

## Before we get started

In case you missed it there are three parts before this, to be honest they aren't required reading because well 
this is a total change of tact on this subject.

* [Part 1 - Not your Grandad's Dotnet](https://cetus.io/tim/Part-1-Not-your-grandads-dotnet/)
* [Part 2 - Pipelines formerly known as channels](https://cetus.io/tim/Part-2-pipelines/)
* [Part 2 - Pipelines OpenSsl](https://cetus.io/tim/Part-3-Pipelines-OpenSsl/)

# Dont' roll your own crypto

The more I researched into TLS, the history, the attacks, looking at the code in OpenSSl, NSS, Go all over the place
the more I started to think that .Net should probably have it's own TLS library. Of course the minute I even considered
this idea outloud I heard the screams... don't roll your own crypto! I get it, crypto is hard, it's brittle, you can use all the 
right words/ciphers and still break the whole lot by a bad implementation.

![Cry Me a River](https://cetus.io/images/rollyourown/cry.jpg)

However I think it is madness. I am not rolling my own crypto dispite what the shouting masses say. The quote was originally
about actually making your own cipher. I had no intention of ever doing that. TLS is a protocol it is not "Crypto". It is also the
corner stone of security in our modern world. Today almost everything already goes over or should go over TLS. Then people are told to not even bother to look into it. 

![NSA Headquarters](https://cetus.io/images/rollyourown/nsa.jpg)

Now let me wind back a little, I don't plan on writing my own cipher code,
there are plenty of libraries out there, they are heavily checked and easily unit tested. 
Intel even gets it's engineers to dig in and make these things faster (AES in OpenSsl for instance). 
My problem is around the actual protocol handling, the state machines, handshakes etc. It doesn't even seem like
a good fit for C so why are we stuck there?
So the reasons not to write a managed TLS library in no particular order were,

1. Don't roll your own crypto - Just don't do something has never been a good enough reason not to try
2. It won't be NIST/[Insert your audit agency] Audited - Is that really true? Most standards approve at the cipher level not at the TLS protocol level if I leverage those then it should be okay still. If not, well so be it.
3. It won't be fast enough - I am not so sure and I will explain

Let's take a look at what Amazon have to say about why they wrote S2N (Signal to noise) even if you don't watch the whole
thing I have bookmarked it at what I think is some of the most important points, before you rely on OpenSSl.

<iframe width="854" height="480" src="https://www.youtube.com/embed/APhTOQ9eeI0" frameborder="0" allowfullscreen></iframe>

So let's assume that I am going to ignore all of that, and do it anyway what should our non functional design goals
for a TLS library be?

1. Buffer overflows - Seems to be one of the biggest pitfalls in any security library
2. Separating CipherText and PlainText so avoid leaking PlainText
3. Long Term secrets should be protected at all costs
4. Ephemeral secrets should be treated carefully, and ideally protecting it from long term persistence
5. The cognitive load of the code should be minimal
6. Unit tests
7. Cross platform
8. Comments pointing to where in standards/specs information comes from
9. It should be able to keep up with SslStream and ideally a lot better
10. Defend against most known attacks

(At this point I am more than happy to know anything I have missed)

On the functionality I want to provide, my starting point is

1. TLS 1.3 out of the box
2. TLS 1.2 -> the lowest version (yet to be decided) added later and in a thoughtful way
3. ECDSA and RSA Certificates, available from a store/TPM or external service with private keys being managed by well known solutions
4. Ephemeral Keys should be kept in "locked" memory off the paging file and zeroed at the earliest possible time
5. A "provider" layer with crypto implemented by OpenSsl and CNG.
6. Session ticket encryption keys should be able to be provided by an external service or provider and rotated with strong encryption

First off a name, because people in the crypto world like to be mystical and high brow and because all the names are taken I managed to
find so with the use of Wikipedia, Leto it is.

> Older sources speculated that the name is related to the Greek λήθη lḗthē (oblivion) and λωτός lotus (the fruit that brings oblivion to those who eat it). It would thus mean "the hidden one".

As the primary goal is to hide our data so that seems reasonable.

TLS 1.3
If we started at 1.2, well so what we have a library like that already sitting on SChannel and OpenSsl. So looking at TLS 1.3 
is a more interesting challenge.

![Testing](https://cetus.io/images/rollyourown/lab.jpg)

## Does my code work?
The nice thing about testing TLS 1.2/1.1/1.0 etc is there is a wealth of test clients. Chrome, Firefox, Safari you get the idea.
But better than a browser during development, 

```
OpenSsl s_client -debug -msg -connect 127.0.0.1:443
```

had been one of my favorites. So that is what I was using for testing, with the latest code and built with TLS 1.3 enabled. As at the time
of writing this post (2017-01-19) DON'T DO THAT. It really really doesn't work, and to be fair I am not sure anyone actually claimed it did.

An example of the problems, if you split the first flight (we will get to that later) of the server over multiple records and incremented the nonce
it just doesn't work, so you have to reuse the nonce... this was the last straw in my testing. I am no expert but reusing a nonce (the clue is in the name)
isn't cool. So after changing my code to try to work with OpenSsl in TLS 1.3 I realised maybe I was right in the first place.

So it was time to go an look for more complete implementations. The first stop was Chrome Canary and Firefox nightly. They are both solid implementations
(once again as of 2017-01-19) of draft 18. The problem with testing with a browser is you end up with a "site not secure" or some other vague response when
you get it wrong. Next stop was NSS, which was starting to look like it worked, however there were issues around stdin on windows. 

![Docker](https://cetus.io/images/rollyourown/docker.png)

Time to turn to docker, 
I used the docker stuff from Tls-Tris to get it setup so a big thanks to that project. I did sometimes look at Mint and Tls-Tris (both Go projects for TLS 1.3),
but I tried to avoid it, other than taking some DER encoded certificates. The reason? Because I think we need multiple independent implementations that can be
tested against each other, as well as having multiple implementations for people to find flaws in and learn from.
(Hint I am also using docker to make sure everything still works cross platform)

So, armed with a docker image with NSS tstclnt and Visual Studio I was off. Over the next few posts I will explain in detail what I have written
so far, what works, what doesn't and how they are implemented. I would love feedback, especially the kind of feedback that is "this is totally broken because of
X". Even if it is worded as "you are stupid and here is why".

As I have written a large amount of the code, I will try to keep the blog posts for this coming thick and fast.

[Github Repo for Leto](https://github.com/drawaes/leto)

A currently working server (at time of publishing) it might crash any day now and you need a TLS 1.3 compatible browser
or you will get rejected.

[TLS 1.3 Server](https://tls13.cetus.io)

