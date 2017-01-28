---
title: "TLS Version Selection"
excerpt: "Getting on with it - Pipes Part 4th Prime. TLS Version Detection"
category: "tim"
header:
  overlay_image: versionselection/shift.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: versionselection/teaser.jpg
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

![Water flowing in a river](https://cetus.io/images/versionselection/water.jpg)

## Getting into the flow

TLS is made up with a flow of messages. So we start with the client connecting a TCP socket to the server, at this point we are in the
"handshake". The client initiates the communication by sending a "client hello" message. At this point the server has no idea what the client
supports in the way of ciphers and TLS versions. 

Often in the current libraries for TLS there is a single set of loops with a lot of
branches for the various versions. I feel this induces a high cognitive load on a developer (read I can't figure out what I am doing) 
trying to understand what is happening
so at this point in the server code we call a "VersionSelectionFactory". This reads the client hello, decides upon the version we are
supporting and returns a state machine specific to that version.

``` csharp
ReadableBuffer messageBuffer;
while (RecordProcessor.TryGetFrame(ref buffer, out messageBuffer))
{
    var recordType = RecordProcessor.ReadRecord(ref messageBuffer, _state);
    if (_state == null)
    {
        if (recordType != RecordType.Handshake)
        {
            Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.unexpected_message, "Requre a handshake for first message");
        }
        _state = VersionStateFactory.GetNewStateMachine(messageBuffer, _listener);
        HandshakeWriting();
    }
```

This shows part of the basic loop for reading data from the socket. We try to get a complete frame. If the frame is incomplete the 
"TryGetFrame" returns false and we return to waiting on the socket for more data. If we have a complete record and we don't yet have
a version specific state machine we check that the message is a handshake. 

The next important part here is we call out to a method that throws a library specific exception basic on the alert message type. We
will go into depth on that later, but for now just know that it is defined by the TLS specification.

Next we call the version state machine which is a simple case statement 

``` csharp
public static IConnectionState GetNewStateMachine(ReadableBuffer buffer, SecurePipelineListener listener)
{
    switch(GetVersion(ref buffer))
    {
        case TlsVersion.Tls12:
            return new ServerStateTls12(listener);
        case TlsVersion.Tls13Draft18:
            return new ServerStateTls13Draft18(listener);
        default:
            Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.protocol_version, "Unsupported version");
            return null;
    }
}
```

From this you can see we are supporting TLS 1.2 and 1.3 Draft 18. This is important TLS 1.3 will have the version number 0x0304
(SSL 3 is 0x0300 and the version numbers have caused much debate). However while the RFC has not yet been finalized and is in Draft
they have given special version numbers to the Draft so that clients and servers can ensure they are using the same specification.

The return null is because the compiler doesn't understand that the alert will always throw an exception and thus exit, so it will
never actually be hit. As a side note, moving your exception throwing outside your code like this allows for better in-lining as methods
with exceptions being thrown struggle to be in-lined.

## Pick a version any version

Before TLS 1.3 selecting a version was simple, the client sent the maximum version number it supported. The server then selected its maximum
it supported up to the clients maximum and sent the response. You only needed to read the very top of the client hello to find the version.
As with many things TLS 1.3 changes all of that. In order to understand the new version selection we need to understand the basic structure of 
Handshake messages a diagram might help [^1]

![Client Hello Structure](https://cetus.io/images/versionselection/clienthello.png)

The version that is in the fixed part for TLS 1.3 is the same as for TLS 1.2. This may seem strange at first however it has been shown
in the past that some server implementations have rejected clients that provide a higher number than they support or knew about when
they were written (which is not what the spec says, this happens when specs meet reality). This means the roll out of previous versions
of TLS have taken a long long time. To avoid this the client hello looks like a TLS 1.2. Instead there is a new extension called supported
versions. 

This will help with backwards compatibility however it means we have to read all the way until the end to find the supported versions
before we can make a decision. Currently I am just reading through the message throwing away all of the information I am skipping only
to parse it later. This can be improved by keeping this information in a temporary state object and passing to the state machines 
and I will take a look at that later.

``` csharp
private static TlsVersion GetVersion(ref ReadableBuffer buffer)
{
    //Jump the version header and the randoms
    buffer = buffer.Slice(HandshakeProcessor.HandshakeHeaderSize);
    TlsVersion version;
    buffer = buffer.SliceBigEndian(out version);
    if (!_supportedVersion.Contains(version))
    {
        Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.protocol_version, $"The version was not in the supported list {version}");
    }
    //Slice out the random
    buffer = buffer.Slice(Hello.RandomLength);
    //No sessions slice and dump
    BufferExtensions.SliceVector<byte>(ref buffer);
    //Skip the cipher suites if we find a version we are happy with
    //then the cipher suite is dealt with by that version
    BufferExtensions.SliceVector<ushort>(ref buffer);
    //Skip compression, we don't care about that either, we just want to get to the end
    BufferExtensions.SliceVector<byte>(ref buffer);
    //And here we are at the end, if we have no extensions then we must be the header version that
    //we accepted earlier
    if (buffer.Length == 0)
    {
        return version;
    }
    buffer = BufferExtensions.SliceVector<ushort>(ref buffer);
    while(buffer.Length >= 8)
    {
        ExtensionType type;
        buffer = buffer.SliceBigEndian(out type);
        var ext = BufferExtensions.SliceVector<ushort>(ref buffer);
        if(type == ExtensionType.supported_versions)
        {
            //Scan the version for supported ones
            return ExtensionsRead.ReadSupportedVersion(ext, _supportedVersion);
        }
    }
    return version;
}
```

I think that is pretty self explanatory however I will explain the SliceVector method, first the code

``` csharp
public static ReadableBuffer SliceVector<[Primitive]T>(ref ReadableBuffer buffer) where T : struct
{
    uint length = 0;
    length = buffer.ReadBigEndian<ushort>();
    buffer = buffer.Slice(sizeof(ushort));
    var returnBuffer = buffer.Slice(0, (int)length);
    buffer = buffer.Slice(returnBuffer.End);
    return returnBuffer;
}
```

This is not the exact method (I removed the switch for the generic to simplify the code). This provides a helper
method for the common "vector" type used in TLS. It is a length prefixed run of bytes. The length can be between 8
and 32bits long (although the most I have seen is 24bits). The [Primitive] tag on the generic type actually doesn't
do anything at this point, but could be used as a way of using a Rosalyn analyzer to make sure you aren't using a random
struct. It might also make it into the compiler so it can't hurt having it there.

We return a new readable buffer with the contents of the vector and we slice the original vector so the start is past the
section we just sliced out. We need to use by ref because the readable buffer is a struct.

I use this method a lot but have a performance concern here. The readable buffer clocks in at ~30bytes at the time of writing
this means I am making 32 byte copies over and over and this could have a performance impact. On the flip side there won't be any
allocations.

So that is that.. we have our TLS version, we have selected our state machine and now we can actually get on with the negotiation!

[^1]: [RFC5246 7.4.1.2 Client Hello](https://tools.ietf.org/html/rfc5246#section-7.4.1.2)