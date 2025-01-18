---
title: "Media over QUIC - Transfork"
abbrev: "moqtf"
category: info

docname: draft-lcurley-moq-transfork-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
v: 3
area: wit
workgroup: moq

author:

 -  fullname: Luke Curley
    organization: Discord
    email: kixelated@gmail.com

 -  fullname: Okutani Daichi

normative:
  moqt: I-D.ietf-moq-transport

informative:

--- abstract

MoqTransfork is designed to serve live tracks over a CDN to viewers with varying latency and quality targets: the entire spectrum between real-time and VOD.
MoqTransfork itself is a media agnostic transport, allowing relays and CDNs to forward the most important content under degraded networks without knowledge of codecs, containers, or even if the content is fully encrypted.
Higher level protocols specify how to use MoqTransfork to encode and deliver video, audio, messages, or any form of live content.

--- middle

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Fork

This draft is based on moq-transport-03 [moqt].
The concepts, motivations, and terminology are very similar on purpose.
When in doubt, refer to the upstream draft.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

But there are practical and conceptual flaws with MoqTransport that need to be addressed.
However it's been difficult to design such an experimental protocol via committee.
Despite years of arguments in person and on GitHub, we've yet to align on even the most critical property of the transport... how to utilize QUIC.
The result is inevitably an unwieldy "compromise", consisting of a modes for each party that make everything more difficult to support or explain.

In our RUSH to standardize a protocol, the QUICR solutions have led to WARP in ideals.

This fork is meant to be a constructive, alternative vision.
I would like to lead by example, demonstrating that you can support real media use-cases while simplifying the protocol.
The working group will keep making progress and hopefully many of these ideas will be incorporated.

The appendix contains a list of high level differences between MoqTransport and MoqTransfork.

# Concepts

MoqTransfork consists of:

- **Session**: An established connection between a client and server used to transmit any number of Tracks by path.
- **Track**: An series of Groups, each of which is delivered and decoded *independently*.
- **Group**: An series of Frames, each of which is delivered and decoded *in orde.
- **Frame**: A sized payload of bytes representing a single moment in time.

The application determines how to split data into tracks, groups, and frames.
MoqTransfork is responsible for the caching and delivery by utilizing rules encoded in headers.
This provides robust and generic one-to-many transmission, even for latency sensitive applications.

## Session

A Session consists of a connection between a QUIC client and server.

A session is established after the necessary QUIC, WebTransport, and MoqTransfork handshakes.
The MoqTransfork handshake consists of version and extension negotiation.

The intent is that sessions are chained together via relays.
A broadcaster could establish a session with an CDN ingest edge while the viewers establish separate sessions to CDN distribution edges.
A MoqTransfork session is hop-by-hop, but the application should be designed end-to-end.

## Track

A Track is a series of Groups identified by a unique path.
A MoqTransfork session is used to publish and subscribe to multiple, potentially unrelated, tracks.

A Track path consists of string parts used to route subscriptions to the correct publisher.
The application determines the structure and encoding of the path.
A Track path MUST be unique within a Session and SHOULD contain entropy to avoid collisions with other sessions, for example timestamps or random values.

A publisher can advertise available tracks via an ANNOUNCE message.
This allows a subscriber to dynamically discover available tracks based on a prefix.
For example: `["meeting", "1234", "alice"]` and `["meeting", "1234", "bob"]`
Alternatively, the application can discover tracks via an out-of-band mechanism.

Each subscription is scoped to a single Track.
A subscription will always start at a Group boundary, either the latest group or a specified sequence number.
A subscriber chooses the ordering and priority of each subscription, hinting to the publisher which Track should arrive first during congestion.
This is critical for a decent user experience during network degradation and the core reason why QUIC can offer real-time latency.

## Group

A Group is an ordered stream of Frames within a Track.

A group is served by a dedicated QUIC stream which may be closed on completion, reset by the publisher, or cancelled by the subscriber.
An active subscription involves delivering multiple potential concurrent groups.
The Frames within a Group will arrive reliably and in order thanks to the QUIC stream.
However, Groups within a Track can arrive in any order or not at all, and which the application should be prepared to handle.

