---
title: Pipelines formally known as channels - Pipes Part 2
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

![Locks](/images/pipelines/lock.jpg)

## Get to the security already!

Onto the SSL/TLS story, when I first started to look at this I probably knew as much as the average developer. A server
has a certificate that is certified by a known certificate authority. 

It has a public/private key and it is used by the client
to make sure the server is who they say they are. This is also then used to perform some sort of "handshake" to agree on a
key (well actually a set of keys) used to then do all communication from then on. This enables authentication of the server
(and the client if using client certs although that is less common) and to secure further communications... easy right?

Well yes and no, my basic assumptions were correct, but I figured lets get some background before I just off and port SslStream.
I thought SslStream has been around for sometime, had Xplat added to it and made in another time. 

Ben Adams seemed to hint [https://github.com/dotnet/corefx/issues/11826](well more than that) at the "allocaty" nature and maybe a fresh look was needed.

So then where to start? I always find that it's best if you want
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
upside was that SslStream is opensource so I decided that it was time to take a look. 

What I found (and is the pattern for
OpenSsl as well) is that you have a set of security credentials or a context and this contains information like a certificate
and your settings. This is then used to generate a set of session keys etc and then that session is essentially detached and not
reliant on the initial credentials.

This is important and one of the reasons that I still believe that SslStream is not the way forward for server side TLS (I am going to 
stop refering to SSL from now on, it's broken). Sslstream has the concept of making a stream, passing in the upchannel or wrapped stream
and the certificate etc and carrying on. This leads to a major inefficency, making a authorization context for every conenction.
Even if you could pool and reset the SslStream you would still have the authorization per connection. 

There is really no way around this but changing the API heavily. As SslStream is used in a lot of places and needs to maintain backwards
compatiblity these changes to the API really aren't feasible. So in summary this is how things are, and where I thought they should be

![Diagram of SslStream vs the proposed](/images/pipelines/concept.png)

So with that in mind I produced a first cut, and after a few reviews of the PR, I was close to having the correct spacing spelling and
code layout :) I ended with code that looked something like this

``` csharp
var cert = new X509Certificate(_certificatePath, _certificatePassword);
var serverContext = new SecurityContext(pipelineFactory : factory, hostname: host, isServer : true, certificate : cert);

private async Task<IPipelineConnection> CreateWrapper(IPipelineConnection connection)
{
   var session = serverContext.CreateSecureChannel(connection);
   await session.HandShakeAsync();
   return session;
}
                
```

That obviously is psuedo code but the concept is that you declare the SecurityContext once for your server application and then it is only the 
session that is created per connection. This should reduce the overhead per connection, and allocations both in the GC code space, but also 
the hidden cost of unmanaged memory (either by OpenSsl or SSPI), cracking open an PKI is by no means free, in memory or compute space. So using SSPI 
I had some code that looked like the following for building the credentials for SSPI

``` csharp
var creds = new SecureCredential()
{
  rootStore = IntPtr.Zero,
  phMappers = IntPtr.Zero,
  palgSupportedAlgs = IntPtr.Zero,
  cMappers = 0,
  cSupportedAlgs = 0,
  dwSessionLifespan = 0,
  reserved = 0,
  dwMinimumCipherStrength = 0, //this is required to force encryption
  dwMaximumCipherStrength = 0,
  version = SecureCredential.CurrentVersion,
  dwFlags = flags,
  certContextArray = IntPtr.Zero,
  cCreds = 0
};
```

Which pretty much says, go with what the OS recommends. Overriding an admins settings for their computers is often not a nice thing to do. It generally
just annoys people. I was very concious of not allocating it had been drummed into me that it was evil and so for the next bit I used a little hack, the secure
credentials expects an array of pointers to certificates, seeing as at this point I only support one this

``` csharp
//pointer to the pointer
IntPtr certPointerPointer = new IntPtr(&certPointer);
creds.certContextArray = certPointerPointer;
```

got me a pointer to a pointer... basically an array of 1 without allocating anything at all. So that is basically all the SecurityContext does,
cracks the certificate (if it is needed) sets up some credentials and stores the basic settings for each connection. 

![Meat](/images/pipelines/meat.jpg)

## The real meat of the code

The real meat of the code
starts in the SecureContext. When a new connection or Pipeline needs to be wrapped in a secure TLS blanket you simple call
securityContext.CreateSecurePipeline(pipeline); this will return an ISecurePipeline. This implements IPipelineConnection allowing you to then chain
it down a tree (eg an HTTP, Websockets or your protocol next). Internally what this call does is create a new SecurePipeline. 

This class is
a generic with SecurePipeline<T> where T : ISecureContext. So the SecurePipeline is common to OpenSsl, SSPI, and maybe SecureTransport(Osx)
and others in the future. This class does most of the Pipelines handling, the byte reading loop etc. The most important public methods are

``` csharp
public Task<ApplicationProtocols.ProtocolIds> HandShakeAsync();
public IPipelineReader Input => _outputChannel;
public IPipelineWriter Output => _inputChannel;
```

I have to agree with others here "HandShake" isn't really a verb, so thinking about it ShakeHandsAsync() seems better here. 

![Discussions](/images/pipelines/argue.jpg)

## API design is part art, part.. nah it's an art

This is something
I have learned working on this, API design is hard. It doesn't just happen, it is thought over and discussed the number of hours I have seen being
burnt by very smart people from midnight -> 3am on the naming and layout of just the API. 

The implementation is often agreed on, maybe with tweaks
to come but the semantics, should this method throw an error in situation x, or should it just return an empty result? What do other parts of .net do?
What would the end user expect? It has given me a new found respect for all those BCL's that I just start using and 9/10 times never needing to
look for more documentation than the intellisense tool-tips. 

Anyway I digress, we have those methods and a few others that don't really matter
at this point. They are the normal pipeline methods (although you can still see the channels lineage at this point) the consumer of the SecurePipeline
writes to the Output and consumes from the input (Pipelines is written so that the consumer of your Pipeline reads those two properties from their
perspective, another homage to the usability for the API consumer).

As a consumer you call HandShakeAsync or just start writing, internally if the handshake hasn't been performed it will be on the first write. That is for
convenience, however if you are writing an upper library that has a lot of initialization cost or you want to redirect your user depending on the handshake
results you may await the handshake directly.

Internally the handshake method simply does 

``` csharp  
return _handShakeCompleted?.Task ?? DoHandShake();
```

which will return straight away if the handshake has been done before, using a cached task (via a task completion source) or it will call the DoHandShake method.
That method is where the real work starts with both Pipelines and the TLS libraries.

``` csharp 
_handShakeCompleted = new TaskCompletionSource<ApplicationProtocols.ProtocolIds>();
if (!_contextToDispose.IsServer)
{
    //If it is a client we need to start by sending a client hello straight away
    await _contextToDispose.ProcessContextMessageAsync(_lowerChannel.Output);
}
```
This checks to see if we are running in client mode, if we are we should initiate the handshake by calling the underlying library with no input data
in SSPI this is a call to InitializeSecurityContextW and in OpenSsl this is a call to SSL__do__handshake (for some versions more on that in a future post).
All of these differences are hidden behind the ProcessContextMessage so in reality the SecurePipeline doesn't care. There are a few things that need to be done
at this point (and of course they are done in totally different ways on SSPI than OpenSsl). There is something that some of you may know of and some may not

## ALPN - Application Layer Protocol Negotiation

This is an extension to TLS, so maybe I have got ahead of myself a little here lets go back to how a handshake works with TLS. In a full handshake (something we
will revisit later) you have the following flow

![SSL Handshake](https://upload.wikimedia.org/wikipedia/commons/a/ae/SSL_handshake_with_two_way_authentication_with_certificates.svg)

Now with Http/2 Google and others realised that there was a lot of back and forward going on there, and then the first thing the client does once they have
connected is say "hey upgrade me to http/2" and they have to wait for a response before any real work can be done. 

In order to streamline this (we are already doing
a TCP handshake under the hood as well, although that can be reduced but that is another topic) the idea is that the client can send along a list of protocols it 
supports during the TLS handshake and that can be negotiated while we negotiate ciphers, key strength and all of the other details.

![Crying](/images/pipelines/crying.jpg)

## Now the problems come ...

My problem with getting this to work
for SSPI, the documentation basically didn't mention it (not the on-line stuff anyway) other than a couple of announcements saying "hey SSPI and SecureChannel support this!"
of course SslStream didn't support it so that was no help either. 

But after much digging, hacking and reading obscure websites I found the answer to my problems when you call
AcceptSecurityContext (for the server side) or InitializeSecurityContextW(for the client side) you have to pass in buffers with your data you want to send to be processed.

In the case of a client normally you would send a null pointer for the first request, however with ALPN you need to send in a single buffer with your Extension information.

On the server side you would normally send in two buffers, one being the data from the client and the second an empty token, used for messages to send to the client regarding
issues with the connection, however in this case we use what would normally be an empty buffer as the second buffer to pass in the ALPN information again.

In order to build the ALPN information SSPI has no real information or help. So it was back to the [IETF RCF](https://tools.ietf.org/html/rfc7301). So from looking at that I
found the data structure that was needed, it's a struct with a extension id, then a length of the following list and then a list of size prefixed strings for the supported
protocols. 

Once again this was a win for the design of having a single "context" that creates each collection as it allows a single fixed buffer to be built at the context
level and then there are no allocations per connection as we reuse that buffer.

Phew.. all of that just to kick off the first handshake message! The rest of the handshake was a little more straight forward. It basically was a loop that looked something like

``` csharp
while (true)
{
    var result = await _lowerChannel.Input.ReadAsync();
    var buffer = result.Buffer;
    try
    {
        if (result.IsCompleted)
        {
            new InvalidOperationException("Connection closed before the handshake completed");
        }
        ReadableBuffer messageBuffer;
        TlsFrameType frameType;
        while (TryGetFrameType(ref buffer, out messageBuffer, out frameType))
        {
            if (frameType != TlsFrameType.Handshake && frameType != TlsFrameType.ChangeCipherSpec)
            {
                throw new InvalidOperationException("Received a token that was invalid during the handshake");
            }
            await _contextToDispose.ProcessContextMessageAsync(messageBuffer, _lowerChannel.Output);
            if (_contextToDispose.ReadyToSend)
            {
                _handShakeCompleted.SetResult(_contextToDispose.NegotiatedProtocol);
                return await _handShakeCompleted.Task;
            }
       }
    }
    finally
    {
        _lowerChannel.Input.Advance(buffer.Start, buffer.End);
    }
}
```

There might seem like a lot going on there but if we break it down it's pretty simple... 

First we await the underlying connection for some data, once we have the data we check that it's
not empty and completed, if so the connection has finished so it's time to exit the loop, we will fail the handshake higher up and clean up anything we have created so far. 

Next I made my
own TLS Frame handler, I could have cheated here and just passed the data straight into the underlying library, however on partial frames which happen often we would have just had to do
a series of interop calls, and in the worst case an allocation and a copy. 

The TLS frame format is pretty simple to follow and not encrypted only the contents are so it makes sense to 
do that as close to the data with the smallest overhead possible. At the same time make our handling of bad frames and data a lot more straight forward because we would now know that
any error back from the underlying libraries would not be a "false" error saying we just need the rest of the frame. 

In a security library it's always best to fail fast (but not too fast, I will explain that also later).

The rest of the code goes like this. If we have an incomplete frame, that is okay we will loop back around (and if it is incomplete and the connection has "completed" eg closed we will 
throw an exception and exit out). Otherwise we check to make sure it is one of the valid handshake messages, an actual handshake or a ChangeCipherSpec (used at the end of the handshake
to say anything from now on must be encrypted with our cool new keys). 

We send the frames into the underlying libraries and they will return either "handshake done", "I am waiting for more data" or throw an exception.

Once we are done, we set the result of the task completion source to the protocols we negotiated (which will be zero if there was no ALPN) and return back to the main program.

It feels like that was a long winded explination, but believe me it was an even longer journey and many late nights from me... but now we had a working handshake and to confirm it, I 
opened 443 on my firewall, punched it through to my laptop and fired up a CDN Http/2 test page

![Success](/images/pipelines/handshakepassed.jpg)

Next time, if you aren't bored out of your mind I will discuss the interesting world of telling security people you wrote a new security
library as a no-one on the internet, learning about threats, and of course Xplat-OpenSsl and some benchmarks thrown in for good measure.

