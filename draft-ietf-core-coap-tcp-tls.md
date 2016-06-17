---
stand_alone: true
ipr: trust200902
docname: draft-ietf-core-coap-tcp-tls-latest
cat: std
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
title: A TCP and TLS Transport for the Constrained Application Protocol (CoAP)
abbrev: TCP/TLS Transport for CoAP
area: Applications Area (app)
wg: CORE
date: 2016-04-21
author:
- role: editor
  ins: C. Bormann
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
normative:
  RFC5246: tls
  RFC7252: coap
  RFC2119: bcp14
  RFC7595: urireg
  RFC0793: tcp
  RFC7301: alpn
  I-D.ietf-dice-profile:
informative:
  I-D.bormann-core-cocoa: cocoa
  I-D.ietf-core-block: block
  I-D.bormann-core-block-bert: bert
  RFC0768: udp
  RFC6347: dtls
  RFC6335: portreg
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

The Hypertext Transfer Protocol (HTTP) was designed with TCP as the underlying
transport protocol.  The Constrained Application Protocol (CoAP), while
inspired by HTTP, has been defined to make use of UDP instead of TCP.  Therefore,
reliable delivery and a simple congestion control and flow control mechanism
are provided by the message layer of the CoAP protocol.

A number of environments benefit from the use of CoAP directly over a reliable
byte stream such as TCP, which already provides these services.  This document
defines the use of CoAP over TCP as well as CoAP over TLS.

--- middle

# Introduction {#introduction}

