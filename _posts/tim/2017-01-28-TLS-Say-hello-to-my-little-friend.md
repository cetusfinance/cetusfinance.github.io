---
title: "TLS Say Hello to my Little Friend"
excerpt: "Getting on with it - Pipes Part 2&#179;. 1.3 Client Hello"
category: "tim"
header:
  overlay_image: sayhello/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: sayhello/teaser.jpg
tags: [pipelines, tls1.3, tls, ssl, leto]
---

### Prior Reading

The series is going to assume you understand the terms introduced previously. 

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)
* [Not all secrets are created equal](https://cetus.io/tim/Not-all-secrets-are-created-equal/)
* [Building Blocks I - Hash/HMAC](https://cetus.io/tim/TLS-Building-blocks-I/)
* [Building Blocks II - Bulk Ciphers](https://cetus.io/tim/TLS-Building-blocks-II/)
* [Building Blocks III - Key Exchange](https://cetus.io/tim/TLS-Building-blocks-III/)
* [Version Detection](https://cetus.io/tim/TLS-Version-Detection/)

## The state of it all

So now we have a state machine for our target version (TLS 1.3). We can take a quick look at some of the methods common to the state machines

``` csharp
void SetClientRandom(ReadableBuffer readableBuffer);
void StartHandshake(ref WritableBuffer writer);
void HandleAlertMessage(ReadableBuffer readable);
Task HandleHandshakeMessage(HandshakeType handshakeMessageType, ReadableBuffer buffer, IPipelineWriter pipe);
// Other properties
```

These are the key methods, the secure connection passes in handshake messages and alerts and lets the state machine deal with it as it sees fit.
The start hand handshake will do nothing on the server state machines and on the client will trigger the writing of the client hello.

Looking inside the TLS 1.3 implementation we get this

``` csharp
public override async Task HandleHandshakeMessage(HandshakeType handshakeMessageType, ReadableBuffer buffer, IPipelineWriter pipe)
{
    WritableBuffer writer;
    switch (State)
    {
        case StateType.None:
        case StateType.WaitHelloRetry:
            if (handshakeMessageType != HandshakeType.client_hello)
            {
                Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.unexpected_message, $"State is wait hello retry but got {handshakeMessageType}");
            }
            Hello.ReadClientHelloTls13(buffer, this)
            //Other logic in the handshake
            return;
        case StateType.WaitClientFinished:
            if (handshakeMessageType != HandshakeType.finished)
            {
                Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.unexpected_message, $"Waiting for client finished but received {handshakeMessageType}");
            }
            Finished.ReadClientFinished(buffer, this);
            //Other logic in the finish part of the handshake
            break;
        default:
            Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.unexpected_message,$"Not in any known state {State} that we expected a handshake messge from {handshakeMessageType}" );
            break;
    }
}
```

Now there is logic that I have cut out but you can see the states are pretty simple. Anyone looking at the code should be able to see that we can be
in three states when we get a handshake message, and that we expect a client hello or finished message respectively. 

What this also illustrates is that
TLS 1.3 has a single round trip (sometimes even Zero before sending data which we will get to later) or at worst a two round trip handshake. This is part of the 
reduced initial latency promised by the new version.

We also see here that at any point an unexpected issue occurs we throw an alert exception and exit, we never try to recover or resume that way we are less open to security flaws.

## Say Hello!

Now to good bit (my definition of what is interesting might be a little different to everyone else). We will revisit a simplified diagram of the hello message

![Client Hello Structure](https://cetus.io/images/sayhello/support.png)

In TLS 1.3 we are going to ignore most of the mandatory data as it has been marked legacy, and only a couple of extensions will be used
 to get up and running which we will go through below and implement the other extensions later.

The read hello is similar to the version select we saw earlier however this time we will stop at the cipher suite list.

``` csharp
var ciphers = BufferExtensions.SliceVector<ushort>(ref readable);
if (connectionState.CipherSuite == null)
{
    connectionState.CipherSuite = connectionState.CryptoProvider.GetCipherSuiteFromExtension(ciphers, connectionState.Version);
}
```

Here we introduce two concepts, the CryptoProvider and the CipherSuite. The CryptoProvider is a container for our HashProviders, KeyExchangeProviders,
and BulkExchangeProviders we discussed previously. It is in here that we can load platform specific implementations like so

``` csharp
if(RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
{
    _keyShareProvider = new KeyExchange.OpenSsl11.KeyshareProvider(); 
    _hashProvider = new Hash.OpenSsl11.HashProvider();
    _bulkCipherProvider = new BulkCipher.OpenSsl11.BulkCipherProvider();
}
else if (RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
{
    _keyShareProvider = new KeyExchange.Windows.KeyshareProvider();
    _hashProvider = new Hash.Windows.HashProvider();
//// and so on
```

This allows us to use the provider that is appropriate for the platform we are operating on. The next part is the supported cipher schemes. These
are defined in the spec for the TLS version [^1] (although they are updated later when new ciphers are accepted or old ones are found to be broken).
We will implement the first 3 algorithms initially. They are

1. TLS__AES__128__GCM__SHA256
2. TLS__AES__256__GCM__SHA384
3. TLS__CHACHA20__POLY1305__SHA256

A quick break down of this is that the first part is TLS and doesn't say anything. The next part is the BulkCipher algorithm so we support AES and 
CHACHA20 (although CHACHA20 isn't currently available in CNG that I can find). The next part is the block chaining mode, and finally is the Hash
algorithm we will use to create a digest of the handshake messages to make sure that no one altered any of the handshake messages to force a bad
or weak negotiation.

We order them in the priority for the server and when we get the list from the client we find the highest priority cipher suite that the client supports.

``` csharp
var list = GetCipherSuites(version);
if (buffer.Length % 2 != 0)
{
    Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.illegal_parameter, "Cipher suite extension is not divisable by zero");
}
var numberOfCiphers = buffer.Length / 2;
var peerCipherList = stackalloc ushort[numberOfCiphers];
for (var i = 0; i < numberOfCiphers; i++)
{
    peerCipherList[i] = buffer.ReadBigEndian<ushort>();
    buffer = buffer.Slice(sizeof(ushort));
}
for (var i = 0; i < list.Length; i++)
{
    for (var x = 0; x < numberOfCiphers; x++)
    {
        if (peerCipherList[x] == list[i].CipherCode)
        {
            return list[i];
        }
    }
}
Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.insufficient_security, "Failed to get a bulk cipher from the cipher extensions");
```

We again make as many checks on the data to ensure that we are not getting bad data at any point, again security library == reject bad data as soon as possible.

Next the loop is designed to be quick and low allocation. We are designing a component for a low down part of a framework. We don't want your TLs library triggering
garbage collections, it's just bad etiquette. 

We need a list of the ciphers that the client supports, but these are short lists so we figure out the number of items in the list
(a ushort is 2 bytes so we divide the length by 2). Then we stackalloc an array, this might be unfamiliar to many C# coders but this will make an array on the stack. This
will be unwound when the method exits and requires no allocations on the heap. It returns a pointer to the array so we need to be in an unsafe context to have this available.

## [Big O](#big-o)

Lastly we loop through our priority list looking for a valid match. This could have been implemented as a dictionary. This is a good example
on when big O notation doesn't stand up to implementation. 

Iterating a list is slower than a dictionary right? Well for a small amount of items this
is not true. The next question, how many is "small"? I found it hard to find a good breakdown so I used benchmarkdotnet[^2] to make a quick program to
test the fact. Here is a table of the results and I have checked in the code as part of the project under tests/benchmarks.

![Benchmark results showing arrays are faster until about 20 items](https://cetus.io/images/sayhello/benchmark.png)

What we can see is for our case of less than 5 items the array is far quicker (setup time for the array and the dictionary is not included in the test).

So there we have it a zero allocation, quick lookup of our cipher list.

![Bunch of keys](https://cetus.io/images/sayhello/keys.jpg)

## Key Exchange

We now have all of the information we need out of the mandatory data. So we move onto the extensions, we look for the supported groups section or the key share.
In TLS 1.3 the client estimates the key exchange type that the server will support and then pre generates a public private key pair. It can do this for more than one
key share type, Chrome and Firefox (both in the dev builds) supply two key shares. The key share contains the public key part of the pair that was generated. If we have 
one of these in our supported list we have everything we need to be able to generate our bulk ciphers.

``` csharp
public static void ReadKeyshare(ReadableBuffer buffer, IConnectionStateTls13 connectionState)
{
    buffer = BufferExtensions.SliceVector<ushort>(ref buffer);
    var ks = connectionState.CryptoProvider.GetKeyshareFromKeyshare(buffer);
    connectionState.KeyShare = ks ?? connectionState.KeyShare;
}
```

You can see here we ask the crypto provider if we support the key share and if so to make an instance of it for us and load up the public key. We need the final
null check because if we don't support the keyshare but have a keyshare from the supported groups we don't want to null out the value.

The supported groups is a "fall back". If the client didn't provide us with a public key from a keyshare we support then we need to find a keyshare we do support.
Because we would prefer to use one that we have a public key for we exit if we have a keyshare already.

``` csharp
public static void ReadSupportedGroups(ReadableBuffer buffer, IConnectionStateTls13 connectionState)
{
    if (connectionState.KeyShare != null)
    {
        return;
    }
    buffer = BufferExtensions.SliceVector<ushort>(ref buffer);
    connectionState.KeyShare = connectionState.CryptoProvider.GetKeyshareFromNamedGroups(buffer);
}
```

The method in the CryptoProvider follows the same pattern as the cipher suite selection.

In summary by the end of the extensions we should have either a key share that has a public key loaded (ideal scenario) or if we couldn't agree on those, a keyshare
without a public key from the peer. Due to the checking for a non null keyshare in the supported groups and not overwriting with null during the keyshare extension this will work no matter the order of the extensions
(there is only 1 case were the extension has to be in a specific position and that is PSK which we are not supporting yet).

At this point we exit the processing of the handshake and continue in the state machine. The first thing we do is create a hash object as we now have a cipher suite so know the hash we are using.
This is a break from the libraries I have seen and as discussed instead of keeping a log of the messages in memory which is both a security risk and a waste of space we use the ability of the hash
to be updated with more and more data. 

``` csharp
this.StartHandshakeHash(buffer);
//If we can't agree on a schedule we will have to send a hello retry and try again
if (!NegotiationComplete())
{
    writer = pipe.Alloc();
    this.WriteHandshake(ref writer, HandshakeType.hello_retry_request, Hello.SendHelloRetry);
    _state = StateType.WaitHelloRetry;
    await writer.FlushAsync();
    return;
}

private bool NegotiationComplete()
{
    if (KeyShare == null || Certificate == null)
    {
        Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.illegal_parameter,$"negotiation complete but no cipher suite or certificate");
    }
    if (!KeyShare.HasPeerKey)
    {
        return false;
    }
    return true;
}
```

Then we check that the negotiation finished with an agreed key share and a certificate the client will accept. If we can't satisfy those two basic requirements we cannot
continue with this client. 

Next we check if we have the public key from the peer agreed. If not we will need to send a "Hello Retry Request". We will then tell the client
the key exchanges we support and the client will have to send a hello again. We need to keep the hash of the client, as the handshake isn't restarting and this ensures
that a man in the middle cannot force us to negotiate a cipher or certificate that they prefer.

And that is the client hello! I have glossed over the certificates but I will leave that to a whole post of it's own. Next we can discuss the server hello (a short topic)
and the TLS 1.3 key schedule.

[^1]: [TLS 1.3 Spec B.4 Cipher Suites](https://tlswg.github.io/tls13-spec/#rfc.appendix.B.4)
[^2]: [Benchmarkdotnet on Github](https://github.com/dotnet/BenchmarkDotNet)