A subscriber may FETCH a specific group starting at a given byte offset.
This is similar to a HTTP request and may be used to recover from partial failures among other things.

## Frame

A Frame is a payload of bytes within a Group.

A frame is used to represent a chunk of data with a known size.
A frame should represent a single moment in time and avoid any buffering that would increase latency.

## Liveliness

A media protocol can only be considered "live" if it can handle degraded network congestion.
MoqTransfork handles this by prioritizing the most important media while the remainder is starved.

The importance of each track/group/frame is signaled by the subscriber and the publisher will attempt to obey it.
Subscribers signal importance using Track Priority and Group Order.
Any data that is excessively starved may be dropped (by either endpoint) rather than block the live stream.

A publisher that serves multiple sessions, commonly a relay, should prioritize on a per-session basis.
Alice may want real-time latency with a preference for audio, while Bob may want reliable playback while audio is muted.
A relay MAY forward subscriber preferences upstream, but when there is a conflict (like the above example), the publisher's preference should be used as a tiebreaker.

# Workflow

This section outlines the flow of messages within a MoqTransfork session.
See the section for Messages section for the specific encoding.

## Connection

MoqTransfork runs on top of WebTransport.
WebTransport is a layer on top of QUIC and HTTP/3, required for web support.
The API is nearly identical to QUIC, however notably lacks stream IDs and has fewer available error codes.

How the WebTransport connection is established is out-of-scope for this draft.
For example, a service MAY use the WebTransport handshake to perform authentication via the URL.

## Termination

QUIC bidirectional streams have an independent send and receive direction.
Rather than deal with half-open states, MoqTransfork combines both sides.
If an endpoint closes the send direction of a stream, the peer MUST also close the send direction.

MoqTransfork contains many long-lived transactions, such as subscriptions and announcements.
These are terminated when the underlying QUIC stream is terminated.

To terminate a stream, an endpoint may:

- close the send direction (STREAM with FIN) to gracefully terminate (all messages are flushed).
- reset the send direction (RESET_STREAM) to immediately terminate.

After resetting the send direction, an endpoint MAY close the recv direction (STOP_SENDING).
However, it is ultimately the other peer's responsibility to close their send direction.

## Handshake

After a connection is established, the client opens a Session Stream and sends a SESSION_CLIENT message, to which the server replies with a SESSION_SERVER message.
The session is active until either endpoint closes or resets the Session Stream.

This session handshake is used to negotiate the MoqTransfork version and any extensions.
See the Extension section for more information.

# Streams

MoqTransfork involves many concurrent streams.

Each transactional action, such as a SUBSCRIBE, is sent over its own stream.
If the stream is closed, potentially with an error, the transaction is terminated.

## Bidirectional Streams

Bidirectional streams are primarily used for control streams.
There's a 1-byte STREAM_TYPE at the beginning of each stream.

| ID   | Stream     | Creator    |
|-----:|:-----------|:-----------|
| 0x0  | Session    | Client     |
| 0x1  | Announced  | Subscriber |
| 0x2  | Subscribe  | Subscriber |
| 0x3  | Fetch      | Subscriber |
| 0x4  | Info       | Subscriber |

### Session

The Session stream contains all messages that are session level.

The client MUST open a single Session Stream immediately
After establishing the QUIC/WebTransport session, the client opens a Session Stream.
There MUST be only one Session Stream per WebTransport session and its closure by either endpoint indicates the MoqTransfork session is closed.

The client sends a SESSION_CLIENT message indicating the supported versions and extensions.
If the server does not support any of the client's versions, it MUST close the stream with an error code and MAY close the connection.
Otherwise, the server replies with a SESSION_SERVER message to complete the handshake.

Afterwards, both endpoints SHOULD send SESSION_UPDATE messages, such as after a significant change in the session bitrate.

This draft's version is combined with the constant `0xff0bad00`.
For example, moq-transfork-draft-03 is identified as `0xff0bad03`.

### Announce

