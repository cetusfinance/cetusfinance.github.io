---
title: "TLS, is it Turtles all the way down?"
excerpt: "Rewind and Restart - Pipes Part Duex"
category: "tim"
header:
  overlay_image: turtlesallthewaydown/turtles.jpg
  overlay_filter: rgba(50, 50, 50, 0.5)
  teaser: turtlesallthewaydown/teaser.jpg
tags: [pipelines, networking, tls, openssl, ssl,nss, sslstream, leto]
---

## If you want to know why?

The previous ramblings I wrote were the "why" I am doing this. If you are interested take a quick read of

* [Maybe it's time to roll my own crypto, someone has to do it?](https://cetus.io/tim/Maybe-its-time-to-roll-my-own-crypto/)

## Pipelines, Frames, Protocols

I am using the new .Net Pipelines that are part of corefxLab (which means the cheese moves a lot). Generally the structure for
a pipelines library would look like this

![Pipelines Basic Diagram](https://cetus.io/images/turtlesallthewaydown/pipes.png)

A pipeline is basically a reader and a writer with a nice buffer pool in the middle. It's more complex and 
I am minimizing the effort put in here around buffer ownership and the push pull relationship but for our purposes that
is not important. So this would work really well if we were doing a simple compression pipeline. But TLS is more complex
than that. We have to think of security and information leak, and TLS has more than one framing.

![Pipelines Basic Diagram](https://cetus.io/images/turtlesallthewaydown/turtles.jpg)

## Turtles all the way down  

First lets talk about the record layer in TLS. There is a header with a length, a message type and a protocol version.
The protocol version has over time become mostly redundant. Old and broken proxies can't cope with an increased version
number. If you want to negotiate with an older implementation it might reject you if the version is also too high.
So we have a frame from the record layer, we unwrap (and decrypt if needed) and now we have our message, right?

Wrong! A message inside the record layer can span multiple frames, a handshake message for instance has a 24bit unsigned
int as it's length, but the record layer uses a 16bit unsigned int. That means that something like a certificate chain
which can be a large handshake message can be across multiple frames. Not only that but we are more than likely to put
multiple handshake messages in a single frame to reduce the number of actual packets and overheads we need to use.

For application frames we will always pass it back up to the consumer frame by frame so that will work with the current model.
Ideally I would like to be able to "store" unwrapped messages (if they are handshakes) until we have a complete message.
Currently I could hold a list, but that would require allocating a list and change all of the code to use a "list" of readable
buffers. I am hoping that the corefxlab team can give me a readable buffer that I can attach another to the end of (linked list?).

[Issue link for adding readable buffers to other buffers without allocation](https://github.com/dotnet/corefxlab/issues/1060)

The way of solving this issue for now, a second internal channel for handshake messages. So we end up with something like

![Pipelines Attempt](https://cetus.io/images/turtlesallthewaydown/almost.png)

Explaining the diagram in more details. We have the socket on the left which is where our encrypted (or at the start of the 
connection unencrypted) data comes in off the wire. We then decrypt and unwrap the record frame. At this point a simple decision
is made, is this an application layer frame or in internal (Alert or Handshake). If so it is redirected over another pipe. 
Here we can have another async loop that is making sure we can unwrap an entire message. That is where (the black hex) the
connection state machine lives. 

![Stop](https://cetus.io/images/turtlesallthewaydown/stop.jpg)

## STOP!

We still have one major problem (that I can see, can you?) with this design. We have a single pool of
buffers that the application code is writing to, and the sockets are consuming from. That means in the same buffers at any
time their could be cipher text or plain text. This is a common design, that I have seen over and over with say
SslStream it decrypts/encrypts the data in place. 

The problem with this is, what happens if encryption fails but the buffer
for whatever reason gets sent? So the final headline architecture, a separate buffer pool for the plain text side and the cipher text.
This should provide some protection from accidentally pushing plain text out to the socket, or the socket overrunning into some plain text
buffer. When we encrypt or decrypt data we will write the result directly onto a buffer on the clear text side.

## Show me the code

As discussed in previous articles I am going ot have a "listener" over the top, that you configure and setup with cipher suites, certificates etc.

``` csharp
public class SecurePipelineListener : IDisposable
    {
        private CryptoProvider _cryptoProvider;
        private PipelineFactory _factory;
        private CertificateList _certificateList;
        private KeyScheduleProvider _keyscheduleProvider;
        private ResumptionProvider _resumptionProvider;
        private bool _allowTicketResumption;
        private ServerNameProvider _serverNameProvider;

        public SecurePipelineListener(PipelineFactory factory, CertificateList certificateList)
        {
            _factory = factory;
            _serverNameProvider = new ServerNameProvider();
            _keyscheduleProvider = new KeyScheduleProvider();
            _cryptoProvider = new CryptoProvider();
            _resumptionProvider = new ResumptionProvider(4, _cryptoProvider);
            _certificateList = certificateList;
        }

        public SecurePipelineConnection CreateSecurePipeline(IPipelineConnection pipeline)
        {
            return new SecurePipelineConnection(pipeline, _factory, this);
        }
```

That's it a container for settings. At the moment they are all hard coded in there but at a later date
we will replace it with the ability to actually configure it at runtime.

So now to get a connection we can use a simple code block like this

``` csharp
using (var factory = new PipelineFactory())
using (var list = new CertificateList())
{
    var thumb = "MYTHUMPRINT";
    list.LoadCertificateFromStore(thumb,true);
    using (var serverContext = new SecurePipelineListener(factory, list))
    using (var socketClient = new System.IO.Pipelines.Networking.Sockets.SocketListener(factory))
    {
          var ip = IPAddress.Any;
          var port = 443;
          var ipEndPoint = new IPEndPoint(ip, port);
          socketClient.OnConnection(async s =>
          {
              var sp = serverContext.CreateSecurePipeline(s);
              await ServerLoop.HandleConnection(sp);
          });
          socketClient.Start(ipEndPoint);

```

And that's it, we need at least one certificate (we will get to multiple certificates later), setup a port and away we go.

Inside the SecurePipeline code the constructor looks like this

``` csharp
public SecurePipelineConnection(IPipelineConnection pipeline, PipelineFactory factory, SecurePipelineListener listener)
{
    _listener = listener;
    _lowerConnection = pipeline;
    _outputPipe = factory.Create();
    _inputPipe = factory.Create();
    _handshakePipe = factory.Create();
    _handshakeOutpipe = factory.Create();
    StartReading();
}

public IPipelineReader Input => _outputPipe;
public IPipelineWriter Output => _inputPipe;
```

This shows the explosion of pipes that we need to be able to handle the internal unwinding of the frames. We map the input to
the output for the application, well because it makes the internal code clearer (reading from an output seems odd and pipelines
is structured in reverse). So we have a "connection" and the overall settings. We can start to take a look at the higher level
state machine that we use for reading data from the socket (or other pipeline).

``` csharp
while (true)
{
    var result = await _lowerConnection.Input.ReadAsync();
    var buffer = result.Buffer;
    try
    {
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

That's a good place to stop and discuss some of the design decisions that are showing up already. We have no internal state machine when a connection is
made. Instead we call out to a factory that will look at the initial handshake and decide the highest version that the client supports. If the client
doesn't support any version we support the factory will take care of throwing the appropriate exception. Through out the library I have attempted to 
throw exceptions early and mostly fail on anything going wrong. This is to avoid trying to "recover" as we are in a security context and recovery leads
to unexpected hacks. 

The next part is why the factory anyway? A number of the TLS/SSL libraries suffer from large amounts of "if version x do y". This makes it very hard to
understand the exact flow of the code for a specific version. By separating the code at this early stage we can then go and look at the exact flow
of a TLS 1.2 connection for instance and compare the behaviour to the specs and to known issues. 

Finally if you watched the S2N video from the first article you can see the logic I am trying to use (when possible) is to test something and exit,
this reduces the depth of the branches and makes it easier to understand for example we could instead of above do this

``` csharp
while (true)
{
    var result = await _lowerConnection.Input.ReadAsync();
    var buffer = result.Buffer;
    try
    {
        ReadableBuffer messageBuffer;
        while (RecordProcessor.TryGetFrame(ref buffer, out messageBuffer))
        {
            var recordType = RecordProcessor.ReadRecord(ref messageBuffer, _state);
            if (_state == null)
            {
                if (recordType == RecordType.Handshake)
                {
                        //DO NESTED STUFF

                }
                //throw exception because it wasn't a Handshake
```

Already the code is nested enough with the while true and the try catches we don't need to add to the problem.

From here we either push the unencrypted and unwrapped data onto our output pipe (the applications input pipe)
or if it is a handshake, alert or change cipher spec onto the state machines input pipe and tell the state machine
to process it.

