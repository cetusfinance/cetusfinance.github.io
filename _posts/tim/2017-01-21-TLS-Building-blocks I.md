---
title: "TLS Building Blocks I"
excerpt: "Getting on with it - Pipes Part 2&#178;. Hashing and HMAC"
category: "tim"
header:
  overlay_image: buildingblocksi/lego.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: buildingblocksi/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl, cng, leto, hash, hmac]
---

### Prior Reading

If you are returning and have read the previous posts, I am amazed you have hung around for so long but thank you. If not
and you want to gain some understanding before continuing here are the previous articles in the series. 

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)
* [Not all secrets are created equal](https://cetus.io/tim/Not-all-secrets-are-created-equal/)

## Building the primitives

Before we can start on all of the exciting bits (okay I might be the only one that finds TLS states exciting but you are reading this
blog so I will assume some level of interest) we need some basic components that will make up the library. 

* Hash and Hash message authentication code (HMAC[^1]) functions
* Symmetrical Ciphers (I tend to call them bulk ciphers as this is their primary role)
* Asymmetrical Ciphers and Certificates
* Key Exchanges

The general approach taken with all of these is a provider model. The reason for this is as described in earlier documents, we want
the library that provides these functions to be replaceable. To this end I will attempt to write both an OpenSSL 1.1 and a Crypto
Next Generation (CNG) version of all of the parts. This will help prove out the ability to slot in new providers and keeps the 
design from being becoming tied to a single underlying library.

## Hash

We will be using hashes and HMAC in a number of areas of the handshakes. They are used to create signatures for certificate messages,
in the "Finished" messages we use them to authenticate that no one has been tampering with the handshake (to say downgrade the ciphers).
The last but probably the most important use is in the secret and key generations.

Both OpenSSL and CNG use a similar interface for HMAC and hash functions with the HMAC just requiring an extra key parameter. They
both two modes

* A single method you pass all of your data to and it returns the result
* Creating a hash object with state that you can add data to over multiple operations and obtain a result

The advantage of the first is that you have a single interop call. As well as that you don't have to hold as much memory for
the state of the current operation. If we can we should use a single call. However TLS often needs a running hash say of all
the messages that we have currently sent/received in the handshake. This is to provide a final value that we can sign to ensure 
there was 
no tampering. One approach I have seen is to hold an entire history of those messages in a buffer, but it is much more efficient
for us to just hold the current state of the hash object and throw away the messages as soon as possible.

After analyzing the use more I realised that the main case for a single operation without having to keep a running hash is when 
we are using HMAC. In fact we never use the HMAC in a long running fashion. So with that in mind the "Hash Provider" interface looks
like this

``` csharp
public interface IHashProvider
{
    IHashInstance GetHashInstance(HashType hashType);
    unsafe void HmacData(HashType hashType, void* key, int keyLength, void* message, int messageLength, void* result, int resultLength);
    int HashSize(HashType hashType);
    void Dispose();
}

public enum HashType : byte
{
    SHA256 = 4,
    SHA384 = 5,
    SHA512 = 6,
}
``` 

The first thing you might notice is the pointers being exposed in the interface. If this makes you wince in a C# interface you are not
alone. The problem I currently have is that I cannot retrieve a pointer and length efficiently from the new Span.. not yet. Apparently
with C# 7 and the further integration of the Span<T> this will be possible and I will be more than happy to remove the exposed pointers
in the interface. Ideally I would like it to be

``` csharp
void HmacData(HashType hashType, Span<byte> key, Span<Byte> message, Span<byte> result);
```

The hashtype enumeration is from the IETF[^2] with values that I don't feel the need to support (MD5, SHA1) removed. They may
need to make a comeback if I am persuaded backwards compatibility is really worth it, I am a long way from that yet. 

So that is it for the provider, next is the "HashInstance" which is our long running stateful hash object.

``` csharp
public interface IHashInstance : IDisposable
{
    int HashSize { get; }
    unsafe void InterimHash(byte* hash, int hashSize);
    unsafe void HashData(byte* message, int messageLength);
    unsafe void FinishHash(byte* hash, int hashSize);
}
```

These operations are pretty basic and the pointer comments from above apply here. We have an operation that can be called multiple
times to add data to the ongoing hash (HashData). If you want to get a hash of all the data added until now but continue to use
the hash you need to call "InterimHash", and FinishHash does what it says on the tin.

Now we can take a look at the internal implementation which is pretty simple. OpenSSL first

``` csharp
//Creates our hash instance, GetHashType is simply a lookup case/switch statement
public IHashInstance GetHashInstance(HashType hashType)
{
    int size;
    IntPtr type = GetHashType(hashType, out size);
    var ctx = EVP_MD_CTX_new();
    ThrowOnError(EVP_DigestInit_ex(ctx, type, IntPtr.Zero));
    return new HashInstance(ctx, size, hashType);
}
//These are both methods from the hash instance. We are not required
//to provide anything more complex than this as I will explain shortly
public unsafe void HashData(byte* buffer, int bufferLength)
{
    ThrowOnError(EVP_DigestUpdate(_ctx, buffer, bufferLength));
}
//Both OpenSSL and CNG "destroy" the state when you call "final"
//or finish on a hash object. Both however provide a way to duplicate
//the current state, so we duplicate and finish the copy allowing us
//to continue using the original
public unsafe void InterimHash(byte* buffer, int length)
{
    var ctx = EVP_MD_CTX_new();
    try
    {
        ThrowOnError(EVP_MD_CTX_copy_ex(ctx, _ctx));
        ThrowOnError(EVP_DigestFinal_ex(ctx, buffer, ref length));
    }
    finally
    {
        ctx.Free();
    }
}
```

CNG is pretty much the same DigestInit becomes BCryptCreateHash, DigestUpdate becomes BCryptHashData and so on. The code
is over in the repo if you are interested.

The last part is the extension methods. We will be getting data we want to hash in a number of different formats (unmanaged
point/lengths, readable buffers, arrays). Rather than implementing them on each object, or requiring an abstract base class
(which I tend to avoid) we have some extension methods. I am going to talk through a couple of these as they introduce important
concepts from the new classes in corefxlabs.

``` csharp
public static unsafe void HashData(this IHashInstance hash, Memory<byte> dataToHash)
{
    GCHandle handle;
    var ptr = dataToHash.GetPointer(out handle);
    try
    {
        hash.HashData((byte*)ptr, dataToHash.Length);
    }
    finally
    {
        if (handle.IsAllocated)
        {
            handle.Free();
        }
    }
}
```

Let me roughly explain my perspective of the memory class. It basically encapsulates memory so that it can be passed around, leased
returned, pooled and so on. It can contain an array as it's backing store or a pointer and length. This is useful because the pipelines
on either side could be be sending us buffers from managed or unmanaged memory, pinned or unpinned. The problem then comes that we need a pointer no
matter where the memory comes from. The memory class contains a "TryGetPointer" and a "TryGetArray". So we try to get the pointer and if it
fails we need to get the array and pin it. Because we can't pin it inside a fixing block to encapsulate it in a method we then pin it using a GC handle
which of course we have to check if we allocated and if we did free once we are done. Now ideally this would take a Span<T> however
there is currently no way to get the pointer back out. Also the "out" if memory serves me correctly will cause havoc with inlining and the JIT but I 
can't justify have the whole if(m.TryGetPointer()) nesting in every method that I need to unwrap. 

Now I have been told that when corefxlab is using C# 7 this will be cleaner, at the least I can use a ValueTuple to return and decompose the GetPointer
extension method and remove the "out". 

The other important one is

``` csharp
public static void HashData(this IHashInstance hash, ReadableBuffer dataToHash)
{
    foreach (var m in dataToHash)
    {
        hash.HashData(m);
    }
}
``` 

This example is a lot more straight forward. But it shows that a readable buffer in essence is a linked list of Memory<T>.

That pretty much covers hashing and HMAC how we have seperate implementations for two different crypto libraries, 
and how we abstract away common code with extension methods.

[^1]: [Hash message authentication code from Wikipedia](https://en.wikipedia.org/wiki/Hash-based_message_authentication_code)
[^2]: [RFC 5246 Section 7.4.1.4.1 Hash types](https://tools.ietf.org/html/rfc5246#section-7.4.1.4.1)

