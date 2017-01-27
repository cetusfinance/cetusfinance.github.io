---
title: "TLS Building Blocks III"
excerpt: "Getting on with it - Pipes Part 2x3. Key Exchange"
category: "tim"
header:
  overlay_image: buildingblocksi/lego.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: buildingblocksi/teaser.jpg
tags: [pipelines, ecdhe, dhe, x5519, tls, openssl, ssl, leto]
---

### Prior Reading

If you are returning and have read the previous posts, I am amazed you have hung around for so long but thank you. If not
and you want to gain some understanding before continuing here are the previous articles in the series. The series is going
to assume you understand the terms introduced previously. 

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)
* [TLS, is it Turtles all the way down?](https://cetus.io/tim/TLS-Turtles-all-the-way-down/)
* [Not all secrets are created equal](https://cetus.io/tim/Not-all-secrets-are-created-equal/)
* [Building Blocks I - Hash/HMAC](https://cetus.io/tim/TLS-Building-blocks-I/)
* [Building Blocks II - Bulk Ciphers](https://cetus.io/tim/TLS-Building-blocks-II/)

## Skipping Ahead

We could switch back and forth between TLS 1.2 and 1.3 however the more complex case is TLS 1.3 so we are going to focus on that.
If there is interest I will write up at the end how to write the TLS 1.2 implementation which should be simple if we can make a 
1.3 version.

![Rings](https://cetus.io/images/buildingblocksiii/rings.jpg)

## Elliptical Curves

In the beginning there was RSA and large primes. We are used to seeing key sizes of 2048bit and larger. However to enable perfect forward
secrecy we need to have ephemeral key exchange. So we have Diffie Hellman, again it requires large key sizes to calculate. So enter
Elliptical curves, in this we have a formula for a curve and we simply pick a random point along that curve to exchange as our secrets.
I won't go in depth about the curve calculation as others will do it more justice [^1]. However elliptical curves provide us with
a much smaller key size and reduced cpu load. While originally this was thought to benefit mobile devices it is really a server with
1000's of connections that benefits the most.

Needless to say we need both a curve definition
as well as the random point on the curve. Originally a server could pick an actual curve definition itself. However it is both time consuming
to find a curve formula (it can take days of CPU crunching) and fraught with issues. A badly selected set of curve parameters can cause
a strong algorithm to become basically useless. 

Enter pre-calculated curves with strong parameters. NIST [^2] has provided the world with a set of standard curves for us to select from
there have been curves provided by other people and teams but they have been less widespread (the brainpool series comes to mind).
Some people take issue with the fact that the NIST curves are provided by the American government and all of the implications that,
that has. However the main issue with using these curves is that it is still possible to select bad points on the curve and to have 
timing issues etc.

So the final step is the curve types called X25519 and X448. These are a curve type and associated function. They were independently
designed and are currently considered to be the best option for both security and ease of implementation. They have some interesting
users such as 

1. iOS 
2. Signal Protocol (Facebook Messanger, Whatsapp, Signal)
3. Tor
4. OpenSSH

There are many more. So that will be our favored key exchange. TLS 1.3 only allows predefined curves, but also parameters for
non elliptical curve definitions as well. The complete set of TLS 1.3 Key Exchange types are as follows

1. X25519
2. X448
3. secp256r1
4. secp521r1
5. secp384r1
6. ffdhe8192
7. ffdhe6144
8. ffdhe4096
9. ffdhe3072
10. ffdhe2048

They are ordered in the order of priority for my current library. This should be able to be configured in the future so that
an admin can choose how they would like to prioritize the various methods.

## Code time!

We follow the same provider model, with OpenSsl and CNG implementations. I have yet to implement the X25519 curves in CNG because
the documentation is less than fantastic, so I have a proof of concept for the NIST curves only. Therefore I will show the OpenSsl 
implementations.

We have the following classes

![Class Layout](https://cetus.io/images/buildingblocksiii/classes.jpg)

You can see that we have the provider but also three types of instances. The finite field instance does what it says on the tin.
The reason for have both a ECCurve and ECFunction instance is because OpenSsl doesn't treat the X25519 and X448 curves as a simple
curve type but instead as it's own high level algorithm.

``` csharp
private unsafe EVP_PKEY CreateParams()
{
    const EVP_PKEY_Ctrl_OP op = EVP_PKEY_Ctrl_OP.EVP_PKEY_OP_PARAMGEN | EVP_PKEY_Ctrl_OP.EVP_PKEY_OP_KEYGEN;

    var ctx = EVP_PKEY_CTX_new_id(EVP_PKEY_type.EVP_PKEY_EC, IntPtr.Zero);
    try
    {
        ThrowOnError(EVP_PKEY_paramgen_init(ctx));
        ThrowOnError(EVP_PKEY_CTX_ctrl(ctx, EVP_PKEY_type.EVP_PKEY_EC, op, EVP_PKEY_Ctrl_Command.EVP_PKEY_CTRL_EC_PARAMGEN_CURVE_NID, _curveNid, null));
        EVP_PKEY key;
        ThrowOnError(EVP_PKEY_paramgen(ctx, out key));
        return key;
    }
    finally
    {
        ctx.Free();
    }
}
```

This function sets up the parameters (predefined curve) on our context and then generates a "key" that has the parameters set. Next we actually
generate a public/private key set.

``` csharp
private void GenerateECKeySet()
{
    if(_eKey.IsValid())
    {
        return;
    }
    var param = CreateParams();
    var keyGenCtx = default(EVP_PKEY_CTX);
    try
    {
        keyGenCtx = EVP_PKEY_CTX_new(param, IntPtr.Zero);
        ThrowOnError(EVP_PKEY_keygen_init(keyGenCtx));
        EVP_PKEY keyPair;
        ThrowOnError(EVP_PKEY_keygen(keyGenCtx, out keyPair));
        _eKey = keyPair;
    }
    finally
    {
        keyGenCtx.Free();
        param.Free();
    }
}
```

I check that I don't already have a private key set. This is because we may need to generate the public/private key from the call
to export our public key, or when we receive the peer key depending on if we are the client or the server side of the communications.

We then generate the key pair and make sure that we clean up both the parameters we setup and the key generation context. Importing
the peers public key is uninteresting but you can look at the code in the repo for that. The final part once we have our private
key and the peers public key is we "mix" them to get our secret value.

``` csharp
public unsafe void DeriveSecret(IHashProvider hashProvider, HashType hashType, void* salt, int saltSize, void* output, int outputSize)
{
    var ctx = EVP_PKEY_CTX_new(_eKey, IntPtr.Zero);
    try
    {
        ThrowOnError(EVP_PKEY_derive_init(ctx));
        ThrowOnError(EVP_PKEY_derive_set_peer(ctx, _clientKey));
        IntPtr len = IntPtr.Zero;
        ThrowOnError(EVP_PKEY_derive(ctx, null, ref len));

        var data = stackalloc byte[len.ToInt32()];
        ThrowOnError(EVP_PKEY_derive(ctx, data, ref len));
        Dispose();
        hashProvider.HmacData(hashType, salt, saltSize, data, len.ToInt32(), output,outputSize);
    }
    finally
    {
        ctx.Free();
    }
}
```

So we create a context for our deriving (or mixing) operation, set the peers public key and our private key. We then call the function
with no buffer, which returns the length of the required buffer. This size will be be small (~40ish bytes) so we allocate the space
on the stack. We then call the function again with the new buffer and get the secret data. 

You may wonder why we then HMAC the secret when we perform all of the rest of key extraction and expansion outside of this method.
If you were thinking this was strange you would be correct and is a change in behavior that was required due to the implementation
of the CNG secrets providers. CNG will not export to a calling library the derived secret, as we need to HMAC it as the first
operation anyway we can export the HMAC'd value. To keep the interfaces identical the OpenSsl implementation had to do the HMAC operation
internally as well.

### Moving on

The last part in the building blocks series is certificates and asymmetrical ciphers. They won't be important until later so for the 
next part I will discuss the TLS 1.3 handshake and actually get some way in establishing the connection.

[^1]: [Cloudflare Elliptic Curve Primer](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/amp/)
[^2]: [National Institute of Standards and Technology](https://www.nist.gov/)