A subscriber can open a Announce Stream to discover tracks matching a prefix.
This is OPTIONAL and the application can determine track paths out-of-band.

The subscriber starts the stream with a ANNOUNCE_PLEASE message.
The publisher replies with ANNOUNCE messages for any matching tracks.
Each ANNOUNCE message contains one of the following statuses:

- `active`: a matching track is available.
- `ended`: a previously `active` track is no longer available.
- `live`: all active tracks have been sent.

Each track starts as `ended` can transition to and from the `active`.
The subscriber SHOULD close the stream if it receives a duplicate status, such as two `active` statuses in a row or an `ended` without `active`.

The `live` status applies to the entire stream instead of a single track.
It SHOULD be sent by the publisher after all `active` ANNOUNCE messages have been sent.
The subscriber SHOULD use this as a signal that it has caught up, for example a track may not exist.

Prefix matching is done on a part-by-part basis.
For example, a prefix of `["meeting"]` would match `["meeting", "1234"]` but not `["meeting-1234"]`.
The application is responsible for the encoding of the prefix, taking case to avoid any ambiguity such as multiple ways to encode the same value.

There MAY be multiple Announce Streams, potentially containing overlapping prefixes, that get their own copy of each ANNOUNCE.

#### Announce Parameters

- **NEW_SESSION_URI Parameter**:
The Parameter Type ID is 0x10.
This Parameter MAY only appear when the Annonce Status is `live` and MUST NOT appears when the Announce Status is any other value.

### Subscribe

A subscriber can open a Subscribe Stream to request a Track.

The subscriber MUST start a Info Stream with a SUBSCRIBE message followed by any number SUBSCRIBE_UPDATE messages.
The publisher MUST reply with an INFO message followed by any number of SUBSCRIBE_GAP messages.
A publisher MAY reset the stream at any point if it is unable to serve the subscription.

The publisher MUST transmit a complete Group Stream or a SUBSCRIBE_GAP message for each Group within the subscription range.
This means the publisher MUST transmit a SUBSCRIBE_GAP if a Group Stream is reset.
The subscriber SHOULD close the subscription when all GROUP and SUBSCRIBE_GAP messages have been received, and the publisher MAY close the subscription after all messages have been acknowledged.

### Fetch

A subscriber can open a Fetch Stream to receive a single Group at a specified offset.
This is primarily used to recover from an abrupt stream termination, causing the truncation of a Group.

The subscriber MUST start a Fetch Stream with a FETCH message followed by any number of FETCH_UPDATE messages.
The publisher MUST reply with the contents of a Group Stream, except starting at the specified offset *after* the GROUP message.
Note that this includes any FRAME messages.

The fetch is active until both endpoints close the stream, or either endpoint resets the stream.

### Info

A subscriber can open an Info Stream to request information about a track.
This is not often necessary as SUBSCRIBE will trigger an INFO reply.

The subscriber MUST start the stream with a INFO_PLEASE message.
The publisher MUST reply with an INFO message or reset the stream.
Both endpoints MUST close the stream afterwards.

## Unidirectional Streams

Unidirectional streams are used for data transmission.

| ID   | Stream | Creator   |
|-----:|:-------|-----------|
| 0x0  | Group  | Publisher |

### Group

A publisher creates Group Streams in response to a Subscribe Stream.

A Group Stream MUST start with a GROUP message and MAY be followed by any number of FRAME messages.
A Group MAY contain zero FRAME messages, potentially indicating a gap in the track.
A frame MAY contain an empty payload, potentially indicating a gap in the group.

Both the publisher and subscriber MAY reset the stream at any time.
When a Group stream is reset, the publisher MUST send a SUBSCRIBE_GAP message on the corresponding Subscribe stream.
A future version of this draft may utilize reliable reset instead.

# Encoding

This section covers the encoding of each message.

## Numbers

Numbers are encoded to the QUIC variable lenght integer.
In this document, a field with "(i)" indicates that it is a number.

## Bytes

Bytes are encoded to length-delimited.
In this document, a field with "(b)" indicates that it is a bytes array such as string.

