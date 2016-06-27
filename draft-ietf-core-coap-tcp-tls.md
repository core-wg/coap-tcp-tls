---
stand_alone: true
ipr: trust200902
docname: draft-ietf-core-coap-tcp-tls-latest
cat: std
coding: utf-8
pi:
  strict: 'yes'
  toc: 'yes'
  tocdepth: '2'
  symrefs: 'yes'
  sortrefs: 'yes'
  compact: 'no'
  subcompact: 'no'
  comments: 'yes'
  inline: 'yes'
title: CoAP (Constrained Application Protocol) over TCP, TLS, and WebSockets
abbrev: TCP/TLS/WebSockets Transports for CoAP
area: Applications Area (app)
wg: CORE
date: 2016-04-21
author:
- ins: C. Bormann
  name: Carsten Bormann
  org: Universitaet Bremen TZI
  street: Postfach 330440
  city: Bremen
  code: D-28359
  country: Germany
  phone: "+49-421-218-63921"
  email: cabo@tzi.org
- ins: S. Lemay
  name: Simon Lemay
  org: Zebra Technologies
  street: 820 W. Jackson Blvd. Suite 700
  city: Chicago
  code: '60607'
  country: United States of America
  phone: "+1-847-634-6700"
  email: slemay@zebra.com
- ins: V. Solorzano Barboza
  name: Valik Solorzano Barboza
  org: Zebra Technologies
  street: 820 W. Jackson Blvd. suite 700
  city: Chicago
  code: '60607'
  country: United States of America
  phone: "+1-847-634-6700"
  email: vsolorzanobarboza@zebra.com
- ins: H. Tschofenig
  name: Hannes Tschofenig
  org: ARM Ltd.
  street: 110 Fulbourn Rd
  city: Cambridge
  code: CB1 9NJ
  country: Great Britain
  email: Hannes.tschofenig@gmx.net
  uri: http://www.tschofenig.priv.at
- ins: K. Hartke
  name: Klaus Hartke
  org: Universitaet Bremen TZI
  street: Postfach 330440
  city: Bremen
  code: 'D-28359'
  country: Germany
  phone: "+49-421-218-63905"
  email: hartke@tzi.org
- ins: B. Silverajan
  name: Bilhanan Silverajan
  org: Tampere University of Technology
  street: Korkeakoulunkatu 10
  city: Tampere
  code: 'FI-33720'
  country: Finland
  email: bilhanan.silverajan@tut.fi
- role: editor
  ins: B. Raymor
  name: Brian Raymor
  org: Microsoft
  street: One Microsoft Way
  city: Redmond
  code: '98052'
  country: United States of America
  email: brian.raymor@microsoft.com
normative:
  RFC0793: tcp
  RFC2119: bcp14
  RFC3986: RFC3986
  RFC4395: RFC4395
  RFC5246: tls
  RFC5785: RFC5785
  RFC6455: RFC6455
  RFC7252: coap
  RFC7301: alpn
  RFC7525: RFC7525
  RFC7595: urireg
  RFC7641: RFC7641
  I-D.ietf-dice-profile:
informative:
  I-D.bormann-core-cocoa: cocoa
  I-D.ietf-core-block: block
  I-D.becker-core-coap-sms-gprs: I-D.becker-core-coap-sms-gprs
  I-D.dijk-core-sleepy-reqs: I-D.dijk-core-sleepy-reqs
  RFC0768: udp
  RFC5234: RFC5234
  RFC6454: RFC6454
  RFC6335: portreg
  RFC6347: dtls
  RFC7230: RFC7230
  HomeGateway:
    title: An experimental study of home gateway characteristics
    author:
    - ins: L. Eggert
      name: Lars Eggert
      org: ''
    date: 2010
    seriesinfo:
      Proceedings: of the 10th annual conference on Internet measurement

--- abstract

The Constrained Application Protocol (CoAP), although inspired by HTTP, was designed to use UDP
instead of TCP.  The message layer of the CoAP over UDP protocol includes services for
reliable delivery, simple congestion control, and flow control.

Some environments would benefit from the availability of CoAP over a reliable
transport such as TCP or WebSockets, which already provides such services.  This document
outlines the changes required to use CoAP over TCP, TLS, and WebSockets transports.

--- middle

# Introduction {#introduction}

