---
title: "The journey continues to Secure Pipelines, via OpenSsl"
excerpt: "Cross platform is hard - Pipes Part 3"
category: "tim"
header:
  overlay_image: pipelines/header.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: pipelines/teaser.jpg
tags: [asp.net, .net, channels, pipelines, networking, tls, openssl, ssl, sslstream]
---

## Before we get started

In case you missed it there are two parts before this if you need to catch up

* [Part 1 - Not your Grandad's Dotnet](https://cetus.io/tim/Part-1-Not-your-grandads-dotnet/)
* [Part 2 - Pipelines formerly known as channels](https://cetus.io/tim/Part-2-pipelines/)

# What does done actually look like?

At this point I was pretty sure I was "done". Other than adding some certificate authentication I could encrypt and decrypt
data from over the wire. I had unit/integration tests showing connecting to an Pipeline-SslStream, Pipeline-Pipeline. I had got rid of 
most of the buffer copies, so I was feeling pretty smug. So I posted this to twitter

![First Benchmark](https://cetus.io/images/pipelinesopenssl/firstbenchmark.jpg)

In reality it was only just beginning. The first comment from the PR (after all of the formatting and spaces/naming issues) was 

> It works on windows, but where is the cross plat?

So that is where the story starts, looking down the barrel of the wonder that is the OpenSsl "Man" pages. I was happily surprised to find
at the core OpenSsl wasn't all that different to SSPI. So the place to start was getting our data into OpenSsl.

![Bios](https://cetus.io/images/pipelinesopenssl/bio.jpg)

## The Bio

The first thing to learn about OpenSsl is that it has the concept of a Bio (Basic In/Out) which is similar to a file handle or a stream.
The functions that we care about are the Read, Write, Control, Create and Free. There are Bios' for all sorts of inputs files, sockets,
a buffering bio the list goes on. 

Most of the examples use a Connection directly but in Pipelines this wouldn't work because the point of the 
library was to be a layer in an established pipeline. So a memory bio was what we needed. This would allow us to read and write bytes directly
into OpenSsl from our own connection. The basics we needed to get this done were

``` csharp
[DllImport(InteropCrypto.CryptoDll, CallingConvention = CallingConvention.Cdecl)]
public extern static int BIO_write(BioHandle b, void* buf, int len);
[DllImport(InteropCrypto.CryptoDll, CallingConvention = CallingConvention.Cdecl)]
public extern static int BIO_read(BioHandle b, void* buf, int len);
[DllImport(InteropCrypto.CryptoDll, CallingConvention = CallingConvention.Cdecl)]
public extern static BioHandle BIO_new(IntPtr type);
[DllImport(InteropCrypto.CryptoDll, CallingConvention = CallingConvention.Cdecl)]
public extern static void BIO_free(IntPtr bio);
[DllImport(InteropCrypto.CryptoDll, CallingConvention = CallingConvention.Cdecl)]
public extern static IntPtr BIO_s_mem();
```

The way of creating a Bio is you call a method in this case "BIO_s_mem" which returns a pointer that you then use to call "BIO_new" and this then returns
a pointer to the bio that you can then use to read and write bytes to.

``` csharp
var bioHandle = BIO_new(BIO_s_mem());
var bytesWritten = BIO_write(bioHandle, myBuffer, myBuffer.Length);
```

So we have our input and output sorted out now. Similar to SSPI we needed an overall "SslContext", but before we can even think about that there is a bunch of initialization

``` csharp
public static void Init()
{
    ERR_load_crypto_strings();
    SSL_load_error_strings();
    OPENSSL_add_all_algorithms_noconf();
    CheckForErrorOrThrow(SSL_library_init());
}
```

Some of these methods have no return value, so I can only guess they never fail? The first two loads the various error strings which really only helps with debugging and could
probably be removed in a full production build. Next we tell the crypto library to load all of the cipher algos and finally we can call Init on the Ssl library.

![Two halves](https://cetus.io/images/pipelinesopenssl/two.jpg)

One interesting thing to note is that OpenSsl is actually two libraries, one contains code for the Bio, crypto algos, 
Certificates and so on. The other contains the code for
Ssl/Tls, the protocol management and various options. Which is also not very different from how SSPI and works. The actual Cryptography part is done by another library completely (Crypto Next Generation or CNG).

Now the library has been initialized we can create our context, there is a different methods for client and server

``` csharp
private static readonly IntPtr ServerMethod = SSLv23_server_method();
private static readonly IntPtr ClientMethod = SSLv23_client_method();
public static IntPtr NewServerContext(VerifyMode mode)
{
    var returnValue = SSL_CTX_new(ServerMethod);
    SSL_CTX_set_options(returnValue, Default_Context_Options);
    SSL_CTX_set_verify(returnValue, mode, IntPtr.Zero);
    return returnValue;
}
public static IntPtr NewClientContext(VerifyMode mode)
{
    var returnValue = SSL_CTX_new(ClientMethod);
    SSL_CTX_set_options(returnValue, Default_Context_Options);
    SSL_CTX_set_verify(returnValue, mode, IntPtr.Zero);
    return returnValue;
}
```

Then it's a simple case of making an Ssl connection using Interop.SSL_new(_sslContext) and finally
create an in and an out Bio and register them with that connection, and set accept or connect depending on
if we are the client or server like so

``` csharp
Interop.SSL_set_bio(_ssl, _readBio, _writeBio);
if (IsServer)
{
    Interop.SSL_set_accept_state(_ssl);
}
else
{
    Interop.SSL_set_connect_state(_ssl);
}
```

Now we get to the handshake loop. The Bios are set relative to the OpenSsl library, so the "WriteBio" is the bio
that OpenSsl will write out to and the "ReadBio" is the bio that OpenSsl will read the bytes off the socket. 
So then we write our bytes (if we are the server and have received a client hello or do nothing if we are starting
a client connection). Next we call 

``` csharp
var result = Interop.SSL_do_handshake(_ssl);
```

A result of 1 means we have successfully completed the handshake and are ready to send/receive encrypted data. Anything else
is an "error" however we need another call to 

``` csharp
//We didn't get an "okay" message so lets check to see what the actual error was
var errorCode = Interop.SSL_get_error(_ssl, result);
if (errorCode == Interop.SslErrorCodes.SSL_NOTHING || errorCode == Interop.SslErrorCodes.SSL_WRITING ||
    errorCode == Interop.SslErrorCodes.SSL_READING)
```

If we get a "nothing/reading/writing" error code it simply means that we have data to write out, or read before we can finish the
handshake, not actually an error. Anything else denotes an actual error and we throw an exception. We check for any data written by
the library to the Bio and push it out to the underlying pipeline. After a successful connection we drop back into the encrypt/decrypt
cycle which basically works the same as SSPI. 

So there we have it, OpenSsl working for handshakes, encrypt/decrypt talking to SSPI
over Pipelines, over streams to SslStream, on Ubuntu, Osx, and Windows (obviously not the SSPI on the first two platforms). We
are done now... right? Wrong, at this point I had a PR but it was far from done. 

In the next posts I wil discuss performance and 
how I started to think about security and threats once real security people started looking at my code they should be far more interesting!
