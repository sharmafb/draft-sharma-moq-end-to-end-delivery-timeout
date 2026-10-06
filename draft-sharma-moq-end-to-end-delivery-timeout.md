---
title: "End-to-End Delivery Timeouts for MOQT"
abbrev: "moq-e2e-delivery-timeout"
category: std

docname: draft-sharma-moq-end-to-end-delivery-timeout-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - media over quic
 - delivery timeout
 - timestamp

venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "sharmafb/draft-sharma-moq-end-to-end-delivery-timeout"
  latest: "https://sharmafb.github.io/draft-sharma-moq-end-to-end-delivery-timeout/draft-sharma-moq-end-to-end-delivery-timeout.html"

smart_quotes: no

author:
 -
    ins: A. Sharma
    fullname: Aman Sharma
    organization: Meta
    email: amsharma@meta.com

normative:
  MOQT: I-D.ietf-moq-transport
  TIMESTAMP: I-D.frindell-moq-timestamp

--- abstract

This document defines an end-to-end Object delivery timeout for Media over
QUIC Transport (MOQT).  It uses Object timestamps to include delay accumulated
across a chain of relays instead of restarting the timeout at each hop.

--- middle

# Introduction

The MOQT OBJECT_DELIVERY_TIMEOUT is measured from the time an Object reaches
the current publisher.  Consequently, an Object can receive a new timeout
budget at every relay.

This document defines a subscription timeout measured against the Object
timeline from {{TIMESTAMP}}.  The first Object establishes a local reference.
Later Objects that fall more than the requested timeout behind that timeline
are no longer forwarded.  This includes upstream delay accumulated after the
reference is established and requires no synchronized clocks.

# Conventions

{::boilerplate bcp14-tagged}

This document uses the terms Object, Publisher, Relay, Subscription, and
Message Parameter as defined in {{MOQT}}.  It uses Object Timestamp and
Timescale as defined in {{TIMESTAMP}}.

This extension applies to version 22 of {{MOQT}}, identified by the `moqt-22`
protocol identifier.  Its use with other versions is undefined.

# Negotiation

An endpoint supports this extension by including the zero-length
END_TO_END_DELIVERY_TIMEOUT Setup Option in SETUP.  The extension is negotiated
when both endpoints include the option.  An endpoint MUST NOT send the Message
Parameter defined below unless the extension was negotiated.

# End-to-End Object Delivery Timeout

The END_TO_END_OBJECT_DELIVERY_TIMEOUT Message Parameter is a variable-length
integer containing a timeout in milliseconds.  It MAY appear in SUBSCRIBE,
SUBSCRIBE_TRACKS, PUBLISH, or a REQUEST_UPDATE that updates a Subscription.  A
value of 0 disables the timeout.

When included in SUBSCRIBE_TRACKS, the parameter is the initial value for each
resulting Subscription and is copied into PUBLISH as specified in {{MOQT}}.  A
publisher MUST send PUBLISH_SKIPPED instead of PUBLISH for a Track that cannot
provide the timestamps required by this extension.

A publisher MUST NOT establish a Subscription with a non-zero value unless it
can compute an Object Timestamp for every Normal Object it might forward.  A
Normal Object without a computable timestamp while the timeout is active makes
the Track malformed.

For each Subscription with a non-zero timeout, the publisher maintains a
reference timestamp and a reference time.  Immediately before forwarding the
first Normal Object after the timeout is enabled, it records the Object's
timestamp as the reference timestamp and its local monotonic time as the
reference time.  That Object is not expired by this extension.

For a later Object with timestamp `T`, the publisher computes its deadline as:

~~~
deadline = reference_time
         + (T - reference_timestamp) / timescale
         + timeout
~~~

Conversions between ticks and time MUST NOT make the deadline earlier.  If the
publisher's current time is later than the deadline, it MUST apply the
expiration behavior specified for OBJECT_DELIVERY_TIMEOUT in {{MOQT}}.  This
comparison applies across all Subgroups in the Subscription.

Changing one non-zero timeout to another preserves the reference.  Disabling
the timeout clears the reference; enabling it again establishes a new reference
with the next Normal Object.

For example, if the reference Object has timestamp 10 seconds and a later
Object has timestamp 12 seconds, a 500 millisecond timeout expires the later
Object 2.5 seconds after the reference Object was forwarded.

This timeout is independent of OBJECT_DELIVERY_TIMEOUT and
SUBGROUP_DELIVERY_TIMEOUT; the first applicable timeout to expire determines
the result.  As with all Message Parameters, a Relay MUST NOT copy this
parameter to an upstream request.  The local timestamp comparison already
includes delay accumulated upstream.

# Limitations

This extension bounds lateness relative to the first forwarded Object.  It
does not bound startup latency, delay Objects that arrive early, or guarantee
that an Object reaches the subscriber after it has been handed to the
underlying transport.  Accuracy depends on the relative stability of the
publisher's monotonic clock and the timestamp clock.

# IANA Considerations

IANA is requested to register the following entry in the "MOQ Setup Options"
registry:

| Type | Name | Specification |
|-----:|:-----|:--------------|
| TBD1 (odd) | END_TO_END_DELIVERY_TIMEOUT | This document |

TBD1 uses the length-prefixed encoding and has a zero-length value.

IANA is also requested to register the following entry in the "MOQ Message
Parameters" registry:

| Parameter Type | Parameter Name | Specification |
|---------------:|:---------------|:--------------|
| TBD2 | END_TO_END_OBJECT_DELIVERY_TIMEOUT | This document |

# Security Considerations

Incorrect timestamps can cause useful Objects to be discarded or stale Objects
to be retained.  Publishers and subscribers SHOULD apply reasonable bounds to
timestamps and timeout values.  When timestamps need protection from an
untrusted Relay, the timestamp Properties SHOULD be carried in Immutable
Properties as described in {{MOQT}} and {{TIMESTAMP}}.

--- back

# Acknowledgments
{:numbered="false"}

The relative-time algorithm in this document is based on a proposal by Martin
Duke.  The timestamp representation was defined by Alan Frindell and Ian Swett.
