---
title: "Dynamic Track Switching for MOQT relays"
category: info

docname: draft-wilaw-moq-dts4moq-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/dts4moq"
  latest: "https://wilaw.github.io/dts4moq/draft-wilaw-moq-dts4moq.html"

author:
  -
    fullname: Will Law
    organization: Akamai
    email: "2762250+wilaw@users.noreply.github.com"

  -
    name: Ian Swett
    organization: Google
    email: ianswett@google.com

  -
    name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

  -
    name: Mo Zanaty
    organization: Cisco
    email: mzanaty@cisco.com
 
   -
    name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com

   -
    name: Ali Begen
    organization: Ozyegin University
    email: ali.begen@ozyegin.edu.tr

   -
    name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr

   -
    name: Gwendal Simon
    organization: Synamedia
    email: gsimon@synamedia.com


normative:
  MOQT: I-D.draft-ietf-moq-transport-16

informative:

...

--- abstract

TODO Abstract


--- middle

# Introduction

This draft adds the capability of Dynamic Track Switching (DTS) to Media over QUIC Transport [MOQT].
Dynamic Track Switching allows a relay to dynamically switch which groups are forwarded from among
a set of subscriptions. One use-case enabled by DTS is Adaptive Bitrate Streaming (ABR), in which
time-aligned media tracks are switched at group boundaries based upon available throughput estimates.

DTS is enabled and disabled by the subscriber. The definition of the switching sets and the metadata
required to implement the switching rules are defined by either the subscriber or the original punblisher.

# Subscribe parameters
We introduce two new message parameters to enable Dynamic Track Switching.

## The SWITCHING-SET-ASSIGNMENT parameter
The SWITCHING-SET-ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE or REQUEST_UPDATE
message. This parameter assigns the accompanying subscription to a DTS switching set and specifies a waiting
time-limit. The parameter body is serialized as follows:

~~~
SWITCHING-SET-ASSIGNMENT {
  Switching set ID (vi64),
  [Throughput threshold (vi64),
  Selection time limit (vi64)]
}
~~~

* Switching set ID - an integer specifying a switching set. The value zero is reserved to indicate no switching set.
* Throughput threshold - the minimum throughput, expressed in integer kilobits per second, necessary to select
  this subscription. This value MUST be omitted if the Switching set ID is zero.
* Selection time limit - the maximum amount of time, in milliseconds, which a relay MUST wait after the arrival of the
  first candidate group in a switching group, before this subscription is disqualified as the preferred selection.
  This value MUST be omitted if the Switching set ID is zero.

## The DTS-ACTIVATION parameter.
The DTS-ACTIVATION parameter (Parameter Type 0x43) MAY appear in a SUBSCRIBE or REQUEST_UPDATE
message. This parameter enables or disables dynamic track switching for a specified switching set. The
parameter body is serialized as follows:

~~~
DTS-ACTIVATION {
  Switching set ID (vi64),
  State (1)
}
~~~

* Switching set ID - an integer referencing a previously defined switching set.
* State - a value of 1 activates dynanmic switching for the specified switching set and a value of 0 deactivates it.

# Workflow

## Client workflow

1. The client decides, through a catalog or other out-of-band mechanism, which of a set of tracks it wishes to enable
   for DTS.
2. The client selects an integer identifier to label this set. This identifier MUST be unique within the MOQT connection.
3. For each track, it issues a SUBSCRIPTION and appends a SWITCHING-SET-ASSIGNMENT parameter. Within that parameter, it
   communicates the set identifier, the throughput threshold and the time limit.
4. On the last SUBCRIPTION, the client also appends the DTS-ACTIVATION parameter and sets it value to 1. Dynamic track
   selection is now active for the switching set.

To disable dynamic track selection for a given switching set, the client sends a DTS-ACTIVATION parameter with a state
value of 0 on a REQUEST-UPDATE message. This action leaves all subscriptions in that switching set still active, but with
a Forward state of 0. To re-enable dynamic track selection for that set, the client sends a DTS-ACTIVATION parameter with
a state value of 1 on a REQUEST-UPDATE message.

To remove one track from a switching set, while leaving the other tracks active, the client issues a REQUEST_UPDATE message
with the request ID referencing the subscription it wishes to remove and a SWITCHING-SET-ASSIGNMENT parameter with a
Switching set ID of zero.

To add a new track to an existing switching set, the client issues a SUBSCRIPTION and appends a SWITCHING-SET-ASSIGNMENT
parameter, with the Switching set ID poiniting at the existing switching set.

## Relay workflow

1. Upon receiving a SWITCHING-SET-ASSIGNMENT parameter, the relay adds the subscription to the specified switching set,
   creating the switching set if it does not yet exist. The Forward state of the subscription is set to zero.
2. Upon receiving a DTS-ACTIVATION parameter with a state of 1, the relay beings active track selection.  Active track
   selection implies that the relay monitor the incoming new groups as well as maintain an estimate of the
   throughput available in the connection. This throughput estimate SHOULD be applicable over the maximum Group duration of the
   tracks being switched.
3. When the first Object 0 of new Group N of track T arrives at the relay, the relay selects the preferred track to forward from the
   switching set. The preferred track is the track with the highest throughput threshold smaller than or equal to the current
   throughput estimate. The relay sets the Forward state to 1 for this track and to 0 for all other tracks in the switching set.
   If no tracks in the switching set satisfy this condition, then all tracks are set to a Forward state of 0. No content will be
   delivered until the decision is re-evaulated at the next Gorup boundary.
5. If the track T happens to be the preferred track, then the relay forwards all Objects from that Group and no futher
   evaluations are required until Group N+1 arrives. If the track T is not the preferred track, then the relay caches the Group of
   track T and starts a selection timer.
6. If Group N of the preferred track arrives while the selection timer > selection time limit of that track, then the relay forwards
   the preferred track, the timer can be stopped and no further evaluations are necessary.
7. If Group N of the preferred track has not arrived by the time the selection timer reaches the selection time
   limit, then the relay selects a new preferred track by removing the previously preferred track from its candidate list and
   re-applying the logic of step 3. If Group N of the new preferred track has already arrived, then it is served from cache,
   irrespective of its Selection time limit. If Group N of the new preferred track has not yet arrived then the selection process repeats
   at step 6.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
