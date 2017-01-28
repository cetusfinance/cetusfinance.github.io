---
title: "TLS 1.3 - Who moved my cheese?"
excerpt: "Getting on with it - Pipes Part 2&#xb2;. Digging into the code"
category: "tim"
header:
  overlay_image: whomovedmycheese/cheese.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: whomovedmycheese/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl,secrecy, leto]
---

### Prior Reading?

This series is likely to end up with a number of parts, I am going to just go ahead and assume prior knowledge.
Everything you should need to know to get up to speed is in the previous posts.

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)



[^1]: [Pseudo Random Numbers from Wikipedia](https://en.wikipedia.org/wiki/Pseudorandom_function_family)
[^2]: [Calculating the premaster secret - TLS 1.2](https://tools.ietf.org/html/rfc5246#section-8.1)
[^3]: [The heart bleed bugs very own website](http://heartbleed.com/)
[^4]: [Diffie–Hellman key exchange from Wikipedia](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
[^5]: [Botching session resumption](https://timtaubert.de/blog/2014/11/the-sad-state-of-server-side-tls-session-resumption-implementations/)
[^6]: [Twitter's implementation of session resumption](https://blog.twitter.com/2013/forward-secrecy-at-twitter-0)
[^7]: [RFC5869 HMAC key derivation function](https://tools.ietf.org/html/rfc5869)
[^8]: [Trusted platform module](https://en.wikipedia.org/wiki/Trusted_Platform_Module)