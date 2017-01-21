---
title: "Not all secrets are created equal"
excerpt: "Getting on with it - Pipes Part Tre. What secrets do we have and how should we store them?"
category: "tim"
header:
  overlay_image: notallsecretsareequal/spy.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: notallsecretsareequal/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl,secrecy, leto]
---

### Prior Reading?

This series is likely to end up with a number of parts, I am going to just go ahead and assume prior knowledge.
Everything you should need to know to get up to speed is in the previous posts.

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)

## The key exchange

The initial part of TLS is comprised of exchanging keys. We want in some way to agree on random information, that both parties
can then use to generate keys without someone listening in being able to produce the same key. Why don't we just use the Public key to
encrypt data? Firstly we don't have a public key from the client, but they could generate one and send it to us right? Well one major 
factor is Public/Private key algorithms are slow and CPU intensive. Symmetric algorithms are far faster at encrypting and decrypting
bulk data.

The first part of a TLS handshake agrees on the ciphers and other properties of the connection, but the primary
task is to exchange the material to generate the keys in secret. The way TLS is structured each side has a separate key for writing, 
and therefore for reading as well. We don't directly exchange the keys, instead we use random data from both sides mixed with 
"context information" to generate the secrets using a pseudo random function(PRF)[^1]. 

By design the PRF is seeded with the various amounts of random data from the client and server and then will produce a random but
predictable sequence of numbers that should go on for a long enough time before repeating that we can consider it continuous. This
is useful because our ciphers can require varying amounts of data so we don't have to return to the client with the cipher selected 
and then wait for another message with the correct amount of data for a key.

The client sends the server some random data (32 bytes of it) and the server sends the same size random data back in the server
hello. At this point none of the communication is not encrypted and anyone listening in has both sets of random data. So the client generates
some more random data, encrypts it with servers public key and sends it over. This is called the pre-master secret. The server decrypts
it with it's private key. Now both sides have the randoms and the pre-master secret but anyone listening in is missing the pre-master secret.

From there we go through the process of generating the master secret and use that to create the key information (which I won't go into here as the mechanism isn't
important for our discussion)[^2]. 