## Parameters

Parameters are encoded to following format.
In this document, a field with "(Parameter)" indicates that it is a parameter.

~~~text
Parameters {
  Parameters Count (i)
  [
    Parameter Type ID (i)
    Parameter Payload (b)
  ]...
}
~~~

**Parameters Count**:
The number of the parameters

**Parameter Type ID**:
The ID of the parameter type.
The reserved parameter type ID is described below.

| ID   | Parameter Type       | Value Type |
|-----:|:---------------------|:----------:|
| 0x1  | Path                 | (b)        |
|------|----------------------|------------|
| 0x2  | Authorization Info   | (b)        |
|------|----------------------|------------|
| 0x3  | Delivery Timeout     | (i)        |
|------|----------------------|------------|
| 0x5  | New Session URI      | (b)        |
|------|----------------------|------------|

**Parameter Payload**:
The encoded value of the parameter.
Parameter paylod encoding rules:

- Byte array or string: encoded in (b).
- Number: encoded in (i).
- Boolean: encoded 1 in (i) for true, 0 in (i) for false.

## Messages

### STREAM_TYPE

All streams start with a short header indiciating the stream type.
## STREAM_TYPE
All streams start with a short header indicating the stream type.

~~~text
STREAM_TYPE {
  Stream Type (i)
}
~~~

The stream ID depends on if it's a bidirectional or unidirectional stream, as indicated in the Streams section.
A reciever MUST close the session if it receives an unknown stream type.

### SESSION_CLIENT

The client initiates the session by sending a SESSION_CLIENT message.

~~~text
SESSION_CLIENT Message {
  Supported Versions Count (i)
  Supported Version (i)
  Session Client Parameters (Parameters)
}
~~~

**Supported Versions Count**:
The number of the versions supported by the client.

**Supported Version**:
The version numbers supported by the client.

**Session Client Parameters**:
The optional parameters.

### SESSION_SERVER

The server responds with the selected version and any extensions.

~~~text
SESSION_SERVER Message {
  Selected Version (i)
  Session Server Parameters (Parameters)
}
~~~

### SESSION_UPDATE

~~~text
SESSION_UPDATE Message {
  Session Bitrate (i)
}
~~~

**Session Bitrate**:
The estimated bitrate of the QUIC connection in bits per second.
This SHOULD be sourced directly from the QUIC congestion controller.
A value of 0 indicates that the session will be terminated soon by the sender. The receiver MUST NOT send a new message to the session and the sender MUST ignore any new messages.

### ANNOUNCE_PLEASE

## ANNOUNCE_PLEASE
A subscriber sends an ANNOUNCE_PLEASE message to indicate it wants any corresponding ANNOUNCE messages.

~~~text
ANNOUNCE_PLEASE Message {
  Track Prefix (b),
}
~~~

**Track Prefix**:
Indicate interest for any tracks that start with this prefix.
This uses byte-for-byte matching on each prefix part.

The publisher MAY close the stream with an error code if the prefix is too expansive.
Otherwise, the publisher SHOULD respond with an ANNOUNCE message for any matching tracks.

### ANNOUNCE

A publisher sends an ANNOUNCE message to advertise a track.
Only the suffix is encoded on the wire, the full path is constructed by prepending the prefix from the corresponding ANNOUNCE_INTEREST.

~~~text
ANNOUNCE Message {
  Announce Status (i),
  [
    Track Suffix (b),
  ],
  Announce Parameters (Parameters),
}
~~~

**Announce Status**:
A flag indicating the announce status.

- `ended` (0): A path is no longer available.
- `active` (1): A path is now available.
- `live` (2): All active paths have been sent.

**Track Suffix**:
The track path suffix.
This field is not present for status `live`, which is not elegant I know.

**Announce Parameters**:
The optional parameters.

### SUBSCRIBE

SUBSCRIBE is sent by a subscriber to start a subscription.

~~~text
SUBSCRIBE Message {
  Subscribe ID (i),
  Track Path Parts (i),
  [ Track Path Part (b), ]
  Track Priority (i),
  Group Order (i),
  Group Min (i),
  Group Max (i),
}
~~~