The [Constrained Application Protocol (CoAP)](#RFC7252) was designed
for Internet of Things (IoT) deployments, assuming that UDP {{RFC0768}} 
or DTLS {{RFC6347}} over UDP can be used unimpeded. UDP is a good choice
for transferring small amounts of data across networks that follow the IP
architecture.

Some CoAP deployments need to integrate well with existing enterprise
infrastructures, where UDP-based protocols may not be well-received or may
even be blocked by firewalls. Middleboxes that are unaware of CoAP usage for
IoT can make the use of UDP brittle, resulting in lost or malformed packets.

To address such environments, this document defines additional bindings for CoAP,
including TCP, TLS, and WebSockets.

~~~~
+-----------------------------------------------------------+
|                                                           |
|                        Application                        |
|                                                           |
+-----------------------------------------------------------+
|                                                           |
|                           CoAP                            |
|                  Requests and Responses                   |
|                                                           |
+ - - - - - - - - - +-------------------+-------------------+
|                   |                   |                   |
|       CoAP        |     CoAP over     |     CoAP over     |
|     Messaging     |    TCP and TLS    |    WebSockets     |
|                   |                   |                   |
+---------+---------+---------+---------+-------------------+
|         |         |         |         |                   |
|   UDP   |  DTLS   |   TCP   |   TLS   |    WebSockets     |
|         |         |         |         |                   |
+---------+---------+---------+---------+-------------------+
~~~~
{: #layering title='Abstract layering of CoAP extended by TCP, TLS, and WebSockets' artwork-align="center"}

Where NATs are present, CoAP over TCP can help with their traversal.
NATs often calculate expiration timers based on the transport layer protocol
being used by application protocols. Many NATs maintain TCP-based NAT bindings
for longer periods based on the assumption that a transport layer protocol, such
as TCP, offers additional information about the session life cycle. UDP, on the other
hand, does not provide such information to a NAT and timeouts tend to be much 
shorter, as confirmed by research {{HomeGateway}}.

Some environments may also benefit from the ability of TCP to exchange
larger payloads (such as firmware images) without application layer
segmentation and to utilize the more sophisticated congestion control
capabilities provided by many TCP implementations.

(Note that there is ongoing work to add more elaborate congestion control
to CoAP as well, see {{-cocoa}}.)

CoAP may be integrated into a Web environment where the front-end
uses CoAP over UDP from IoT devices to a cloud infrastructure and then CoAP
over TCP between the back-end services. A TCP-to-UDP gateway can be used at
the cloud boundary to communicate with the UDP-based IoT device.

To make IoT devices work smoothly in these demanding environments, CoAP
needs to make use of a different transport protocol, namely TCP {{RFC0793}},
in some situations secured by TLS {{RFC5246}}.

Some corporate networks only allow Internet access via a HTTP proxy.
In this case, the best transport for CoAP would be the [WebSocket Protocol](#RFC6455).
The WebSocket protocol provides two-way communication between a client
and a server after upgrading an [HTTP/1.1](#RFC7230) connection and may
be available in an environment that blocks CoAP over UDP. Another scenario
for CoAP over WebSockets is a CoAP application running inside a web browser
without access to connectivity other than HTTP and WebSockets.

This document specifies how to access resources using CoAP requests
and responses over the WebSocket Protocol. This allows
connectivity-limited applications to obtain end-to-end CoAP
connectivity either by communicating CoAP directly with a CoAP server
accessible over a WebSocket Connection or via a CoAP intermediary
that proxies CoAP requests and responses between different transports,
such as between WebSockets and UDP.


## Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.

This document assumes that readers are familiar with the terms and
concepts that are used in {{RFC6455}} and {{RFC7252}}.

# CoAP over TCP

The interaction model of CoAP over TCP is very similar to the one for
CoAP over UDP, with the key difference that using TCP voids the need to
provide certain transport layer protocol features at the CoAP level, 
such as reliable delivery, fragmentation and reassembly, as well as 
congestion control. The protocol stack is illustrated in {{stack}} 
(derived from {{RFC7252}}, Figure 1).

~~~~
        +----------------------+
        |      Application     |
        +----------------------+
        +----------------------+
        |  Requests/Responses  |  CoAP (RFC 7252)
        |----------------------|
        |    Message Adapter   |  This Document
        +----------------------+
        +-----------+      ^
        |    TLS    |  or  |
        +-----------+      v
        +----------------------+
        |          TCP         |
        +----------------------+
~~~~
{: #stack title='The CoAP over TLS/TCP Protocol Stack'}

Since TCP offers reliable delivery, there is no need to offer a redundant
acknowledgement at the CoAP messaging layer.

Since there is no need to carry around acknowledgement semantics,
messages do not require a message type; no
message layer acknowledgement is expected or even possible.
By the nature of TCP,
messages are always transmitted
reliably over TCP. {{fig-untyped1}} (derived from {{RFC7252}}, Figure 3) shows this
message exchange graphically.  A UDP-to-TCP gateway will therefore discard
all
empty messages, such as empty ACKs (after operating on them at the
message layer), and re-pack the contents of all non-empty CON, NON, or
ACK messages (i.e., those ACK messages that have a piggy-backed
response) into untyped messages.

Similarly, there is no need to detect duplicate delivery of a message.
In UDP CoAP, the Message ID is used for relating acknowledgements to
Confirmable messages as well as for duplicate detection.
Since the Message ID thus is not meaningful over TCP, it is elided (as indicated
by the
dashes in {{fig-untyped1}}).

~~~~
        Client                Server
           |                     |
           | (no type) [------]  |
           +-------------------->|
           |                     |
~~~~
{: #fig-untyped1 title='Untyped Message Transmission over TCP.'}

A response is sent back as defined in {{RFC7252}}, as illustrated in
{{fig-untyped2}} (derived from {{RFC7252}}, Figure 6).

~~~~
        Client                Server
           |                    |
           | (no type) [------] |
           | GET /temperature   |
           |   (Token 0x74)     |
           +------------------->|
           |                    |
           | (no type) [------] |
           |   2.05 Content     |
           |   (Token 0x74)     |
           |     "22.5 C"       |
           |<-------------------+
           |                    |
~~~~
{: #fig-untyped2 title='Untyped messages carrying Request/Response.'}

EDITOR: These are "leftover" paragraphs from the original Overview
section. Keeping until I perform a complete editorial pass of the
"roughed in" material.

Conceptually, the CoAP over TCP/TLS specification replaces most of
CoAP's message layer by a new message adapter on top of TCP or TLS
that provides a framing mechanism on top of the byte stream
provided by TCP/TLS, conveying the length information about each CoAP
message that on datagram transports is provided by the datagram layer
below (UDP, DTLS).

The message adapter also adds a way to use signaling messages to
perform various housekeeping operations on the TCP, see
[I-D.bormann-core-signaling].

When CoAP is used over TLS then some of the housekeeping features are
already available with the TLS Handshake protocol; less new
functionality is then required.

Modifications to CoAP beyond the replacement of the message layer
(e.g., to introduce further optimizations) are intentionally avoided.

## Message Format {#tcp-message-format}

The CoAP message format defined in {{RFC7252}}, as shown in 
{{CoAP-Header}}, relies on the datagram transport (UDP, or DTLS over
UDP) for keeping the individual messages separate and for providing 
length information. 

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Ver| T |  TKL  |      Code     |          Message ID           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #CoAP-Header title='RFC 7252 defined CoAP Message Format.'}

In a stream oriented transport protocol such as TCP, a form of message 
delimitation is needed.  For this purpose, CoAP over TCP introduces a 
length field with variable size. {{fig-shim}} shows the adjusted CoAP 
header format with a modified structure for the fixed header (first 4
bytes of the UDP CoAP header), which includes the
length information of variable size, shown here as an 8-bit length.

~~~~

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Len=13 |  TKL  | Length (8-bit)|      Code     | TKL bytes ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #fig-shim title='CoAP Header with 8-bit Length in Header.'}

The initial byte of the frame contains two nibbles, in a similar way
to the CoAP option encoding (see Section 3.1 of {{RFC7252}}). 

Len:
: The first nibble, Len, is interpreted as a 4-bit unsigned integer.
  A value between 0 and 12 directly indicates the length of message
  in bytes starting with the first bit of the Options field. The
  other three values have a special meaning:

    13:
    : An 8-bit unsigned integer follows the initial byte and indicates
      the length of options/payload minus 13.

    14:
    : A 16-bit unsigned integer in network byte order follows the
      initial byte and indicates the length of options/payload minus
      269.

    15:
    : A 32-bit unsigned integer in network byte order follows the
      initial byte and indicates the length of options/payload minus
      65805.

TKL:
: The second nibble of the initial byte indicates the token length.

The following figures show the shim headers for the 0-bit, 16-bit, and 
the 32-bit headers. 

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Len  |  TKL  |      Code     | Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #fig-shim1 title='CoAP Header with elided Length Header.'}

For example: A CoAP message just containing a 2.03 code with the
token 7f and no options or payload would be encoded as shown in {{fig-shim2}}.

~~~~
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      0x01     |      0x43     |      0x7f     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

 Len   =    0 ------>  0x01
 TKL   =    1 ___/
 Code  =  2.03     --> 0x43
 Token =               0x7f
~~~~
{: #fig-shim2 title='CoAP Header Example.'}

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Len=14 |  TKL  | Length (16 bits)              |   Code        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #fig-shim3 title='CoAP Header with 16-bit Length in Header.'}

~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Len=15 |  TKL  | Length (32 bits)                              
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               |    Code       |  Token (if any, TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #fig-shim4 title='CoAP Header with 32-bit Length in Header.'}

The semantics of the other CoAP header fields are left unchanged.



## Message Transmission

EDITOR: This is extremely similar to the WebSocket section of the same name.

As CoAP exchanges messages asynchronously over the TCP connection, the
client can send multiple requests without waiting for responses.  For
this reason, and due to the nature of TCP, responses are returned
during the same TCP connection as the request.  In the event that the
connection gets terminated, all requests that have not yet elicited a
response are implicitly canceled; clients may transmit the request
again once a connection is reestablished.

Furthermore, since TCP is bidirectional, requests can be sent from
both the connecting host and the endpoint that accepted the connection.
In other words, the question who initiated the TCP connection has no bearing on the
meaning of the CoAP terms client and server.

# Interaction with Block-Wise Transfer

The message size limitations defined in Section 4.6 of CoAP {{RFC7252}}
are no longer strictly necessary when CoAP is used over a reliable
byte stream transport. While this appears to obviate
the need for the Block-wise transfer protocol {{-block}}, entirely
getting rid of it is not a generally applicable solution, as:

* large messages, such as firmware downloads, may cause undesired
  head-of-line blocking when a single TCP connection is used;

* a UDP-to-TCP gateway may simply not have the context to convert a
  message with a Block option into the equivalent exchange without any
  use of a Block option.

This specification extends the block protocol to be
able to make use of the larger messages over a reliable transport.
This extension is called 'Block-Wise Extension for Reliable Transport
(BERT)'.

{{-block}} uses a message size of maximum 1024 bytes and this
specification extends this maximum size to 1024 KiB (= 1 MiB) by
re-interpreting the length information.

The use of this new extension is signalled by sending Block1 or
Block2 options with SZX == 7 (a "BERT option"). (SZX == 7 is a value
that was reserved in {{-block}}.)

In control usage, a BERT option is interpreted in the same way as the
equivalent option with SZX == 6, except that it also indicates the
capability to process BERT blocks.  As with the basic Block protocol,
the recipient of a CoAP request with a BERT option in control usage is
allowed to respond with a different SZX value, e.g. to send a non-BERT
block instead.

In descriptive usage, a BERT option is interpreted in the same way as
the equivalent option with SZX == 6, except that the payload is
allowed to contain a multiple of 1024 bytes (non-final BERT block) or
more than 1024 bytes (final BERT block).

The recipient of a non-final BERT block (M=1) conceptually partitions
the payload into a sequence of 1024-byte blocks and acts exactly as
if it had received this sequence in conjunction with block numbers
starting at, and sequentially increasing from, the block number given
in the Block option.  In other words, the entire BERT block is
positioned at the byte position that results from multiplying the
block number with 1024.  The position of further blocks to be
transferred is indicated by incrementing the block number by the
number of elements in this sequence (i.e., the size of the payload
divided by 1024 bytes).

As with SZX == 6, the recipient of a final BERT block (M=0) simply
appends the payload at the byte position that is indicated by the
block number multiplied with 1024.

The following sub-sections provide a few examples that involve
 BERT options.  Extending the notation used in
that section, a value of SZX == 7 is shown as "BERT", or as
"BERT(nnn)" to indicate a payload of size nnn.

In all these examples, a Block option is shown in a decomposed way
indicating the kind of Block option (1 or 2) followed by a colon, and
then the block number (NUM), more bit (M), and block size exponent
(2**(SZX+4)) separated by slashes.  E.g., a Block2 Option value of 33
would be shown as 2:2/0/32), or a Block1 Option value of 59 would be
shown as 1:3/1/128.

## Example: GET with BERT Blocks

The first example, see {{fig-bert1}}, shows a GET request with a response that
is split into three BERT blocks.  The first response contains 3072
bytes of payload; the second, 5120; and the third, 4711.  Note how
the block number increments to move the position inside the response
body forward.

~~~~
CLIENT                                       SERVER
  |                                            |
  | GET, /status                       ------> |
  |                                            |
  | <------   2.05 Content, 2:0/1/BERT(3072)   |
  |                                            |
  | GET, /status, 2:3/0/BERT           ------> |
  |                                            |
  | <------   2.05 Content, 2:3/1/BERT(5120)   |
  |                                            |
  | GET, /status, 2:8/0/BERT          ------>  |
  |                                            |
  | <------   2.05 Content, 2:8/0/BERT(4711)   |
~~~~
{: #fig-bert1 title='GET with BERT blocks.'}

## Example: PUT with BERT Blocks

The following example, see {{fig-bert2}}, demonstrates a PUT exchange with
BERT blocks.

~~~~
CLIENT                                        SERVER
  |                                             |
  | PUT, /options, 1:0/1/BERT(8192)     ------> |
  |                                             |
  | <------   2.31 Continue, 1:0/1/BERT         |
  |                                             |
  | PUT, /options, 1:8/1/BERT(16384)    ------> |
  |                                             |
  | <------   2.31 Continue, 1:8/1/BERT         |
  |                                             |
  | PUT, /options, 1:24/0/BERT(5683)    ------> |
  |                                             |
  | <------   2.04 Changed, 1:24/0/BERT         |
  |                                             |
~~~~
{: #fig-bert2 title='PUT with BERT blocks.'}

# CoAP over WebSockets {#overview}

CoAP over WebSockets can be used in a number of configurations. The
most basic configuration is a CoAP client seeking to retrieve or
update a CoAP resource located at a CoAP server that exposes a
WebSocket endpoint ({{arch-1}}). The CoAP client takes
the role of the WebSocket client, establishes a WebSocket Connection
and sends a CoAP request, to which the CoAP server returns a CoAP
response. The WebSocket Connection can be used for any number of
requests.

~~~~
 ___________                            ___________
|           |                          |           |
|          _|___      requests      ___|_          |
|   CoAP  /  \  \  ------------->  /  /  \  CoAP   |
|  Client \__/__/  <-------------  \__\__/ Server  |
|           |         responses        |           |
|___________|                          |___________|
        WebSocket  =============>  WebSocket
          Client     Connection     Server
~~~~
{: #arch-1 title='CoAP Client (WebSocket client) accesses CoAP Server (WebSocket server)' artwork-align="center" }

The challenge in this configuration is to identify resource in the
namespace of the CoAP server:
When the WebSocket Protocol is used by a dedicated client directly
(i.e., not from a web page through a web browser), the client can
connect to any WebSocket endpoint. This means it is necessary that
the client is able to determine both the WebSocket endpoint (identified
by a "ws" or "wss" URI) and the path and query of the CoAP resource within
that endpoint from the same URI. When the WebSocket Protocol is used
from a web page, the choices are more limited {{RFC6454}}, but the challenge persists.

{{uris}} proposes a new "coap+ws" URI scheme that
identifies both a WebSocket endpoint and a resource within that
endpoint as follows:

~~~~
      coap+ws://example.org/sensors/temperature?u=Cel
           \______  ______/\___________  ___________/
                  \/                   \/
                                     Uri-Path: "sensors"
ws://example.org/.well-known/coap    Uri-Path: "temperature"
                                     Uri-Query: "u=Cel"
~~~~
{: #uri-example title='The "coap+ws" URI Scheme' artwork-align="center" }

Another possible configuration is to set up a CoAP forward proxy
at the WebSocket endpoint. Depending on what transports are available
to the proxy, it could forward the request to a CoAP server with a
CoAP UDP endpoint ({{arch-2}}), an SMS endpoint
(a.k.a.&nbsp;mobile phone), or even another WebSocket endpoint. The
client specifies the resource to be updated or retrieved in the
Proxy-URI Option.


~~~~
 ___________                ___________                ___________
|           |              |           |              |           |
|          _|___        ___|_         _|___        ___|_          |
|   CoAP  /  \  \ ---> /  /  \ CoAP  /  \  \ ---> /  /  \  CoAP   |
|  Client \__/__/ <--- \__\__/ Proxy \__/__/ <--- \__\__/ Server  |
|           |              |           |              |           |
|___________|              |___________|              |___________|
        WebSocket ===> WebSocket      UDP            UDP
          Client        Server      Client          Server
~~~~
{: #arch-2 title='CoAP Client (WebSocket client) accesses CoAP Server (UDP server) via a CoAP proxy (WebSocket server/UDP client)' artwork-align="center"}

A third possible configuration
is a CoAP server running inside a web browser
({{arch-3}}). The web browser initially connects to a
WebSocket endpoint and is then reachable through the WebSocket
server. When no connection exists, the CoAP server is not reachable;
it therefore can be considered a
[Sleepy Endpoint (SEP)](#I-D.dijk-core-sleepy-reqs).
Because the WebSocket server is the only way to reach the CoAP
server, the CoAP proxy should be a Reverse Proxy.


~~~~
 ___________                ___________                ___________
|           |              |           |              |           |
|          _|___        ___|_         _|___        ___|_          |
|   CoAP  /  \  \ ---> /  /  \ CoAP  /  /  \ ---> /  \  \  CoAP   |
|  Client \__/__/ <--- \__\__/ Proxy \__\__/ <--- \__/__/ Server  |
|           |              |           |              |           |
|___________|              |___________|              |___________|
           UDP            UDP      WebSocket <=== WebSocket
         Client          Server      Server        Client
~~~~
{: #arch-3 title='CoAP Client (UDP client) accesses sleepy CoAP Server (WebSocket client) via a CoAP proxy (UDP server/WebSocket server)' artwork-align="center"}

Further configurations are possible, including those where a
WebSocket Connection is established through an HTTP proxy.

CoAP over WebSockets is intentionally very similar to CoAP as defined
over UDP. Therefore, instead of presenting CoAP over WebSockets as a
new protocol, this document specifies it as a series of deltas from
{{RFC7252}}.

## Opening Handshake {#handshake}

Before CoAP requests and responses can be exchanged, a WebSocket
Connection needs to be established as defined in Section 4 of
{{RFC6455}}. {{handshake-example}} shows an example.

The WebSocket client MUST include the subprotocol name "coap" in
the list of protocols, which indicates support for the protocol
defined in this document. Any later, incompatible versions of
CoAP or CoAP over WebSockets will use a different subprotocol
name.

The WebSocket client includes the hostname of the WebSocket server
in the Host header field of its handshake as per {{RFC6455}}. The Host
header field also indicates the default
value of the Uri-Host Option in requests from the WebSocket client
to the WebSocket server.


~~~~
GET /.well-known/coap HTTP/1.1
Host: example.org
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: coap
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: coap
~~~~
{: #handshake-example title='Example of an Opening Handshake' artwork-align="center"}


## Message Format {#websocket-message-format}

Once a WebSocket Connection has been established, CoAP requests and
responses can be exchanged as WebSocket messages. Since CoAP uses a
binary message format, the messages are transmitted in binary data
frames as specified in Sections 5 and 6 of {{RFC6455}}.

The message format is very similar to the format specified for
[CoAP over UDP](#RFC7252). The differences
are as follows:

* Since the underlying TCP connection provides retransmissions and
  deduplication, there is no need for the reliability mechanisms
  provided by CoAP over UDP. This means the "T" and "Message ID" fields in
  the CoAP message header can be elided.

* Furthermore, since the CoAP version is already negotiated during
  the opening handshake, the "Ver" field can be elided as well.



~~~~
  0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   R   |  TKL  |      Code     |    Token (TKL bytes) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   Options (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|1 1 1 1 1 1 1 1|    Payload (if any) ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~
{: #ws-message-format title='CoAP Message Format over WebSockets' artwork-align="center"}

The resulting message format is shown in {{ws-message-format}}. The
four most-significant bits
of the first byte are reserved (R) and MUST be set to zero. The
remaining fields and structure are the same as defined in {{RFC7252}}.

Requests and response messages can be fragmented as specified in
Section 5.4 of {{RFC6455}}, though typically they are
sent unfragmented as they tend to be small and fully buffered before
transmission. The WebSocket protocol does not provide
means for multiplexing; if it is not desirable for a large message to
monopolize the connection, requests and responses can be transferred in a
blockwise fashion as defined in {{I-D.ietf-core-block}}.

Messages MUST NOT be Empty (Code 0.00), i.e., messages always carry
either a request or a response.


## Message Transmission {#requests-responses}

CoAP requests and responses are exchanged asynchronously over the
WebSocket Connection, i.e., a CoAP client can send multiple requests
without waiting for a response and the CoAP server can return
responses in any order. Responses MUST be returned over the same
connection as the originating request. Concurrent requests are
differentiated by their Token, which is scoped locally to the
connection.

The connection is bi-directional, so requests can be sent both by
the entity that established the connection and the remote host.

Retransmission and deduplication of messages is provided by the
WebSocket Protocol. CoAP over WebSockets therefore does not make a
distinction between Confirmable or Non-Confirmable messages, and does
not provide Acknowledgement or Reset messages.

Since the WebSocket Protocol provides ordered delivery of messages,
the mechanism for reordering detection when
[observing resources](#RFC7641) is not needed. The value of the
Observe Option in notifications therefore MAY be empty on transmission
and MUST be ignored on reception.


## Connection Health {#liveliness}

When a client does not receive any response for some time after
sending a CoAP request (or, similarly, when a client observes a
resource and it does not receive any notification for some time),
the connection between the WebSocket client and the WebSocket
server may be lost or temporarily disrupted without the client
being aware of it.

To check the health of the WebSocket Connection (and thereby of all
active requests, if any), the client can send a Ping frame or an
unsolicited Pong frame as specified in Section 5.5 of
{{RFC6455}}. There is no way to retransmit a request without
creating a new one. Re-registering interest in a resource is
permitted, but entirely unnecessary.

## Closing the Connection {#close}

The WebSocket Connection is closed as specified in Section 7 of {{RFC6455}}.

All requests for which the CoAP client has not received
a response yet are cancelled when the connection is closed.
If the client observes one or more resources over the WebSocket
Connection, then the CoAP server (or intermediary in the role of
the CoAP server) MUST remove all entries associated with the client
from the lists of observers when the connection is closed.

# CoAP URIs {#URI}

CoAP over UDP {{RFC7252}} defines the "coap" and "coaps" URI schemes for
identifying CoAP resources and providing a means of locating the resource. 

## CoAP over TCP and TLS URIs

CoAP over TCP uses "coap_tcp" URI scheme. CoAP over TLS uses the "coaps+tcp"
scheme. The rules from Section 6 of {{RFC7252}} apply to both of these URI schemes.

{{RFC7252}}, Section 8 (Multicast CoAP) is not applicable to these schemes.

Resources made available via one of the "coap+tcp" or "coaps+tcp" schemes
have no shared identity with the other scheme or with the "coap" or
"coaps" scheme, even if their resource identifiers indicate the
same authority (the same host listening to the same port).
The schemes constitute distinct namespaces and, in combination with
the authority, are considered to be distinct
origin servers.

### coap+tcp URI scheme

~~~~
coap-tcp-URI = "coap+tcp:" "//" host [ ":" port ] path-abempty
               [ "?" query ]
~~~~

The semantics defined in {{RFC7252}}, Section 6.1, apply to this
URI scheme, with the following changes:

* The port subcomponent indicates the TCP port
at which the CoAP server is located.  (If it is empty or not given,
then the default port 5683 is assumed, as with UDP.)

### coaps+tcp URI scheme {#coapstcp-uri-scheme}

~~~~
coaps-tcp-URI = "coaps+tcp:" "//" host [ ":" port ] path-abempty
                [ "?" query ]
~~~~

The semantics defined in {{RFC7252}}, Section 6.2, apply to this
URI scheme, with the following changes:

* The port subcomponent indicates the TCP port at which the TLS server
  for the CoAP server is located.  If it is empty or not given, then
  the default port 443 is assumed (this is different from the default
  port for "coaps", i.e., CoAP over DTLS over UDP).

* When CoAP is exchanged over TLS port 443, the "TLS Application
  Layer Protocol Negotiation Extension" {{-alpn}} MUST be used to allow 
  demultiplexing at the server-side.

## CoAP over WebSockets URIs {#uris}

For the first configuration discussed in {{overview}},
this document defines two new URIs schemes that can be used for
identifying CoAP resources and providing a means of locating these
resources: "coap+ws" and "coap+wss".

Similar to the "coap" and "coaps" schemes, the "coap+ws" and
"coap+wss" schemes organize resources hierarchically under a CoAP
origin server. The key difference is that the server is potentially
reachable on a WebSocket endpoint instead of a UDP endpoint.

The WebSocket endpoint is identified by a "ws" or "wss" URI
that is composed of the authority part of the "coap+ws" or
"coap+wss" URI, respectively, and the well-known path
"/.well-known/coap" {{RFC5785}}.
The path and query parts of a "coap+ws" or "coap+wss" URI
identify a resource within the specified endpoint which can be
operated on by the methods defined by the CoAP protocol.

The syntax of the "coap+ws" and "coap+wss" URI schemes is specified
below in Augmented Backus-Naur Form (ABNF) {{RFC5234}}.
The definitions of "host", "port", "path-abempty" and "query" are the
same as in {{RFC3986}}.


~~~~
coap-ws-URI =
   "coap+ws:" "//" host [ ":" port ] path-abempty [ "?" query ]

coap-wss-URI =
   "coap+wss:" "//" host [ ":" port ] path-abempty [ "?" query ]
~~~~
{: artwork-align="center"}

The port component is OPTIONAL; the default for "coap+ws" is port
80, while the default for "coap+wss" is port 443.

Fragment identifiers are not part of the request URI and thus MUST
NOT be transmitted in a WebSocket handshake or in the URI options
of a CoAP request.

### Decomposing and Composing URIs

The steps for decomposing a "coap+ws" or "coap+wss" URI into
CoAP options are the same as specified in Section 6.4 of {{RFC7252}}
with the following changes:

* The \<scheme> component MUST be "coap+ws" or "coap+wss"
  when converted to ASCII lowercase.

* A Uri-Host Option MUST only be included in a request when
  the \<host> component does not equal the uri-host
  component in the Host header field in the WebSocket
  handshake.

* A Uri-Port Option MUST only be included in a request if
  \|port\| does not equal the port component in the Host header
  field in the WebSocket handshake.

The steps to construct a URI from a request's options are
changed accordingly.

# Security Considerations {#security}

Implementations of CoAP MUST use TLS version 1.2 or higher for CoAP over TLS.
The general TLS usage guidance in {{RFC7525}} SHOULD be followed.

Guidelines for use of cipher suites and TLS extensions can be found in {{I-D.ietf-dice-profile}}.

TLS does not protect the TCP header. This may, for example, 
allow an on-path adversary to terminate a TCP connection prematurely 
by spoofing a TCP reset message.

CoAP over WebSockets and CoAP over TLS-secured WebSockets do not
introduce additional security issues beyond CoAP and DTLS-secured CoAP
respectively {{RFC7252}}. The security considerations of {{RFC6455}} apply.

# IANA Considerations {#iana}

## Service Name and Port Number Registration

IANA is requested to assign the port number 5683 and the service name "coap+tcp",
in accordance with {{RFC6335}}.

Service Name.
:   coap+tcp

Transport Protocol.
:   tcp

Assignee.
:   IESG \<iesg@ietf.org>

Contact.
:   IETF Chair \<chair@ietf.org>

Description.
:   Constrained Application Protocol (CoAP)

Reference.
:   [RFCthis]

Port Number.
:   5683
{: vspace='0'}


Similarly, IANA is requested to assign the
service name "coaps+tcp", in accordance with {{RFC6335}}.
However, no separate port number is used for "coaps" over TCP; instead,
the ALPN protocol ID defined in {{alpnpid}} is used over port 443.

Service Name.
:   coaps+tcp

Transport Protocol.
:   tcp

Assignee.
:   IESG \<iesg@ietf.org>

Contact.
:   IETF Chair \<chair@ietf.org>

Description.
:   Constrained Application Protocol (CoAP)

Reference.
:  {{-alpn}}, [RFCthis]

Port Number.
:   443  (see also {{alpnpid}} of [RFCthis]})
{: vspace='0'}


## URI Scheme Registration

This document registers two new URI schemes, namely "coap+tcp" and
"coaps+tcp", for the use of CoAP over TCP and for CoAP over TLS over
TCP, respectively. The "coap+tcp" and "coaps+tcp" URI schemes can thus
be compared to the "http" and "https" URI schemes.

The syntax of the "coap" and "coaps" URI schemes is specified in
Section 6 of {{RFC7252}} and the present document re-uses their
semantics for "coap+tcp" and "coaps+tcp", respectively, with the
exception that TCP, or TLS over TCP is used as a transport protocol.

IANA is requested to add these new URI schemes to the registry
established with {{-urireg}}.

### coap+ws

This document requests the registration of the Uniform Resource
Identifier (URI) scheme "coap+ws". The registration request complies
with {{RFC4395}}.

URL scheme name.
:	coap+ws

Status.
:	Permanent

URI scheme syntax.
:	Defined in Section N of [RFCthis]

URI scheme semantics.
:	The "coap+ws" URI scheme provides a way to identify resources that
	are potentially accessible over the Constrained Application Protocol (CoAP)
	using the WebSocket Protocol.

Encoding considerations.
:	The scheme encoding conforms to the encoding rules established for URIs
	in {{RFC3986}}, i.e., internationalized and reserved characters are expressed
	using UTF-8-based percent-encoding.

Applications/protocols that use this URI scheme name.
:	The scheme is used by CoAP endpoints to access CoAP resources using the WebSocket protocol.

Interoperability considerations.
:	None.

Security Considerations.
:	See Section N of [RFCthis]

Contact.
:	IETF chair \<chair@ietf.org>

Author/Change controller.
:	IESG \<iesg@ietf.org>

References.
:	[RFCthis]
{: vspace='0'}

### coap+wss
This document requests the registration of the Uniform Resource
Identifier (URI) scheme "coap+wss". The registration request complies
with {{RFC4395}}.

URL scheme name.
:	coap+wss

Status.
:	Permanent

URI scheme syntax.
:	Defined in Section N of [RFCthis]

URI scheme semantics.
:	The "coap+ws" URI scheme provides a way to identify resources that
	are potentially accessible over the Constrained Application Protocol (CoAP)
	using the WebSocket Protocol secured with Transport Layer Security (TLS).

Encoding considerations.
:	The scheme encoding conforms to the encoding rules established for URIs
	in {{RFC3986}}, i.e., internationalized and reserved characters are expressed
	using UTF-8-based percent-encoding.

Applications/protocols that use this URI scheme name.
:	The scheme is used by CoAP endpoints to access CoAP resources using the WebSocket protocol
	secured with TLS.

Interoperability considerations.
:	None.

Security Considerations.
:	See Section N of [RFCthis]

Contact.
:	IETF chair \<chair@ietf.org>

Author/Change controller.
:	IESG \<iesg@ietf.org>

References.
:	[RFCthis]
{: vspace='0'}

## Well-Known URI Suffix Registration

IANA is requested to register the 'coap' well-known URI in the "Well-Known URIs" registry. This
registration request complies with {{RFC5785}}:

URI Suffix.
:	coap

Change controller.
:	IETF

Specification document(s).
:	[RFCthis]

Related information.
:	None.
{: vspace='0'}

## ALPN Protocol ID {#alpnpid}

IANA is requested to assign the following value in the registry
"Application Layer Protocol Negotiation (ALPN) Protocol IDs" created
by {{-alpn}}:

Protocol.
:   CoAP

Identification Sequence.
:   0x63 0x6f 0x61 0x70 ("coap")

Reference.
:   [RFCthis]
{: vspace='0'}

## WebSocket Subprotocol Registration

IANA is requested to register the WebSocket CoAP subprotocol under the "WebSocket Subprotocol Name Registry":

Subprotocol Identifier.
:	coap

Subprotocol Common Name.
:	Constrained Application Protocol (CoAP)

Subprotocol Definition.
:	[RFCthis]
{: vspace='0'}


--- back

# CoAP over WebSocket Examples {#examples}

This section gives examples for the first two configurations
discussed in {{overview}}.

An example of the process followed by a CoAP client to retrieve the
representation of a resource identified by a "coap+ws" URI might be as
follows. {{example-1}} below illustrates the WebSocket and
CoAP messages exchanged in detail.

1. The CoAP client obtains the URI
  \<coap+ws://example.org/sensors/temperature?u=Cel>,
  for example, from a resource representation that it retrieved
  previously.

1. It establishes a WebSocket Connection to the endpoint URI composed
  of the authority "example.org" and the well-known path
  "/.well-known/coap", \<ws://example.org/.well-known/coap>.

1. It sends a single-frame, masked, binary message containing a CoAP
  request. The request indicates the target resource with the
  Uri-Path ("sensors", "temperature") and Uri-Query ("u=Cel")
  options.

1. It waits for the server to return a response.

1. The CoAP client uses the connection for further requests, or the
  connection is closed.



~~~~
   CoAP        CoAP
  Client      Server
(WebSocket  (WebSocket
  Client)     Server)

     |          |
     |          |
     +=========>|  GET /.well-known/coap HTTP/1.1
     |          |  Host: example.org
     |          |  Upgrade: websocket
     |          |  Connection: Upgrade
     |          |  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
     |          |  Sec-WebSocket-Protocol: coap
     |          |  Sec-WebSocket-Version: 13
     |          |
     |<=========+  HTTP/1.1 101 Switching Protocols
     |          |  Upgrade: websocket
     |          |  Connection: Upgrade
     |          |  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
     |          |  Sec-WebSocket-Protocol: coap
     |          |
     |          |
     +--------->|  Binary frame (opcode=%x2, FIN=1, MASK=1)
     |          |    +-------------------------+
     |          |    | GET                     |
     |          |    | Token: 0x53             |
     |          |    | Uri-Path: "sensors"     |
     |          |    | Uri-Path: "temperature" |
     |          |    | Uri-Query: "u=Cel"      |
     |          |    +-------------------------+
     |          |
     |<---------+  Binary frame (opcode=%x2, FIN=1, MASK=0)
     |          |    +-------------------------+
     |          |    | 2.05 Content            |
     |          |    | Token: 0x53             |
     |          |    | Payload: "22.3 Cel"     |
     |          |    +-------------------------+
     :          :
     :          :
     |          |
     +--------->|  Close frame (opcode=%x8, FIN=1, MASK=1)
     |          |
     |<---------+  Close frame (opcode=%x8, FIN=1, MASK=0)
     |          |
~~~~
{: #example-1 title='A CoAP client retrieves the representation of a resource identified by a "coap+ws" URI'}

{{example-2}} shows how a CoAP client uses a CoAP
forward proxy with a WebSocket endpoint to retrieve the representation
of the resource "coap://[2001:DB8::1]/". The use of the forward
proxy and the address of the WebSocket endpoint are determined by the
client from local configuration rules. The request URI is specified
in the Proxy-Uri Option. Since the request URI uses the "coap" URI
scheme, the proxy fulfills the request by issuing a Confirmable GET
request over UDP to the CoAP server and returning the response over
the WebSocket connection to the client.


~~~~
   CoAP        CoAP       CoAP
  Client      Proxy      Server
(WebSocket  (WebSocket    (UDP
  Client)     Server)   Endpoint)

     |          |          |
     +--------->|          |  Binary frame (opcode=%x2, FIN=1, MASK=1)
     |          |          |    +------------------------------------+
     |          |          |    | GET                                |
     |          |          |    | Token: 0x7d                        |
     |          |          |    | Proxy-Uri: "coap://[2001:DB8::1]/" |
     |          |          |    +------------------------------------+
     |          |          |
     |          +--------->|  CoAP message (Ver=1, T=Con, MID=0x8f54)
     |          |          |    +------------------------------------+
     |          |          |    | GET                                |
     |          |          |    | Token: 0x0a15                      |
     |          |          |    +------------------------------------+
     |          |          |
     |          |<---------+  CoAP message (Ver=1, T=Ack, MID=0x8f54)
     |          |          |    +------------------------------------+
     |          |          |    | 2.05 Content                       |
     |          |          |    | Token: 0x0a15                      |
     |          |          |    | Payload: "ready"                   |
     |          |          |    +------------------------------------+
     |          |          |
     |<---------+          |  Binary frame (opcode=%x2, FIN=1, MASK=0)
     |          |          |    +------------------------------------+
     |          |          |    | 2.05 Content                       |
     |          |          |    | Token: 0x7d                        |
     |          |          |    | Payload: "ready"                   |
     |          |          |    +------------------------------------+
     |          |          |
~~~~
{: #example-2 title='A CoAP client retrieves the representation of a resource identified by a "coap" URI via a WebSockets-enabled CoAP proxy'}

# Change Log

The RFC Editor is requested to remove this section at publication.

## Since draft-core-coap-tcp-tls-02

Merged draft-savolainen-core-coap-websockets-07

# Acknowledgements {#acknowledgements}
{: numbered="no"}

We would like to thank Stephen Berard, Geoffrey Cristallo, 
Olivier Delaby, Christian Groves, Nadir Javed,
Michael Koster, Matthias Kovatsch, Achim Kraus, David Navarro,
Szymon Sasin, Zach Shelby, Andrew Summers, Julien Vermillard, 
and Gengyu Wei for their feedback.

# Contributors {#contributors}
{: numbered="no"}

	Teemu Savolainen
	Nokia Technologies
	Hatanpaan valtatie 30
	Tampere FI-33100
	Finland

	Email: teemu.savolainen@nokia.com

<!--  LocalWords:  TCP CoAP UDP firewalling firewalled TLS IP SCTP
 -->
<!--  LocalWords:  DCCP IoT optimizations ACKs acknowledgement TKL
 -->
<!--  LocalWords:  prepending URI DTLS demultiplexing demultiplex pre
 -->
<!--  LocalWords:  IANA ALPN Middleboxes NATs ACK acknowledgements
 -->
<!--  LocalWords:  datagram prepended CBOR namespaces subcomponent
 -->
<!--  LocalWords:  Assignee Confirmable untyped
 -->