![listening hare](https://cetus.io/images/notallsecretsareequal/listen.jpg)

## Who's listening?

The problem with the above procedure is that if someone records the entire conversation and later gains access to the private key of the server
they can recreate the master secret and therefore the keys. That means they can decrypt the entire conversation at any time in the future.
This key could be obtained via legal or illegal means, it makes no difference. Heart bleed[^3] is an example
of when a servers private key could have been leaked, leaving all historical conversations compromised. 

Enter "perfect forward secrecy" and Diffie Hellman. This algorithm has an interesting history that goes a long way back. The important property is
that we can generate a public/private key pair with the other party doing the same. This is generated for each connection, both sides then share their
public key. Each side "mixes" their private key and both public keys and comes out with the same data. Even though we don't know the private key
from the other side. Wikipedia has a wonderful explanation of this so I won't repeat it here[^4]. By generating the public/private pair for a connection
and as long as we dispose of these pairs correctly (and keep the output and master secret safe) in the future if we give up the servers private key
no one can decrypt the conversation. Not even the client or the server could do so. This is what is called perfect forward secrecy.

![Ticket](https://cetus.io/images/notallsecretsareequal/sheep.jpg)

## Resumption?

If all of this sounds like a lot of back and forward and expensive calculations then you would be correct. Most of the overhead and increased
latency associated with TLS connections is caused by the initial handshake and key exchange. So a "session id" was introduced. This is a opaque number
that is assigned to a clients connection. The client stores this along with the master secret and so does the server. When a client reconnects
within a validity period (defined by the IETF as no more than 24 hours) it sends the session id. The server then looks up the information from
the previous handshake. This allows us to skip a lot of the expensive part of the handshake. This has a couple of downsides, first the server has
to store all of these connections, and next we have to rely on the server storing them correctly otherwise sometime in the future someone could
decrypt our previous conversation. So the server needs to ensure this information is not persisted and ideally is destroyed after the validity period.
That means keeping them in memory, but more than that we need to keep them in memory and ensure that the memory is not paged to disk. In fact this is the 
same type of storage we require for the key and the master key during the connection.

The other problem with the above scenario is web farms. If you have 10 front end servers with clients connecting to anyone of them, they all need
to have access to the database of session id's and secrets. It might be possible but what if you now have 1000 front end computers across the world
you need a distributed database of session id -> secret mappings and it needs to be fast (you might connect and then use the session id to open another
connection straight away) and secure.

![Ticket](https://cetus.io/images/notallsecretsareequal/ticket.jpg)

## Resumption Take 2 - Session Ticket's

Faced with all of these issues a new method for session resumption was designed. Here the server collects all of the data it needs to resume the session. It
might in the simplest form be the information it would have stored in the above resumption method. It then encrypts this with a key that it never sends out
over the wire. When a client wants to reconnect it can just send back this encrypted block, the server decrypts and validates the information and then
resumes the session. 

This has a number of nice properties, the key used to encrypt the session state can be shared amongst all machines in a web farm.
This means that a client can connect to machine A and resume with machine B without a complex database being required. You can also support an almost
infinite number of clients because the server isn't storing anything for each of these clients. There are problems with this of course. The keys
need to be rotated if they aren't then we are breaking "perfect forward secrecy" and some implementations of this have serious flaws[^5]. Most implementations
can take an external source for the key that encrypts the data. However the implementations normally only cycle the key on a restart. 

This is fine if you 
write terrible software and it needs to restart often, but most web servers can stay up for days or weeks before being restarted. So if we want to implement
session resumption via tickets we need a way to generate and rotate keys. Ideally we should have a provider model so that people could provide their own implementations
if they wish. The initial attempt will roughly be in line with how twitter wrote session resumption[^6]. 

## Enter TLS 1.3

We have yet to discuss TLS 1.3 other than a small mention. It has a similar set of secrets as above. The key differences are

* The simple key exchange case has been removed - perfect forward secret is the only option
* There are multiple "secrets" created on a schedule
* These secrets are then used to make contextual secrets that have hashes of the current conversation mixed in
* This is then used to generate keys, either side can ask for keys to be regenerated according to a schedule
* The PRF has been replaced with a HMAC key derivation function (HKDF[^7]) that splits the tasks of extract and expand that is done by the PRF

![Stock](https://cetus.io/images/notallsecretsareequal/stock.jpg)

## Taking stock

So all of this means we need a way of resuming sessions and rotating keys, and storing secrets. It may not be clear but we have two types of secrets we
need to store. Ephemeral secrets (session keys, master/premaster and schedule secrets) and permanent secrets. 

The most important of these is the permanent
secrets (the private keys of the public/private pair belonging to the organisation). It is beyond the scope of the library to take care of these, ideally
this should be stored and used from a hardware device like a trusted platform module[^8] but at worst the operating system or an external library that
has been tested and checked should do the heavy lifting. To this end the library will delegate the certificate use to either OpenSSL (on Unix)
, SecureTransport (on Apple) and Cryptography Next Generation (on Windows). 

The next secret is the Ephemeral ones, we need quick access and the protection
required is somewhat less. If they are stolen it would not be good but it would only disclose the current in flight sessions. 
What we want to avoid is the data being moved around without wiping so garbage collected memory is off the table. The garbage collector could at anytime
move our data and now there is a copy of our data that we can't control or wipe. The next problem is the operating system paging this data to disk,
so we need a way of stopping that happening. There is another way it could be persisted to disk - sleep/hibernate and VM pausing. There is not a lot 
we can do about this but it is a good reason to encrypt your disks. Encrypting your disk still requires not paging however because a running application
can access the encrypted disk. 

![Sand](https://cetus.io/images/notallsecretsareequal/sand.jpg)

## Enter the Ephemeral buffer pool - show me the code

We will design our buffer around the OwnedMemory class from corefxlabs to make it work with the new slices. Because it requires some interop we will need
platform specific implementations.

``` csharp
public class EphemeralBufferPoolWindowOrUnix:IDisposable
{
    private IntPtr _memory;
    private int _bufferCount;
    private int _bufferSize;
    private ConcurrentQueue<EphemeralMemory> _buffers = new ConcurrentQueue<EphemeralMemory>();
    private UIntPtr _totalAllocated;
        
    public EphemeralBufferPoolWindowOrUnix(int bufferSize, int bufferCount)
    {
        if (bufferSize < 1) throw new ArgumentOutOfRangeException(nameof(bufferSize));
        if (bufferCount < 1) throw new ArgumentOutOfRangeException(nameof(bufferSize));
```
That is simple and common to all the impmentations. We grab a block of memory and then store availabe buffers in the concurrent queue. We initialize the 
memory like this on Windows

``` csharp
SYSTEM_INFO sysInfo;
GetSystemInfo(out sysInfo);
var pages = (int)Math.Ceiling((bufferCount * bufferSize) / (double)sysInfo.dwPageSize);
var totalAllocated = pages * sysInfo.dwPageSize;
_bufferCount = totalAllocated / bufferSize;
_bufferSize = bufferSize;
_totalAllocated = new UIntPtr((uint)totalAllocated);

_memory = VirtualAlloc(IntPtr.Zero, _totalAllocated, MemOptions.MEM_COMMIT | MemOptions.MEM_RESERVE, PageOptions.PAGE_READWRITE);
VirtualLock(_memory, _totalAllocated);
```

Because we are locking the pages from being paged we need to allocate whole pages. First we get the page size from the OS and find the minimum number
of pages we need to store the requested number of blocks * size. We then update the number of blocks that are available and allocate the memory using 
VirtualAlloc. This means we are in full control of the memory and the GC won't touch our buffers. We tell the OS that it should both reserve and commit
the memory allowing us to use it straight away. Finally we call VirtualLock which stops the OS from paging it to disk. On Unix the same procedure takes
place with the interop slightly changed

``` csharp
// Page size
var pageSize = SysConf(SysConfName._SC_PAGESIZE);
// Allocate one large block of memory
_memory = MMap(IntPtr.Zero, (ulong)_totalAllocated, MemoryMappedProtections.PROT_READ | MemoryMappedProtections.PROT_WRITE, MemoryMappedFlags.MAP_PRIVATE | MemoryMappedFlags.MAP_ANONYMOUS, new IntPtr(-1), 0);
// Lock the pages into memory
MLock(_memory, (ulong)_totalAllocated)
```

Renting a block simply requires dequeuing a buffer. Returning a rented block requires an extra step on windows it looks like this

``` csharp
public void Return(OwnedMemory<byte> buffer)
{
    var eBuffer = buffer as EphemeralMemory;
    if (eBuffer == null)
    {
        Debug.Fail("The buffer was not ephemeral");
        return;
    }
    Debug.Assert(buffer2.Rented, "Returning a buffer that isn't rented!");
    eBuffer.Rented = false;
    RtlZeroMemory(eBuffer.Pointer, (UIntPtr)buffer2.Length);
    _buffers.Enqueue(buffer2);
}
```

So when we return a buffer we zero the memory before putting it back for reuse. This allows us to destroy secrets as soon as possible. On Unix the procedure
is the same except we use Memset. When we dispose (or finalize) the pool we zero the buffer out and then release the pages back to the OS.

So that is all for now. The session resumption and key rotation will be a topic for another post. This can all be a little dry and the code is sparse
but I feel it is an important topic and integral part of the library design.

Next up ..... ~~TLS 1.3 we begin~~ I had to revise this, I realised there was just too much code that the statemachines use
to simply skip it. I will explain the primitives instead first.

[^1]: [Pseudo Random Numbers from Wikipedia](https://en.wikipedia.org/wiki/Pseudorandom_function_family)
[^2]: [Calculating the premaster secret - TLS 1.2](https://tools.ietf.org/html/rfc5246#section-8.1)
[^3]: [The heart bleed bugs very own website](http://heartbleed.com/)
[^4]: [Diffie–Hellman key exchange from Wikipedia](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)
[^5]: [Botching session resumption](https://timtaubert.de/blog/2014/11/the-sad-state-of-server-side-tls-session-resumption-implementations/)
[^6]: [Twitter's implementation of session resumption](https://blog.twitter.com/2013/forward-secrecy-at-twitter-0)
[^7]: [RFC5869 HMAC key derivation function](https://tools.ietf.org/html/rfc5869)
[^8]: [Trusted platform module](https://en.wikipedia.org/wiki/Trusted_Platform_Module)