---
title: "Instruments Galore"
excerpt: "Finally we can calculate some numbers!"
category: "tim"
header:
  overlay_image: https://cetus.io/images/instrumentsgalore/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: https://cetus.io/images/instrumentsgalore/teaser.jpg
tags: [fx, ir, solving, qwack]
---

# ArrayPool, MemoryPool, IOwnedMemory and all that Jazz

For this post we are going to jump straight into looking at some pools. First up is one in .Net called "ArrayPool"

The basic methods and use looks something like this

``` csharp
var pool = new System.Buffers.ArrayPool();
var myBuffer = pool.Rent();

// some code to do cool things with a buffer

pool.Return(myBuffer);
```

