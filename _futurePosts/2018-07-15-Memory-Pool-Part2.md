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
# Our First Foray into Pools

## The Shared ArrayPool

For this post we are going to jump straight into looking at some pools. First up is one in .Net called "ArrayPool"

The basic methods and use looks something like this

``` csharp
var pool = new System.Buffers.ArrayPool<byte>.Shared;
var myBuffer = pool.Rent(minBufferSize: 1024);

// some code to do cool things with a buffer

pool.Return(myBuffer);
```

This might be the first, and probably the easiest way for most people to get into pooling. In fact if you use the "defaults" on pipelines and internally in things like SslStream you are already using this pool. So let's dig into the properties of this pool to use it as a baseline for future pool discussions.

## The API

First up you get back an array. If you don't know anything about the new Span<T> or Memory<T> types now would be a good time to go and get some in depth understanding of these types. Stephen Toub has a great post about this <!***************INSERT TOUB BLOG POST>here<***********/>. I am going to assume you understand those types, if you don't go and read that post.

So we have an array, if we want to use sections or slices of that array we can put it inside a Span<T> or Memory<T> and if the method can only take an array we can use that as well (we could also use the older ArraySegment but we are going to ignore that for the newer types). So we have solid flexibility however, we need to ensure that the array is returned to the pool. Our lower level code must be the one returning the array because any method we pass the array into doesn't have any understanding of the lifetime of the array or where it came from.

So say we have some code that takes data from the network, does some processing and then hands that array off to some application that may take a while, we need to either copy that data inside the application code and return the buffer straight away to the networking code, hard-code the ArrayPool into our application code (meaning that we are now dependent on the one implementation of a pool), or finally we could wrap the array in a class or struct and have that type understand where the array should be returned.

This would allow decoupling the buffer and pool implementation from our application. So if we would develop something like this what might it look like?

Would we make it implement IDisposable and return the buffer on dispose to be reused? It would probably contain a Memory<T> rather than an array to give us more flexibility around the source of the memory (if we wanted to use unmanaged memory or chunks of an array which we will get into in later articles). Well helpfully there are classes in .NET Core 2.1 that already provides this called IOwnedMemory and we will get to that in later pools. But keep in mind that this ArrayPool returns an array that has no ownership semantics.

## Inside the sausage factory

The default shared buffer uses a bucketing principle. Because you can ask for various sized buffers and it is unknown ahead of time what these might be it has a number of buckets of various sized buffers available for us. So first we can take a look at the defaults for the number of buckets and the maximum size for our bufferS.

``` csharp
/// <summary>The default maximum length of each array in the pool (2^20).</summary>
private const int DefaultMaxArrayLength = 1024 * 1024;
/// <summary>The default maximum number of arrays per bucket that are available for rent.</summary>
private const int DefaultMaxNumberOfArraysPerBucket = 50;
```

The shared pool uses the default settings. So we can see 1mb is the largest array and we have 50 arrays per bucket. This means if we try to take 51 buffers in a single bucket and return them all then the 51st buffer will be left for the garbage collector to clean up. This is something to keep in mind if you plan on having a number of buffers outside the pool at anyone time. Also remember that you might not be the only one using the pool so that number could be a lot lower.

Next we can take a look at the logic inside the pool for taking a buffer.

``` csharp
int index = Utilities.SelectBucketIndex(minimumLength)
if (index < _buckets.Length)
{
// Search for an array starting at the 'index' bucket. If the bucket is empty, bump up to the next higher bucket and try that one, but only try at most a few buckets
const int MaxBucketsToTry = 2;
int i = index;
do
{
    // Attempt to rent from the bucket.  If we get a buffer from it, return it.
    buffer = _buckets[i].Rent();
    if (buffer != null) return buffer;
} while (++i < _buckets.Length && i != index + MaxBucketsToTry);

// The pool was exhausted for this buffer size.  Allocate a new buffer with a size corresponding to the appropriate bucket.
buffer = new T[_buckets[index]._bufferLength];
```

I have edited the code for brevity (removing logging and some spacing), however the basic flow is

1. Find the bucket we should get the buffer from
2. Check if there is a free buffer in this bucket
3. If there is no free buffer check in the next highest bucket and repeat 2 (for a max of 2 buckets)
4. If no buffer is available still then return a new buffer

So far so good, we can see that it is going to be more expensive than just a "new" operation however we knew that already. Now we can take a look inside the buckets Rent method.

``` csharp
bool lockTaken = false, allocateBuffer = false;
try
{
    _lock.Enter(ref lockTaken);
    if (_index < buffers.Length)
    {
        buffer = buffers[_index];
        buffers[_index++] = null;
        allocateBuffer = buffer == null;
    }
}
finally
{
    if (lockTaken) _lock.Exit(false);
}

if (allocateBuffer) buffer = new T[_bufferLength];
return buffer;
```

So here we take a spin lock for a short time, check if there is a free buffer in the bucket and if that slot is empty (we have never allocated for it yet) we will just create a new buffer.




 There is nothing wrong with this method however you should be aware of a few things about this pool and why it might not be the best approach for your application.




## Enter IOwnedMemory<T>