The [Constrained Application Protocol (CoAP)](#RFC7252) was designed
for Internet of Things (IoT) deployments, assuming that UDP can be
used unimpeded --- UDP {{RFC0768}}, or DTLS {{RFC6347}} over UDP; it is a good choice for
transferring small amounts of data across networks that follow the IP
architecture.
Some CoAP deployments, however, may have to integrate well with
existing enterprise infrastructure, where the use of UDP-based
protocols may not be well-received or may even be blocked by firewalls.
Middleboxes that are unaware of CoAP usage for IoT can make the use of UDP
brittle, resulting in lost or malformed packets.

Where NATs are still present,
CoAP over TCP can also help with their traversal. NATs often calculate
expiration timers
based on the transport layer protocol being used by application protocols.
Many NATs are built around the assumption that a transport layer protocol
such as
TCP gives them additional information about the session life cycle
and keep TCP-based NAT bindings around for a longer period. UDP, on the other
hand,
does not provide such information to a NAT and timeouts tend to be
much shorter, as research confirms {{HomeGateway}}.

Some environments may also benefit from the more sophisticated congestion
control capabilities provided by many TCP implementations.
(Note that there is ongoing work to add more elaborate congestion control
to CoAP as well, see {{-cocoa}}.)

Finally, CoAP may be integrated into a Web environment where the front-end
uses CoAP from IoT devices to a cloud infrastructure but the CoAP messages
are then transported in TCP between the back-end services.
A TCP-to-UDP gateway can be
used at the cloud boundary to talk to the UDP-based IoT.

To make IoT devices work smoothly in these demanding environments, CoAP
needs
to make use of a different transport protocol, namely TCP {{RFC0793}},
in some situations secured by TLS {{RFC5246}}.

The present document
describes a shim header that conveys length information about
each CoAP message.  Modifications to CoAP beyond the replacement
of the message layer
(e.g., to introduce further optimizations) are intentionally avoided.


# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in {{RFC2119}}.


# Constrained Application Protocol

The interaction model of CoAP over TCP is very similar to the one for
CoAP over UDP, with the key difference that using TCP voids the need to
provide certain transport layer protocol features, such as reliable
delivery, fragmentation and reassembly, as well as congestion control,
at the CoAP level. The protocol stack is illustrated in {{stack}} (derived
from {{RFC7252}}, Figure 1).


~~~~
        +----------------------+
        |      Application     |
        +----------------------+
        +----------------------+
        |  Requests/Responses  |  CoAP (RFC7252)
        |----------------------|
        |    Message adapter   |  This Document
        +----------------------+
        +-----------+    ^
        |    TLS    |    |
        +-----------+    v
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

A
response is sent back as defined
in {{RFC7252}}, as
illustrated in {{fig-untyped2}} (derived from {{RFC7252}}, Figure 6).

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
{: #fig-untyped2 title=Untyped messages carrying Request/Response.'}


# Message Format

The CoAP message format defined in {{RFC7252}}, as shown in 
{{CoAP-Header}}, relies on the datagram transport (UDP, or DTLS over
UDP) for keeping the individual messages separate.


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

## Discussion

One observation is that, over a reliable byte stream transport, the message
size limitations defined in
Section 4.6 of {{RFC7252}} are no longer strictly necessary.
Consenting [^how] implementations may want to interchange messages
with payload sizes larger than 1024 bytes, potentially also obviating the
need for the Block protocol {{-block}}.  It must be noted that
entirely getting rid of the block protocol is
not a generally applicable solution, as:

* a UDP-to-TCP gateway may simply not have the context to convert a
  message with a Block option into the equivalent exchange without any
  use of a Block option;
* large messages might also cause undesired head-of-line blocking;
* the 2-byte message length field causes another, larger upper bound to the
  message length.

{{-bert}} proposes to extend the block-wise transfer protocol to allow
for larger block sizes as are possible over TCP and TLS.

[^how]: There is currently no defined way to arrive at this consent.
{: source="cabo"}

The general assumption is therefore that the block protocol will
continue to be used over TCP, even if TCP-based applications
occasionally do exchange messages with payload sizes larger than desirable in UDP.

# Message Transmission

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
  use of a Block option.[^1]

[^1]: I don't understand this argument.
{: source="cabo"}

This specification extends the block protocol to be
able to make use of the larger messages over a reliable transport.
This extension is called 'Block-Wise Extension for Reliable Transport
(BERT)'.

{{-block}} uses a message size of maximum 1024 bytes and this
specification extends this maximum size to 1024 KiB (= 1 MiB) by
re-interpreting the length information.[^2]

[^2]: 1 MiB is an arbitrary limit.  This will be fixed if we have
        something like Max-Message-Size in the signalling draft.
{: source="cabo"}

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


# CoAP URI {#URI}

CoAP {{RFC7252}} defines the "coap" and "coaps" URI schemes for
identifying CoAP resources and providing a means of locating the
resource. RFC 7252 defines these resources for use with CoAP over UDP.

The present specification introduces two new URI schemes, namely "coap+tcp"
and "coaps+tcp".  The rules from Section 6 of {{RFC7252}} apply to
these two new URI schemes.

{{RFC7252}}, Section 8 (Multicast CoAP), does not apply to the URI
schemes defined in the present specification.

Resources made available via one of the "coap+tcp" or "coaps+tcp" schemes
have no shared identity with the other scheme or with the "coap" or
"coaps" scheme, even if their resource identifiers indicate the
same authority (the same host listening to the same port).
The schemes constitute distinct namespaces and, in combination with
the authority, are considered to be distinct
origin servers.

## coap+tcp URI scheme

~~~~
coap-tcp-URI = "coap+tcp:" "//" host [ ":" port ] path-abempty
               [ "?" query ]
~~~~

The semantics defined in {{RFC7252}}, Section 6.1, apply to this
URI scheme, with the following changes:

* The port subcomponent indicates the TCP port
at which the CoAP server is located.  (If it is empty or not given,
then the default port 5683 is assumed, as with UDP.)

## coaps+tcp URI scheme {#coapstcp-uri-scheme}

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

* When CoAP is exchanged over TLS port 443 then the "TLS Application
  Layer Protocol Negotiation Extension" {{-alpn}} MUST be used to allow demultiplexing
  at the server-side unless out-of-band information ensures that the
  client only interacts with a server that is able to demultiplex CoAP
  messages over port 443. This would, for example, be true for many
  IoT deployments where clients are pre-configured to
  only ever talk with specific servers.[^alwaysalpn]

[^alwaysalpn]: Shouldn't we simply always require ALPN?
    The protocol should not be defined in such a way that it
    depends on some undefined pre-configuration mechanism.
{: source="cabo"}


# Security Considerations {#security}

This document defines how to convey CoAP over TCP and TLS. It does not
introduce new vulnerabilities beyond those described already in the
CoAP specification. CoAP {{RFC7252}} makes use of DTLS 1.2 and this
specification consequently uses TLS 1.2 {{RFC5246}}. CoAP MUST NOT be
used with older versions of TLS. Guidelines for use of cipher suites
and TLS extensions can be found in {{I-D.ietf-dice-profile}}.


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


## URI Schemes

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


## ALPN Protocol ID {#alpnpid}

IANA is requested to assign the following value in the registry
"Application Layer Protocol Negotiation (ALPN) Protocol IDs" created
by {{-alpn}}:

Protocol:
:   CoAP

Identification Sequence:
:   0x63 0x6f 0x61 0x70 ("coap")

Reference:
:   [RFCthis]
{: vspace='0'}


# Acknowledgements {#acknowledgements}

We would like to thank Stephen Berard, Geoffrey Cristallo, Olivier Delaby,
Michael Koster, Matthias Kovatsch, Szymon Sasin, Andrew Summers, and Zach Shelby for their
feedback.


--- back

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
