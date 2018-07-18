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

# In the current heat we are all feeling let's discuss pooling in .Net

A pool is a set of objects that we can reuse over and over again without having to create a new instance each time. Pooling is an important part of writing an efficient and high performance server side application. While pooling is sometimes used client side in specialized scenarios it will most commonly be found in server side applications so that is what we will mostly deal with here. One client side application of pooling you might find is in the area of gaming where similar sets of objects (say for graphics) are produced over and over in fast tight rendering loops. There are different implications for those pools which we might touch on but generally this is focused on a server serving many clients quickly.

# Why do you use a pool? 

Allocating is practically "free" with a managed memory based runtime. All you need to do to allocate in its simple form is increment a pointer. However when you create many objects and then free them the freeing of these objects can become costly. During a garbage collection the code needs to walk the tree of objects to find ones that have been freed and to then mark them (find objects that are no longer rooted and can be cleaned up). Once they are freed heaps need to be compacted and then memory released or zeroed and reused for more allocations.

Allocating and de-allocating large numbers of objects leads to behaviors like Garbage collector pauses or if you are running on a more aggressive GC mode (workstation GC) it will appear as high CPU use outside of your application code or an inability to scale. This is why there has been a big push to reduce "allocations" in the .NET framework. However sometimes you need objects as not everything can be on the stack either due to size or lifetime requirements (the need to survive across async boundaries).

# The middle aged object crisis

So very short lived and small sized objects can be stack based (structs, stackalloc etc), objects that are very short lived or long lived are usually not a large problem either (they still have a cost but unless you are making many in a tight loop they generally won't cause large issues). However we have the problem of "middle aged objects".

# The Generational problem

.NET has a multi generation garbage collector, generation zero collections happen often and are quick. However if an object survives this it will be promoted to the next generation (baring certain edge cases). As these are pushed up the generation each collection of these generations takes a lot longer and is more costly. This generally works well however if we take the basic example of a socket based application we might create a new buffer (say a Memory<T> from an empty array) and then we wait for data. This will most likely cross a gen zero collection, we will use the buffer to collect data and pass it off to some application code and then create a new empty Memory<T> and wait for more data. The application will then release the first buffer and this now becomes a "Middle age" buffer. It will likely hang around for a while and if we are going fast enough these will build up quickly and cause a full multi generation GC. This can cause a "stop the world" point in the GC and freeze our application.

So how can we avoid this middle age object problem? We can take some power back from the GC. To do this we will create our own pool of buffers. The socket can take a buffer from this pool, fill it and pass it along to the application code. When the application code finishes with the buffer it will give it back to the pool to allow it to be reused. The socket doesn't need to understand when the application code is done with the buffer and when it should be reused.

# Wait how can we do better than the GC devs?

It might seem like a certain level of hubris to think we can do better than the Garbage Collector developers at Microsoft. The reality is it would be if that is what we were suggesting. However we have one important advantage. The GC developers make a memory allocator/de-allocator that is at a distinct disadvantage to our situation. We have application knowledge that we can tailor our pooling to and we can give feedback from our application when we will need the memory, when we are done with it, and the exact type of object we will need.

# Okay so what next?

We are going to focus on pools of buffers for this series. By buffers I define this as a series of repeated structs. Normally this is an array of byte or some other primitive although you can use other structs. Pooling stateful single object instances is a different topic with a heavier focus on resetting state and other considerations.

# Pooling Pros and Cons

There are a number of different tradeoffs for pooling but none of these are absolutes, and often they vary depending on the actual implementation of the pools. So we will touch on the issues and explore them in more depth and we take a deeper look at the pools we are going to investigate.

## Pros

1. Reduced Garbage Collection
1. Data locality
1. Ability to provide memory back pressure
1. Can actually reduce memory use

One is really the main reason for pooling, and is often enough reason alone to use pooling. The other two are more niche scenario.

## Cons

1. Can (usually will) have more overhead to get an instance from the pool than "new T[]"
1. Can be slower to return to the pool or the cost is paid entirely on the returning thread
1. Often objects/buffers aren't pre-zeroed
1. Can hold onto more memory
1. Needs more careful coding, returned buffers that are held and written to can cause bugs!

## Next

Now that we have layed out what memory/buffer/object pools are, and why we might want to use them we can move onto the next part in the series and take a look at some implementations in .Net Core and how they are constructed.



