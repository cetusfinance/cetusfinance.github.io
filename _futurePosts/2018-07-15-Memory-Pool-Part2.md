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
var pool = new System.Buffers.ArrayPool<byte>.Shared;
var myBuffer = pool.Rent(minBufferSize: 1024);

// some code to do cool things with a buffer

pool.Return(myBuffer);
```

This might be the first, and probably the easiest way for most people to get into pooling. There is nothing wrong with this method however you should be aware of a few things about this pool and why it might not be the best approach for your application.

First up you get back an array. If you don't know anything about the new Span<T> or Memory<T> types now would be a good time to go and get some in depth understanding of these types. Stephen Toub has a great post about this <!***************INSERT TOUB BLOG POST>here<***********/>. I am going to assume you understand those types, if you don't go and read that post.

So we have an array, if we want to use sections or slices of that array we can put it inside a Span<T> or Memory<T> and if the method can only take an array we can use that as well. So we have solid flexibility however, we need to ensure that the array is returned to the pool. Our higher level code must be the one returning the array because any method we pass the array into doesn't have any understanding of the lifetime of the array or where it came from.

So say we have some code that takes data from the network, does some processing and then hands that array off to some application that may take a while, we need to either copy that data inside the application code and return the buffer straight away to the networking code, hardcode the ArrayPool into our application code (meaning that we are now dependent on the one implementation of a pool), or finally we could wrap the array in a class or struct and have that understand where the array should be returned. This would allow decoupling the buffer and pool implementation from our application. So if we would develop something like this what might it look like?

Would we make it implement IDisposable and return the buffer on dispose to be reused? It would probably contain a Memory<T> rather than an array to give us more flexibility around the source of the memory (if we wanted to use unmanaged memory or chunks of an array which we will get into in later articles). Well helpfully there are classes in .NET Core 2.1 that already provide this.

## Enter IOwnedMemory<T>