**Subscribe ID**:
A unique idenfier chosen by the subscriber.
A Subscribe ID MUST NOT be reused within the same session, even if the prior subscription has been closed.

**Track Path Parts**:
The number of parts in the track path.

**Track Path**:
The unique path of the track.
The number of parts MUST be within the range of 1 to 32.
The entire path combined MUST be smaller than 1024 bytes.

**Track Priority**:
The transmission priority of the subscription relative to all other active subscriptions within the session.
The publisher SHOULD transmit *higher* values first during congestion.

**Group Order**:
The transmission order of the Groups within the subscription.
The publisher SHOULD transmit groups based on their sequence number in default (0), ascending (1), or descending (2) order.

**Group Min**:
The minimum group sequence number plus 1.
A value of 0 indicates the latest Group Sequence as determined by the publisher.

**Group Max**:
The maximum group sequence number plus 1.
A value of 0 indicates there is no maximum and the subscription continues indefinitely.

**Fetch Parameters**:
The optional parameters.

### SUBSCRIBE_UPDATE

A subscriber can modify a subscription with a SUBSCRIBE_UPDATE message.

~~~text
SUBSCRIBE_UPDATE Message {
  Track Priority (i)
  Group Order (i)
  Group Min (i)
  Group Max (i)
  Subscribe Update Parameters (Parameters)
}
~~~

**Track Priority**:
The new track priority; see SUBSCRIBE.
The publisher SHOULD use the new priority for any blocked streams.

**Group Order**:
The new group order; see SUBSCRIBE.
The publisher SHOULD use the new order for any blocked streams.

**Group Min**:
The new minimum group sequence, or 0 if there is no update.

**Group Max**:
The new maximum group sequence, or 0 if there is no update.

**Subscribe Update Parameters**:
The optional parameters.

If the Min and Max are updated, the publisher SHOULD reset any blocked streams that are outside the new range.

### SUBSCRIBE_GAP

A publisher transmits a SUBSCRIBE_GAP message when it is unable to serve a group for a SUBSCRIBE.

~~~text
SUBSCRIBE_GAP {
  Gap Min (i),
  Gap Max (i),
  Group Error Code (i),
}
~~~

**Gap Min**:
The minimum sequence number of the dropped range.
A value of 0 signifies that the publisher do not know where the gap starts.

**Gap Max**:
The maximum sequence number of the dropped range.
A value of 0 signifies that no more groups are not sent.

**Group Error Code**:
An error code indicated by the application.

- Internal Error (Code: 0x00): indicates the publisher cancels data streams in the range for some reason.
- Send Interrupted (Code: 0x01): indicates the subscriber cancels data streams in the range.
- Out of Range (Code: 0x02): indicates the publisher does not served any groups in the range.
- Cache Expired (Code: 0x03): indicates the cache for the groups has expired.
- Delivery Timeout (Code: 0x04): indicates that the publisher was not able to deliver groups within the delivery timeout.

### INFO

The INFO message contains the current information about a track.
This is sent in response to a SUBSCRIBE or INFO_PLEASE message.

~~~text
INFO Message {
  Track Priority (i)
  Group Latest (i)
  Group Order (i)
}
~~~

**Track Priority**:
The priority of this track within the session.
The publisher SHOULD transmit subscriptions with *higher* values first during congestion.

**Group Latest**:
The latest group as currently known by the publisher.
A relay without an active subscription SHOULD forward this request upstream

**Group Order**:
The publisher's intended order of the groups within the subscription: none (0), ascending (1), or descending (2).

### INFO_PLEASE

The INFO_PLEASE message is used to request an INFO response.

~~~text
INFO_PLEASE Message {
  Track Path Parts (i)
  [ Track Path Part (b) ]
}
~~~

### FETCH

A subscriber can request the transmission of a (partial) Group with a FETCH message.

~~~text
FETCH Message {
  Track Path Parts (i)
  [ Track Path Part (b) ]
  Track Priority (i)
  Group Sequence (i)
  Frame Sequence (i)
  Fetch Parameters (Parameter)
}
~~~

