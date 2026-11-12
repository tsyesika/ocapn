# Warning: This is a draft specification likely to undergo significant change

These documents are draft specifications which will likely undergo significant
change as the group converges on the design of OCapN. If you're interested in
being part of that work, please join!

Authors: Jessica Tallon & The OCapN Pre-standardization Group.

# Additional Documents

- [OCapN CapTP](../CapTP%20Specification.md): Secure messaging protocol for
  object communication.
- [OCapN Locators](../Locators.md): Representation of object references for both
  in-band and out-of-band usage.
- [Noise Protocol](https://noiseprotocol.org/noise.html): The Noise Protocol
  Framework specification.

# Introduction

This netlayer provides strong encryption of the messages sent with it, and
provides peer authentication. The netlayer, unlike many others, is what is known
as a meta-netlayer. Meta-netlayers don't specify a specific transport mechanism
but use other OCapN netlayers as a transport layer, this is done by layering
OCapN on top of OCapN. The result of composing this netlayer with another
netlayer is the other netlayer doesn't need to provide encryption or
bidirectional peer authentication.

# Noise

This netlayer uses [Noise Protocol](https://noiseprotocol.org/noise.html)
framework. Noise Protocol provides many different patterns that can be used,
this netlayer uses the XK pattern. Noise allows for different key generation,
cipher suite and hashing algorithms. The Noise netlayer uses
[x25519](https://noiseprotocol.org/noise.html#the-25519-dh-functions) key
generation,
[AES-GCM](https://noiseprotocol.org/noise.html#the-aesgcm-cipher-functions)
cipher functions, and
[SHA-256](https://noiseprotocol.org/noise.html#the-sha256-hash-function) for its
wide availability. This means the Noise netlayer always uses
`Noise_XK_25519_AESGCM_SHA256` as the protocol name.

The implementation of Noise is out-of-scope for this document. Implementers are
referred to the Noise Protocol specification.

Noise imposes a maximum message limit of 2^64^-1 within a cipher state. OCapN
sessions are expected to be relatively short lived so it's unlikely for any
session to reach this limit. Each peer within a Noise session MUST ensure they
do not exceed this message limit. An implementation MUST abort the OCapN session
and if reconnect to prevent exceeding the message limit.

Each Noise netlayer is expected to generate a static key pair to be used with
Noise for all sessions. The public part of this static key forms the peer's
identity as described in [Noise Locator](#noise-locator) section.

# Objects

Since the netlayer is built on top of OCapN, several objects are defined by this
specification to ensure delivery of messages. This specification does not
require any specific design of the object framework, the specification describes
these objects in terms of the behavior upon messages sent to these objects. The
messaging between these objects is specified within the CapTP specification.

## [Noise Connector](#noise-connector)

The Noise connector object is how remote peers initiate new connections. A new
incoming connection occurs by messaging the Noise connector with a reference to
a new [outbox](#outbox) object.

The Noise connector then needs to create its own [outbox](#outbox) object which
is used by the sender to deliver messages to. This outbox is provided in
response to the message. See the
[establishing a connection](#establishing-connection) section for more details.

## [Outbox](#outbox)

This object is designed to accept new messages which are delivered to the peer
which spawned the outbox object. The object takes a single argument which is an
encrypted OCapN message or Noise handshake message. The message is either a:

- Bytearray: The encrypted data represented as a bytearray.
- A list of chunked messages with the first element being an integer
  representing the total length, followed by any number of bytearray chunks.

Noise has a maximum message size of 65535. Implementations should be careful to
ensure messages do not exceed this. If a message is sent that is beyond the
maximum message size, it should be chunked into smaller messages. The chunks are
encrypted individually in the order they appear in the list.

Messages are decrypted in order in which they appear and the final message is
concatenated to form a single OCapN message. The total length prefix is not
encrypted by noise and could be modified during transport. Peers MUST therefore
ensure to handle length prefixes which do not match length of the chunked
message.

Messages to the outbox have no return value, so implementations should send
messages to the object using `op:deliver-only`.

# [Noise Locator](#noise-locator)

The OCapN locator specifies the following:

- **Designator**: This is the Noise public key data encoded as base32 (without
  padding).
- **Transport**: Reserves the symbol "noise", however this can be used with
other netlayers to produce a composite netlayer. In those modes a different
transport may be selected.
- **Hints**: A hashmap with a single entry. The entry has the key set to
"connector", the value is the string representation of a sturdy reference to the
[Noise connector](#noise-connector) object for the peer.

# [Establishing a new connection](#establishing-connection)

When CapTP asks the Noise netlayer to establish a new Noise connection given a
specific [OCapN Peer Locator](#noise-locator) the netlayer should extract the
Noise static key from the designator and the Noise connector from the hints.

The peer should then create a new
[HandshakeState Object](https://noiseprotocol.org/noise.html#the-handshakestate-object)
with:

1. `handshake_pattern` set to the appropriate value for the `XK` pattern
2. `initiator` flag set to true
3. `prologue` a zero-length bytearry.
4. `s` set to peer's own static key pair
5. `rs` set to the decoded public key from the OCapN Peer Locator

The peer should then spawn a new [Outbox object](#outbox) and then arrange with
the CapTP connection it's using to retrieve a live reference from the Noise
connector sturdy reference from the OCapN Locator.

Once it has a live reference to the Noise connector, it should send that object
a message with a reference to its outbox actor. The message should be sent with
a `op:deliver`, as the return value will be the outbox for it to use.

e.g.
```
;; Imagine the Noise connector is exported at position 1 and it's exporting its
;; own outbox object at position 50
<op:deliver <desc:export 1>
            [<desc:import-object 50>]
            3
            listen-mode
            resolver>
```

Each peer MUST then go through the Noise handshake as specified in the Noise
Protocol specification. Noise permits a payload to be sent alongside the
handshake message, but since this payload has weaker security properties than
after the handshake, it's not used. The handshake payloads should therefore be
set to a zero-length bytearray.

Noise handshakes work on a turn-based approach where one peer writes a message
and the other peer then reads this message. The peers then switch roles after
each turn. The first turn the initiator writes a handshake message and then the
responder reads the message. Please refer to the Noise protocol specification
for more details.

If any errors occur during the Noise handshake, the side MUST terminate the
connection. If the handshake completes without errors, messages can then be
exchanged. Each sent message should be encrypted in order and delivered to the
[outbox object](#outbox) provided by the other peer. Messages should be read
and decrypted in order from the [outbox object](#outbox) created by them. These
messages should then be provided to CapTP.

# [Incoming connection](#incoming-connection)

As described in the [establishing a new connection](#establishing-connection)
section, a new incoming connection will be initiated by the initiating peer
sending a message to the [Noise connector object](#noise-connector). When a peer
receives this message, they should begin by setting up their own
[HandshakeState Object](https://noiseprotocol.org/noise.html#the-handshakestate-object)
with:

1. `handshake_pattern` set to the appropriate value for the `XK` pattern
2. `initiator` flag set to false
3. `prologue` a zero-length bytearry.
4. `s` set to peer's own static key pair

They do not need to set up any remote key as that is communicated to the peer in
the handshake messages.

The peer should create its own [outbox object](#outbox) which MUST be returned
from the call to the message to the connector object. This outbox object is used
by the initiating peer to deliver both Noise handshake messages and CapTP
messages (after a successful Noise handshake).

The handshake and exchange of messages should occur in the same way for both
initiating peers and responding peers as outlined in
[establishing a new connection](#establishing-connection).

# Funding

This project was funded through the
[NGI0 Commons Fund](https://nlnet.nl/commonsfund), a fund established by
[NLnet](https://nlnet.nl/) with financial support from the European Commission's
[Next Generation Internet](https://ngi.eu/) programme. Learn more on the
[NLnet project page](https://nlnet.nl/project/OCapN-federatedrelays/).

[<img src="https://nlnet.nl/logo/banner.svg" alt="Logo NLnet: abstract logo of four people seen from above" width="20%" />](https://nlnet.nl)
[<img src="https://nlnet.nl/image/logos/NGI0CommonsFund_tag.svg" alt="Logo NGI Zero Commons Fund: letterlogo shaped like a tag" width="20%" />](https://nlnet.nl/commonsfund)
