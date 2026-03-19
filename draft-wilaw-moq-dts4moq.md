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
 - moqt
 - dts
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/dts4moq"
  latest: "https://wilaw.github.io/dts4moq/draft-wilaw-moq-dts4moq.html"

author:
  - fullname: Will Law
    organization: Akamai
    email: "2762250+wilaw@users.noreply.github.com"
  - name: Ian Swett
    organization: Google
    email: ianswett@google.com
  - name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
  - name: Mo Zanaty
    organization: Cisco
    email: mzanaty@cisco.com
  - name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com
  - name: Ali Begen
    organization: Ozyegin University
    email: ali.begen@ozyegin.edu.tr
  - name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr
  - name: Gwendal Simon
    organization: Synamedia
    email: gsimon@synamedia.com


normative:
  MOQT: I-D.draft-ietf-moq-transport-16

informative:

...

--- abstract

Adds the capabaility of Dynamic Track Switching (DTS) to Media over QUIC Transport [MOQT].


--- middle

# Introduction

This draft adds the capability of Dynamic Track Switching (DTS) to Media over QUIC Transport [MOQT].
Dynamic Track Switching allows a relay to dynamically switch which groups are forwarded from among
a set of subscriptions. One use-case enabled by DTS is Adaptive Bitrate Streaming (ABR), in which
time-aligned media tracks are switched at group boundaries based upon available throughput estimates.

The definition of the switching sets and the metadata required to implement the switching rules are
defined by the subscriber. The subscriber also activates and deactivates switching on a given set.

# The SWITCHING-SET-ASSIGNMENT parameter
We introduce a new message parameter to enable Dynamic Track Switching.

The SWITCHING-SET-ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE or REQUEST_UPDATE
message. This parameter assigns the accompanying subscription to a DTS switching set, sets a throughput
threshold and throughput fraction and tells the relay whether it should begin actively switching the set.
The parameter body is serialized as follows:

~~~
SWITCHING-SET-ASSIGNMENT {
  Switching set ID (vi64),
  Throughput threshold (vi64),
  Set throughput fraction (vi64),
  Activate switching (1)
}
~~~

* Switching set ID - an integer specifying a switching set. A track MUST only be assigned to one switching set at
  a time.
* Throughput threshold - the minimum throughput, expressed in integer kilobits per second, necessary to select
  this subscription.
* Set throughput fraction - the fraction of the connection throughput that should be allocated to this switching set,
  expressed as a positive integer N, such that the set will be allocated 1/N of the estimated connection throughput.
* Activate switching  - 0 if the client will be adding more subscriptions to the set or 1 if the client is complete
  and the relay should activate switching.

# Content requirements

For tracks to participate in a dynamic switching set, they

* MUST be time aligned at group boundaries.
* MUST hold equivalent and independent versions of the same content, encoded at different bitrates. DTS is not a solution
  for switching between Scalable Video Coding (SVC) layers.

# Workflow

## Client workflow

1. The client decides, through a catalog or other out-of-band mechanism, which of a set of tracks it wishes to enable
   for DTS.
2. The client selects an integer identifier to label this set. This identifier MUST be unique within the MOQT connection.
3. For each track, it issues a SUBSCRIPTION and appends a SWITCHING-SET-ASSIGNMENT parameter. Within that parameter, it
   communicates the set identifier, the throughput threshold, the set's throughput fraction and the activation state. The
   activation state SHOULD be 0 for all but the last track to be assigned to the set. The set's throughput fraction SHOULD
   remain consistent for each set identifier. If it changes between subscriptions, then the last value supplied will be
   applied to the set.
5. On the last SUBSCRIPTION, the client sets the activation state flag to 1.  Dynamic track selection is now active for
   the switching set.

To add a new track to an existing switching set, the client issues a SUBSCRIPTION and appends a SWITCHING-SET-ASSIGNMENT
parameter, with the Switching set ID pointing at the existing switching set.

To remove one track from a switching set, the client unsubscribes from that track.

To disable dynamic track selection for a given switching set, but maintain the relay's definition of the set, the client
sends a REQUEST_UPDATE message for any one of the tracks contained within the switching set and attaches the
SWITCHING-SET-ASSIGNMENT parameter with an activate switching value of zero.

To resume switching on a given switching set, the client sends a REQUEST_UPDATE message for any one of the tracks contained
within the switching set and attach the SWITCHING-SET-ASSIGNMENT parameter with an activate switching value of 1.

To disable dynamic track selection for a given switching set, and remove the set from the relay,  the client unsubscribes
from all tracks in the switching set.

## Relay workflow

1. Upon receiving a SWITCHING-SET-ASSIGNMENT parameter, the relay adds the subscription to the specified switching set,
   creating the switching set if it does not yet exist. The Forward State of the subscription is set to zero. The
   Set Throughput Fraction is stored as a property of the switching set.
2. Upon receiving a SWITCHING-SET-ASSIGNMENT parameter with a Activate switching value of 1, the relay begins active track
   selection between all the tracks assigned to that switching set.  Active track selection implies that the relay monitors
   the incoming new groups as well as maintains an estimate of the throughput available in the connection. This throughput
   estimate SHOULD be applicable over the maximum Group duration of the tracks being switched.
3. As the first Object 0 of new Group N of track T within switching set S arrives at the relay, the relay calculates the preferred
   track to forward from the switching set. The preferred track is the track with the highest throughput threshold smaller than
   or equal to the current throughput estimate divided by the throughput fraction. If the preferred track is T, then the relay sets
   the Forward state to 1 for this track and to 0 for all other tracks in the switching set. If no tracks in the switching set
   satisfy this condition, then all tracks are set to a Forward state of 0. No content will be delivered until the decision is
   re-evaulated at the next Group boundary.
4. The prior step repeats at each Group boundary as long as the switching set contains at least one track.
5. The throughput threshold of any track, as well as the throughput fraction of the set and the activate switching state MAY be
   updated at any time via a REQUEST_UPDATE message associated with any of the constituent tracks within the switching set.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