**Track Priority**:
The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session.
The publisher should transmit *higher* values first during congestion.

**Group Sequence**:
The sequence number of the group.

**Frame Sequence**:
The starting frame number.
All frames from this point forward are transmitted.

**Fetch Parameters**:
The optional parameters.

### FETCH_UPDATE

A subscriber can modify a FETCH request with a FETCH_UPDATE message.

~~~text
FETCH_UPDATE Message {
  Track Priority (i)
}
~~~

**Track Priority**:
The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session.
The publisher should transmit *higher* values first during congestion.

### GROUP

The GROUP message contains information about a Group, as well as a reference to the subscription being served.

~~~text
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
  Group Priority (i)
}
~~~

**Subscribe ID**:
The corresponding Subscribe ID.
This ID is used to distinguish between multiple subscriptions for the same track.

**Group Sequence**:
The sequence number of the group.

**Group Priority**:
The Priority of the group.

### FRAME

The FRAME message is a payload at a specific point of time.

~~~text
FRAME Message {
  Payload (b)
}
~~~

**Payload**:
An application specific payload.
A generic library or relay MUST NOT inspect or modify the contents unless otherwise negotiated.

# Appendix: Changelog

Notable changes between versions of this draft.

## This fork

### Changes from moq-transfork

- Added Group Priority to GROUP message
- Added New Session URI Paramerter to ANNOUNCE message
- Added DELIVERY_TIMEOUT parameter
- Renamed Group Start to Gap Min
- Changed Group Count to Gap Max
- Added datagram support
- Added native QUIC support

### Changes from moq-transport

- Removed the GOAWAY message.
- SESSION_UPDATE message is used to signal termination instead of the GOAWAY message.
- New Session URI is communicated as a parameter in ANNOUNCE message instead of a field in GOAWAY message.
- Relaxed the restrictions on updates for subscription.

## moq-transfork-04
- Removed Group Expires.

## moq-transfork-03

- Broadcast and Track have been merged.
- Tracks now have a variable length path instead of a (broadcast, name) tuple
- ANNOUNCE contains the suffix instead of the full path.
- ANNOUNCE contains a new status: active, ended, or live.
- GROUP_DROP has been renamed to SUBSCRIBE_GAP.
- FETCH is now at frame boundaries instead of byte offsets.
- The protocol is more polite: some messages have been renamed to *_PLEASE.
- Moved the use-cases appendix to a separate document.

## moq-transfork-02

- Document version numbers.
- Added ANNOUNCE_INTEREST to opt-into ANNOUNCE messages.
- Remove ROLE extension.

## moq-transfork-01

- Removed datagram support
- Removed native QUIC support
- Moved Expires from GROUP to SUBSCRIBE
- Added FETCH_UPDATE
- Added ROLE=Any
- Track Priority is now descending.

Datagram and native QUIC support may be re-added in a future draft.

## moq-transfork-00

Based on moq-transport-03.
The significant changes have been broken into sections.

- Bikeshedding
  - Renamed Track Namespace to Broadcast
  - Renamed Object to Frame
- Each unidirectional stream contains a single group.
  - Removed stream per track.
  - Removed stream per object.
- Subscriber chooses track priority and group order.
- Bidirectional control stream per ANNOUNCE/SUBSCRIBE/FETCH/INFO.
  - Removed ANNOUNCE_ERROR
  - Removed ANNOUNCE_DONE
  - Removed UNANNOUNCE
  - Removed SUBSCRIBE_OK
  - Removed SUBSCRIBE_ERROR
  - Removed SUBSCRIBE_DONE
  - Removed UNSUBSCRIBE
- Groups IDs are sequential.
- All sequences are explicitly delivered.
  - (unreliable) GROUP for each sequence.
  - Or (reliable) SUBSCRIBE_GAP when a stream is reset.
- Fetch Group via offset
- Added INFO_PLEASE and INFO
- Replaced SUBSCRIBE_OK with INFO


# Security Considerations

TODO Security

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
