---
title: "TLS Time to Make Our Keys"
excerpt: "Getting on with it - Pipes Part 2&#179;+1. 1.3 The Key Schedule"
category: "tim"
header:
  overlay_image: keys/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: keys/teaser.jpg
tags: [pipelines, tls1.3, tls, ssl, leto, keyschedule]
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
* [Client Hello](https://cetus.io/tim/TLS-Say-hello-to-my-little-friend/)

![ledger](https://cetus.io/images/keys/ledger.jpg)

## Some Book Keeping

The server needs to send back a "server hello" to the client. This confirms that we are happy with the negotiation and confirms the 
parameters we have selected. We include a block of random data here just as the client did for it's hello. 

``` csharp
public static WritableBuffer SendServerHello13(WritableBuffer buffer, IConnectionStateTls13 connectionState)
{
    buffer.Ensure(RandomLength + sizeof(ushort));
    buffer.WriteBigEndian(connectionState.Version);
    var memoryToFill = buffer.Memory.Slice(0, RandomLength);
    connectionState.CryptoProvider.FillWithRandom(memoryToFill);
    buffer.Advance(RandomLength);
    buffer.WriteBigEndian(connectionState.CipherSuite.CipherCode);
    BufferExtensions.WriteVector<ushort>(ref buffer, ExtensionsWrite.WriteExtensionList, connectionState);
    return buffer;
}
```

You can see that we tell the client the selected cipher suite, and the version. This time it is the actual selected version not TLS 1.2.
As we are still ignoring pre shared keys the only valid extension here is the public key part of the keyshare generated on the server.
This is also the last unencrypted message in the conversation. 

``` csharp
_state = StateType.SendServerHello;
writer = pipe.Alloc();
this.WriteHandshake(ref writer, HandshakeType.server_hello, Hello.SendServerHello13);
//block our next actions because we need to have sent the message before changing keys
DataForCurrentScheduleSent.Reset();
await writer.FlushAsync();
await DataForCurrentScheduleSent;
_state = StateType.ServerAuthentication;
```

This is the flow in the state machine. We set the current state to indicate we are sending the server hello. Then we allocate a Writable buffer
on the pipeline to send data to the client. The pipelines are async in nature. The problem here is that the next phase will generate encryption keys
if this takes place before the outgoing pipe actually writes our frames they will get encrypted and the handshake will fail. 

To stop this race condition we have a Signal type structure which Marc Gravell came up with. It allows us to wait until the next pipeline has completed
sending all of the data in the pipe at this time. Only once this has been done do we switch into the server authentication state and begin generating 
the keys. 

The last quick important point

``` csharp
public static void WriteHandshake(this IConnectionStateTls13 state, ref WritableBuffer writer, HandshakeType handshakeType, Func<WritableBuffer, IConnectionStateTls13, WritableBuffer> contentWriter)
{
    var dataWritten = writer.BytesWritten;
    writer.WriteBigEndian(handshakeType);
    BufferExtensions.WriteVector24Bit(ref writer, contentWriter, state);
    if (state.HandshakeHash != null)
    {
        var hashBuffer = writer.AsReadableBuffer().Slice(dataWritten);
        state.HandshakeHash.HashData(hashBuffer);
    }
}
```

This method attaches the handshake message type to the top of the message, wraps the message in a 24bit length prefixed vector (the inverse of the read vector we discussed earlier) and finally adds
the contents of the message to long running hash.

## Key Schedules

With the book keeping out of the way it is time to talk about the key schedules. This is a radical departure from TLS 1.2, there you have a single set of keys that are used for all encrypted traffic.
In TLS 1.3 you have a schedule of secrets and keys that they generate.

![TLS 1.3 Key Schedule](https://cetus.io/images/keys/schedule.png)

Here the coloured boxes represent secrets that are generated using information from the handshake but don't contain any "context". It is important for our secrets we finally use to contain context
data as well as random and secret data. The reason is that the secrets are then "bound" to the messages that have gone in the past which stops a man (or woman, or just a bot) in the middle changing
the stream of messages and being able to manipulate the results. So our secrets we actually use are a mix of this context data (hash of the current messages sent and received) and the secret for the
stage in the coloured box.

We store all of the secrets and the methods for generating the keys in a KeySchedule class. It uses the EphemeralBufferPool[^1] to store these secrets.

``` csharp
public unsafe KeySchedule(IConnectionStateTls13 state, EphemeralBufferPoolWindows pool, ReadableBuffer resumptionSecret)
{
    /* ...... Code removed for brevity */
    //Pointers to each of the secrets because we allocated a single buffer for all the secrets
    _serverHandshakeTrafficSecret = _clientHandshakeTrafficSecret + _hashSize;
    _masterSecret = _serverHandshakeTrafficSecret + _hashSize;
    _clientApplicationTrafficSecret = _masterSecret + _hashSize;
    /* .......Code removed for brevity */
    HkdfFunctions.HkdfExtract(CryptoProvider.HashProvider, CipherSuite.HashType, null, 0, resumptionPointer, secretLength, _secret, _hashSize);
}
```

This is a shortened version of the constructor. We allocate the buffer for the secrets and layout a number of pointers into the buffer for each
stages secret. Because all of the secrets end up being the hash size this is easy to layout. Finally we create out first secret (the Early secret)
based on either the PSK (which is the resumption secret). For now we assume that the buffer is empty so we will pass a null.

Because this is the first secret in the chain we have no previous secret to feed into the extract method so that is null also. Now we can take a look
at the HKDF extract function.

``` csharp
public static unsafe void HkdfExtract(IHashProvider provider, HashType hashType
    , void* salt, int saltLength, void* ikm, int ikmLength
    , void* output, int outputLength)
{
    if (saltLength == 0)
    {
        salt = (byte*)s_zeroArray;
        saltLength = provider.HashSize(hashType);
    }
    if (ikmLength == 0)
    {
        ikm = (byte*)s_zeroArray;
        ikmLength = provider.HashSize(hashType);
    }
    provider.HmacData(hashType, salt, saltLength, ikm, ikmLength, output, outputLength);
}
```

This is another place that could do with Span<T> that we can get the pointer back out of. Then the method signature could almost halve in size. You can see here we check
to see if the salt or message data is zero in length, if so we use a pointer to a buffer of bytes with the value of zero. The length we need to use is the length of the
hash algorithm defined in the cipher suite we selected during the hello.

Other than that it is a straight forward HMAC. So now we can look at how we get to our handshake secret.

``` csharp
keyShare.DeriveSecret(CryptoProvider.HashProvider, CipherSuite.HashType, _secret, _hashSize, _secret, _hashSize);
```

Here we are passing in the early secret as the key for the HMAC function that occurs inside the (EC)DHE function
described earlier[^2]. We then overwrite the early secret as we don't need it anymore with the handshake secret
this both saves memory and erases the early secret at the point we no longer require it.

When we come to the master secret we simply HMAC the early secret again, but with zeros for the key as we have no
more secret data to add.

That's it for the secrets in the coloured boxes...

## Traffic secrets

As mentioned earlier the secrets we have generated don't have any context. This is where the traffic secrets come
into play. 

``` csharp
public unsafe void GenerateHandshakeTrafficSecrets(Span<byte> hash)
{
    HkdfFunctions.ServerHandshakeTrafficSecret(CryptoProvider.HashProvider, CipherSuite.HashType, _secret, hash, new Span<byte>(_serverHandshakeTrafficSecret, _hashSize));
    HkdfFunctions.ClientHandshakeTrafficSecret(CryptoProvider.HashProvider, CipherSuite.HashType, _secret, hash, new Span<byte>(_clientHandshakeTrafficSecret, _hashSize));
} 
```

We can see that the function takes in the secret we generated earlier and a hash. The hash is the messages up until from the client hello, a hello retry
if needed (and the hello response to that) and the server hello. This is then passed into a function that takes in the handshake secret, the hash and a 
label. These labels are to differentiate the purpose of the generated secret. They all start with "TLS 1.3, " and then a label based on the purpose and the
entire string as the ASCII representation of the bytes. We have a class containing all of the constants we required

``` csharp
public static readonly byte[] ClientHandshakeTrafficSecret = Encoding.ASCII.GetBytes(Prefix + "client handshake traffic secret");
public static readonly byte[] ServerHandshakeTrafficSecret = Encoding.ASCII.GetBytes(Prefix + "server handshake traffic secret");
public static readonly byte[] ClientApplicationTrafficSecret = Encoding.ASCII.GetBytes(Prefix + "client application traffic secret");
public static readonly byte[] ServerApplicationTrafficSecret = Encoding.ASCII.GetBytes(Prefix + "server application traffic secret");
```

Now we can look at how these three parts we have are combined. The spec defines something called the "Derive-Secret Function" this simply
calls the HKDF Expand Label function with the output size set to the hash size. The expand function creates a buffer out of the label and the hash.
The layout of the buffer looks like

- 2 Bytes - Length of output
- 1 Byte - Length of Label
- n Bytes - The label as ASCII
- 1 Byte - Length of Context Hash
- h Bytes - Context Hash

To assemble this buffer we use the same stackalloc as before because once again the size is limited and we want to avoid heap allocations where
possible and practical

``` csharp
var hkdfSize = HkdfLabelHeaderSize + label.Length + hash.Length;
var hkdfLabel = stackalloc byte[hkdfSize];
var hkdfSpan = new Span<byte>(hkdfLabel, hkdfSize);
hkdfSpan.Write16BitNumber((ushort)output.Length);
hkdfSpan = hkdfSpan.Slice(sizeof(ushort));
hkdfSpan.Write((byte)label.Length);
hkdfSpan = hkdfSpan.Slice(sizeof(byte));
label.CopyTo(hkdfSpan);
hkdfSpan = hkdfSpan.Slice(label.Length);
hkdfSpan.Write((byte)hash.Length);
hkdfSpan = hkdfSpan.Slice(sizeof(byte));
hash.CopyTo(hkdfSpan);
```

You can see here we are using the new span class heavily. We create the stackalloc space and now have a byte pointer. We then wrap this in a Span<byte>
we write the 16bit number to this (bigendian as all numbers in TLS) and slice the Span to be the pointing after the data we just wrote. We continue
with each of the parts of the required buffer.

Next we can feed this into the Expand function with secret being the pseudo random key, and the buffer from above being the extra context info. The main
difference between the extract and expand functions is that the expand function can produce a stream of any size output.

![HKDF Expand](https://cetus.io/images/keys/Expand.gif)

This shows that we HMAC the inputs. We copy the result to our output and feed the HMAC and the info as well as incrementing the final byte by 1 back
into the HMAC function. We keep doing this until we have enough bytes for our requirements. The traffic secrets are the same length as the hash so they are
a single loop. 

When we generate the keys and IV's on the other hand we may require more data than the size of the hash length.

``` csharp
private unsafe IBulkCipherInstance GetKey(byte* secret, int secretLength)
{
    var newKey = CryptoProvider.CipherProvider.GetCipherKey(CipherSuite.BulkCipherType);
    var key = stackalloc byte[newKey.KeyLength];
    var keySpan = new Span<byte>(key, newKey.KeyLength);
    var iv = stackalloc byte[newKey.IVLength];
    var ivSpan = new Span<byte>(iv, newKey.IVLength);
    HkdfFunctions.HkdfExpandLabel(CryptoProvider.HashProvider, CipherSuite.HashType
        , secret, secretLength, Tls1_3Consts.TrafficKey, new Span<byte>(), keySpan);
    newKey.SetKey(keySpan);
    HkdfFunctions.HkdfExpandLabel(CryptoProvider.HashProvider, CipherSuite.HashType
        , secret, secretLength, Tls1_3Consts.TrafficIv, new Span<byte>(), ivSpan);
    newKey.SetIV(ivSpan);
    return newKey;
}
```

Here we can see that we pass in the secret that applies to our key generation, we use a string for the key "TLS 1.3, key" and "TLS 1.3, iv" for the key and
IV respectively. We use stackalloc again because the keys are small (32byte for CHACHA20 is the biggest) and it stops the secret key going into the heap
but instead it is in the stack which will be rapidly written over.

Using the handshake key we encrypt the rest of the messages in the hello from the server. Once we have finished sending those messages we use the same
pattern as before to switch to the "traffic" keys that will be used to actually send the application data.

Next up we will discuss sending the certificates and signing the messages to prove we own the certificate.


*[HKDF]: HMAC Key Derivation Function
*[HMAC]: Hash Message Authentication Code
*[PSK]: Pre-shared key
*[IV]: Initialization Vector

[^1]: [Previous post on the EphemeralBufferPool](https://cetus.io/tim/Not-all-secrets-are-created-equal/)
[^2]: [Previous post on key exchange](https://cetus.io/tim/TLS-Building-blocks-III/)
