---
title: "TLS Building Blocks II"
excerpt: "Getting on with it - Pipes Part 2&#178+1;. Bulk Ciphers"
category: "tim"
header:
  overlay_image: buildingblocksi/lego.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: buildingblocksi/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl, leto, AES, CHACHA20, AEAD]
---

### Prior Reading

If you are returning and have read the previous posts, I am amazed you have hung around for so long but thank you. If not
and you want to gain some understanding before continuing here are the previous articles in the series. 

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)
* [Not all secrets are created equal](https://cetus.io/tim/Not-all-secrets-are-created-equal/)
* [Building Blocks I](https://cetus.io/tim/TLS-Building-blocks-I/)

![Bulk Grain](https://cetus.io/images/buildingblocksii/bulk.jpg)

## Shoveling Data

The second important primitive that we need to support a full TLS stack is the bulk encryption ciphers. These are symmetrical in 
nature due to the high performance and smaller key sizes for the same perceived level of protection. They are symmetric in that
the same key can decrypt the data that it encrypts.

There are two main types of symmetrical ciphers, block and streaming. They are pretty much as they sound, a streaming cipher allows
you to continuously encrypt an arbitrary length stream of bytes. The block cipher on the other hand needs to work on a specific block
size of data. Currently there are no recommended stream ciphers left, so we are stuck with block ciphers. 

![Disposable Numbers](https://cetus.io/images/buildingblocksii/disposable.jpg)

## Nonce Calculation

We need a nonce for the ciphers we will implement Initialially. That means a new value for every frame that can't be reused for a
given set of keys. TLS 1.2 doesn't define how to calculate this value just that we will need a 12byte value for our chosen ciphers.

So there is a problem right there. The spec never defines how this should be generated. Most implementations (SChannel for example)
use a 64bit incrementing number. OpenSSL on the other hand starts at a random number and then increments. Both of these work fine,
however some libraries and devices have been found to violate the only required property of the nonce. They repeat it! The other issue
with this scheme is that because the reader of a message has no way of deriving the nonce it must be sent with every frame adding an
8 byte overhead.

Some have suggested that it could just be dictated that you use an incrementing sequence number, forgo the sending of the 8 bytes and
be done with it. It's not a bad idea, but TLS 1.3 has an even better one.

Enter TLS 1.3, and one of the many changes. TLS 1.3 defines that the nonce will be constructed using the sequence number of the message
sent using the current key (not of the conversation and not since the first encrypted message, as you will see later these are different 
concepts in TLS 1.3). However we won't use the sequence number directly, instead we will XOR the 64bit sequence number with a 12 byte random
produced at the time of key generation. Therefore we don't need to send the nonce as both sides can derive it for any specific message,
anyone listening in won't see the nonce, and if there is a bad TLS client that tries to reuse a nonce the message will fail decryption and be
disconnected.

The upside, we can actually use the scheme defined in TLS 1.3 for our 1.2 implementation.

![Wax Seal](https://cetus.io/images/buildingblocksii/seal.jpg)

## Authentication

If someone alters any bit of a message, due to the fact that we have a block chaining mode
it should alter a large chunk of the following plaintext when we decrypt it (in theory all of it). 
However if you say know that the HTTP headers in a message are likely to be the first x characters, you could just start 
flipping bits after that and see what happens on the server. Maybe nothing or maybe you can crash a server or cause something to 
happen to the user. In order to protect against this we need some type of authentication mechanism. This will allow us to validate 
that the message sent was the message received. 

Traditionally SSL/TLS used the HMAC that we discussed in previous articles to produce a digest of the message that we can authenticate. 
Once again this concept is just fine, however there are attacks on the way it is implemented. Most (in fact for TLS 1.3 all) 
ciphers that we use in modern deployments are Authenticated Encryption with 
Associated Data (AEAD)[^2] ciphers. This means that they both authenticate as well as encrypt the data. 
The Associated Data part means we can add in extra information that isn't encrypted and authenticate that it wasn't changed.

How TLS 1.2 and TLS 1.3 deal with this differs again. TLS 1.2 takes the frame header, lengths and some other information and uses this 
as the additional part of the AEAD cipher. TLS 1.3 on the other hand doesn't use any additional information (which is completely valid)
but instead every frame that is encrypted is marked as the same message type and the actual message type is then put inside the encrypted
data.

Finally this authentication does not come for free, we need to include a "tag" at the bottom of the frame that contains the information we 
need to authenticate the message. In the case of AES this is 16 bytes of data after the message.

## Show me some code already!

The provider model comes into play here again. We have something like this

``` csharp
public interface IBulkCipherProvider
{
    IBulkCipherInstance GetCipherKey(BulkCipherType cipher);
    void Dispose();
}
```

The provider is pretty simple. We ask for a key of a specific cipher type and it returns it to us. This is platform specific but similar
in nature so we will just look for now at the OpenSSL 1.1 version

``` csharp
public class BulkCipherProvider:IBulkCipherProvider
{
    private const int MaxBufferSize = 32 + 12 + 12;
    private readonly EphemeralBufferPoolWindows _bufferPool = new EphemeralBufferPoolWindows(MaxBufferSize, 10000);

    public IBulkCipherInstance GetCipherKey(BulkCipherType cipher)
    {
        int keySize, nonceSize, overhead;
        var type = GetCipherType(cipher, out keySize, out nonceSize, out overhead);
        if (type != IntPtr.Zero)
        {
            return new AeadBulkCipherInstance(type, _bufferPool, nonceSize, keySize, overhead);
        }
        return null;
    }
```
The provider has a EphemeralBufferPool that we discussed earlier[^2]. It sets the buffer sizes to the max that we could need
to store a nonce + nonce random + key size. Currently the number of these buffers is hard coded and this should be changed but 
for a test implementation 10k concurrent keys (5k connections) should be more than enough.

Next we look at the actual cipher instance

``` csharp
public interface IBulkCipherInstance:IDisposable
{
    int Overhead { get; }
    int KeyLength { get; }
    int IVLength { get; }
    void SetKey(Span<byte> key);
    void SetIV(Span<byte> iv);
    void Decrypt(ref ReadableBuffer messageBuffer);
    void Encrypt(ref WritableBuffer buffer, ReadableBuffer plainText, RecordType recordType);
    void WithPadding(int paddingSize);
    void Encrypt(ref WritableBuffer buffer, Span<byte> plainText, RecordType recordType);
}
```

The first two methods are used by the state machine to set the key and IV (Initialization Vector is somewhat used interchangeably with
nonce in the world of TLS). 
There are two methods for encrypting data, one is for data coming from a readablebuffer (and therefore a pipeline, either the handshake
or the application data pipeline) and one for encrypting data from a Span<T> for shorter messages (like Alerts). 

The decrypt method currently decrypts in place, however I am not sure this is a good idea as we end up with plaintext in a buffer that
comes from an encrypted pipeline. The issue with decrypting this directly onto one of the decrypted pipelines is that in TLS 1.3 we
don't actually know what the message type is until we unwrap it. It is something I will have to take a further look at, maybe making
a scratch pipeline that we decrypt to, and then just append the buffer to the correct pipeline at the end.

Below is the encrypt method from the OpenSSL instance, with some cruft cut out the full version is on github.

``` csharp
int outLength;
ThrowOnError(EVP_CipherInit_ex(_ctx, _cipherType, IntPtr.Zero, (void*)_keyPointer, (void*)_ivPointer, (int)KeyMode.Encryption));
foreach(var b in plainText)
{
    if(b.Length == 0)
    {
        continue;
    }
    buffer.Ensure(b.Length);
    var inPtr = b.GetPointer(out inHandle);
    var outPtr = buffer.Memory.GetPointer(out outHandle);
    try
    {
        outLength = buffer.Memory.Length;
        ThrowOnError(EVP_CipherUpdate(_ctx, outPtr, ref outLength, inPtr, b.Length));
        buffer.Advance(outLength);
    }
    finally
    {
        //Cleanup handles
    }
}
```

Foreach buffer inside the readable we encrypt the data in that buffer. This is one place we have an advantage over SChannel. Both CNG
and OpenSSL support incremental encryption of the data, encrypting the bytes we have in the current buffer and outputting that many
bytes. SChannel requires all the data in one single buffer, not supporting fragments so if we have fragments we first have to copy 
the data causing a large allocation or another set of large buffers to be pooled.

Finally for TLS 1.3 we write the record type to the bottom (and any padding which we will leave for now) and then we write the authentication
tag last

``` csharp
writePtr = buffer.Memory.GetPointer(out outHandle);
outLength = 0;
ThrowOnError(EVP_CipherFinal_ex(_ctx, null, ref outLength));
ThrowOnError(EVP_CIPHER_CTX_ctrl(_ctx, EVP_CIPHER_CTRL.EVP_CTRL_GCM_GET_TAG, _overhead, writePtr));
buffer.Advance(_overhead);
IncrementSequence();
```

The finally method of interest is the IncrementSequence() which calculates the next nonce.

``` csharp
public unsafe void IncrementSequence()
{
    var i = _iVLength - 1;
    var vPtr = (byte*)_ivPointer;
    while (i > 3)
    {
        unchecked
        {
            var val = vPtr[i] ^ _sequence[i];
            _sequence[i] = (byte)(_sequence[i] + 1);
            vPtr[i] = (byte)(_sequence[i] ^ val);
            if (_sequence[i] > 0)
            {
                return;
            }
        }
        i -= 1;
    }
    Alerts.AlertException.ThrowAlert(Alerts.AlertLevel.Fatal, Alerts.AlertDescription.decode_error, "Failed to increment sequence on Aead Cipher");
}
```

There seems to be a lot going on in here for a simple increment. However we are incrementing a bigendian uint64. So we start from the last
byte, rather than storing the original random we only store the xor value. To get the original random we xor again against the sequence
then we add 1 to the sequence and redo the XOR. If the byte wrapped we need to do the next byte. This means most of the time we will only update
a single byte rather than XORing all of the bytes. If we get to past the 8th byte we throw an exception because we are not allowed to wrap the nonce
as it would cause it to repeat.

Decrypting is pretty much the same in reverse. I haven't implemented the TLS 1.2 version yet but it is basically the same with some authentication
data and without the padding code and the message type at the end.

That's building block two out of the way, so we have 1/2 of our primitives done.

[^1]: [Cryptographic Nonce from Wikipedia](https://en.wikipedia.org/wiki/Cryptographic_nonce)
[^2]: [Not all secrets are created equal](https://cetus.io/tim/Not-all-secrets-are-created-equal/)