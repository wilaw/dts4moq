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

TODO Abstract


--- middle

# Introduction

This draft adds the capability of Dynamic Track Switching (DTS) to Media over QUIC Transport [MOQT].
Dynamic Track Switching allows a relay to dynamically switch which groups are forwarded from among
a set of subscriptions. One use-case enabled by DTS is Adaptive Bitrate Streaming (ABR), in which
time-aligned media tracks are switched at group boundaries based upon available throughput estimates.

DTS is enabled and disabled by the subscriber. The definition of the switching sets and the metadata
required to implement the switching rules are defined by either the subscriber or the original punblisher.

# Requirements

This section describes the requirements that Dynamic Track Switching places on original publishers,
end subscribers, and relays. These requirements are derived from the use cases described in
{{usecase-appendix}}.

The use cases cover a range of real-world applications:

- **Single switching set:** A single media source with multiple quality renditions, such as
  Adaptive Bitrate Streaming ({{usecase-abr}}).

- **Multiple switching sets:** Several concurrent media sources each with quality renditions,
  requiring bandwidth allocation based on relative priorities—video conferencing
  ({{usecase-videoconf}}), screen sharing ({{usecase-screenshare}}), and VR/AR streaming
  ({{usecase-vr}}).

- **Switching set(s) with guaranteed streams:** Adaptive media combined with fixed-bandwidth
  streams requiring guaranteed delivery—cloud gaming ({{usecase-gaming}}), live sports
  ({{usecase-sports}}), and teleoperation ({{usecase-teleop}}).

## Original Publisher Requirements

Original publishers are responsible for producing and advertising media tracks that can be
dynamically switched by relays.

- MUST publish multiple time-aligned renditions of the same content at different quality levels
  (see {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}},
  {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- MUST ensure groups across renditions are temporally aligned to enable seamless switching at
  group boundaries (see {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-vr}}, {{usecase-sports}})

- SHOULD advertise throughput requirements for each rendition to enable bandwidth-based selection
  (see {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}},
  {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- SHOULD indicate which tracks belong to the same switching set or alternate group
  (see {{usecase-abr}}, {{usecase-videoconf}})

- SHOULD indicate content type characteristics and relative priorities when publishing multiple
  content types (see {{usecase-screenshare}}, {{usecase-gaming}}, {{usecase-teleop}})

- SHOULD publish guaranteed or fixed-bandwidth streams (such as HUD overlays, stats, or telemetry)
  as separate tracks when bandwidth guarantees are needed
  (see {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- SHOULD minimize encoding latency for latency-critical streams such as control cameras or
  interactive game video (see {{usecase-gaming}}, {{usecase-teleop}})

- MUST ensure all streams share a common time reference when temporal synchronization across
  multiple streams is required (see {{usecase-sports}}, {{usecase-teleop}})

- SHOULD indicate spatial relationships or layer hierarchy when applicable
  (see {{usecase-vr}})

## End Subscriber Requirements

End subscribers are responsible for establishing subscriptions and communicating switching
preferences to relays.

- MUST subscribe to all desired renditions within a switching set and indicate which subscriptions
  form a switching set for relay coordination (see {{usecase-abr}})

- MUST be prepared to decode any rendition within the switching set (see {{usecase-abr}})

- MUST subscribe to multiple switching sets simultaneously when receiving multiple independent
  streams (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}}, {{usecase-gaming}},
  {{usecase-sports}}, {{usecase-teleop}})

- MUST indicate relative importance, weight, or priority for each switching set to guide relay
  bandwidth allocation (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}},
  {{usecase-sports}}, {{usecase-teleop}})

- MAY dynamically adjust priorities based on changing conditions such as active speaker changes,
  gaze direction updates, or user interaction
  (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}}, {{usecase-sports}})

- SHOULD specify total bandwidth budget or constraints when applicable (see {{usecase-videoconf}})

- MAY specify minimum acceptable quality thresholds for critical content
  (see {{usecase-screenshare}}, {{usecase-vr}}, {{usecase-sports}})

- SHOULD specify latency requirements for latency-critical streams
  (see {{usecase-gaming}}, {{usecase-teleop}})

## Relay Requirements

Relays are responsible for making dynamic track selection decisions and forwarding the
appropriate groups to downstream subscribers.

- MUST track available bandwidth to each downstream subscriber (see {{usecase-abr}})

- MUST select exactly one rendition per switching set to forward at any given time
  (see {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}},
  {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- MUST switch renditions at group boundaries to maintain decodability
  (see {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-vr}}, {{usecase-gaming}},
  {{usecase-sports}}, {{usecase-teleop}})

- MUST allocate forwarding capacity across multiple switching sets simultaneously when serving
  subscribers with multiple concurrent streams
  (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}}, {{usecase-gaming}},
  {{usecase-sports}}, {{usecase-teleop}})

- MUST respect subscriber-indicated weights and priorities when selecting groups to forward
  (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-sports}})

- SHOULD implement degradation based on relative weights when total capacity is insufficient,
  reducing quality of lower-priority content before higher-priority content
  (see {{usecase-videoconf}}, {{usecase-screenshare}})

- MUST respond rapidly to updated priority weights from subscribers
  (see {{usecase-videoconf}}, {{usecase-vr}}, {{usecase-sports}})

- MUST prioritize forwarding groups for guaranteed streams before performing adaptive allocation
  for other streams (see {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- MUST prioritize latency for latency-critical streams such as control cameras or interactive
  game video (see {{usecase-gaming}}, {{usecase-teleop}})

- MUST maintain temporal synchronization when forwarding multiple related streams
  (see {{usecase-sports}}, {{usecase-teleop}})

- SHOULD minimize switching latency when bandwidth conditions change (see {{usecase-abr}})

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

# Use Cases {#usecase-appendix}

This appendix describes several use cases that motivate Dynamic Track Switching,
organized by the complexity of switching set configurations.

## Single Switching Set

These use cases involve a single media source with multiple quality renditions.

### Adaptive Bitrate Streaming (ABR) {#usecase-abr}

In adaptive bitrate streaming, a single media source (e.g., a live video stream) is encoded
at multiple quality levels (renditions) with different bitrates and resolutions. The goal is
to deliver the highest quality rendition that the network path can sustain at any given moment.
When bandwidth decreases, the system should switch to a lower quality rendition to avoid
rebuffering. When bandwidth increases, it should switch to a higher quality rendition to
improve viewer experience.

The original publisher encodes the content into multiple renditions (e.g., 1080p at 5 Mbps,
720p at 2 Mbps, 480p at 800 kbps) and publishes each as a separate track with temporal
alignment at group boundaries. The publisher/subscriber advertises the throughput requirements and
indicates that these tracks form a switching set. The end subscriber subscribes to all
renditions in the switching set and receives whichever rendition the relay selects. The
relay monitors downstream bandwidth, selects the highest quality rendition that fits within
the available capacity, and switches to a different rendition at group boundaries when
bandwidth conditions change.

~~~
                         ┌────────────────────────────────────────────┐
                         │           Original Publisher               │
                         │                                            │
                         │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
                         │  │ 1080p   │  │  720p   │  │  480p   │     │
                         │  │ 5 Mbps  │  │ 2 Mbps  │  │ 800kbps │     │
                         │  └────┬────┘  └────┬────┘  └────┬────┘     │
                         │       │            │            │          │
                         └───────┼────────────┼────────────┼──────────┘
                                 │            │            │
                                 ▼            ▼            ▼
                         ┌─────────────────────────────────────────────┐
                         │                  Relay                      │
                         │                                             │
                         │   Receives all renditions, selects one      │
                         │   based on throughput thresholds and        │
                         │   downstream bandwidth                      │
                         └────────────────────┬────────────────────────┘
                                              │
                                              │  Selected rendition
                                              │  (e.g., 720p @ 2 Mbps)
                                              ▼
                         ┌─────────────────────────────────────────────┐
                         │              End Subscriber                 │
                         │                                             │
                         │   Receives single stream, quality varies    │
                         │   based on available bandwidth              │
                         │                                             │
                         └─────────────────────────────────────────────┘
~~~

## Multiple Switching Sets

These use cases involve several concurrent media sources, each with quality renditions,
requiring bandwidth allocation based on relative priorities.

### Video Conferencing Grid Layout {#usecase-videoconf}

In a video conference with multiple participants, each participant's video may be displayed
in a grid layout. When many participants are present, not all videos can be displayed at
full resolution due to screen real estate and bandwidth constraints. The system needs to
deliver multiple participant streams simultaneously, potentially at different quality levels
based on their importance (e.g., active speaker at high quality, other participants at
lower quality).

Each original publisher (participant) encodes their video at multiple quality levels and
publishes these as a switching set. The end subscriber subscribes to multiple switching
sets (one per participant) and assigns relative weights to indicate importance—for example,
giving the active speaker higher weight than other participants. When the active speaker
changes, the subscriber adjust relative weights. The relay allocates
its forwarding capacity across all switching sets according to the subscriber-indicated
weights, selecting appropriate quality levels for each participant stream to fit within
the total available bandwidth.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Original Publishers                                 │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Participant │  │ Participant │  │ Participant │  │ Participant │   ...    │
│  │     A       │  │     B       │  │     C       │  │     D       │          │
│  │ hi/med/lo   │  │ hi/med/lo   │  │ hi/med/lo   │  │ hi/med/lo   │          │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │
│         │                │                │                │                 │
└─────────┼────────────────┼────────────────┼────────────────┼─────────────────┘
          │                │                │                │
          ▼                ▼                ▼                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Allocates bandwidth across multiple switching sets (participants)          │
│   Selects quality per participant based on:                                  │
│     - Subscriber-indicated weights per switching set                         │
│     - Total available bandwidth                                              │
│     - Throughput thresholds per rendition                                    │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Multiple streams at varying qualities
                                      │  (e.g., A@hi, B@med, C@lo, D@lo)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                             End Subscriber                                   │
│                                                                              │
│   ┌───────────────────────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│   │                           │  │         │  │         │  │         │       │
│   │     Participant A         │  │  Part B │  │  Part C │  │  Part D │       │
│   │     (high quality)        │  │  (med)  │  │  (low)  │  │  (low)  │       │
│   │                           │  │         │  │         │  │         │       │
│   └───────────────────────────┘  └─────────┘  └─────────┘  └─────────┘       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

### Screen Sharing with Video {#usecase-screenshare}

A participant shares their screen while also transmitting camera video. The screen content
may have different characteristics than camera video (e.g., higher resolution for text
readability, lower frame rate acceptable). The system needs to prioritize bandwidth between
screen sharing and camera video based on content type and subscriber preferences.

The original publisher encodes both screen share and camera video at multiple quality levels,
publishing each content type as a separate switching set. The publisher indicates that screen
share has higher priority than camera video. The end subscriber subscribes to both switching
sets and may specify a minimum acceptable quality (throughput) for the screen share to
ensure text remains readable. The relay manages bandwidth allocation between the two content types,
degrading camera video quality before reducing screen share quality when bandwidth becomes constrained.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                            Original Publisher                                │
│                                                                              │
│   ┌─────────────────────────────┐      ┌─────────────────────────────┐       │
│   │      Screen Share           │      │      Camera Video           │       │
│   │  ┌───────┐ ┌───────┐        │      │  ┌───────┐ ┌───────┐        │       │
│   │  │1080p  │ │ 720p  │        │      │  │ 720p  │ │ 360p  │        │       │
│   │  │2 Mbps │ │800kbps│        │      │  │1.5Mbps│ │400kbps│        │       │
│   │  └───┬───┘ └───┬───┘        │      │  └───┬───┘ └───┬───┘        │       │
│   │      │         │            │      │      │         │            │       │
│   └──────┼─────────┼────────────┘      └──────┼─────────┼────────────┘       │
│          │         │                          │         │                    │
└──────────┼─────────┼──────────────────────────┼─────────┼────────────────────┘
           │         │                          │         │
           ▼         ▼                          ▼         ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Manages two switching sets                                                 │
│   Allocates bandwidth based on subscriber-indicated weights                  │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Screen@1080p + Camera@360p
                                      │  (prioritizing screen readability)
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                             End Subscriber                                   │
│                                                                              │
│   ┌───────────────────────────────────────────┐  ┌────────────────────┐      │
│   │                                           │  │                    │      │
│   │           Screen Share                    │  │   Camera (small)   │      │
│   │           (high quality for text)         │  │   (lower quality)  │      │
│   │                                           │  │                    │      │
│   └───────────────────────────────────────────┘  └────────────────────┘      │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

### VR/AR Streaming {#usecase-vr}

Virtual and augmented reality applications require streaming high-resolution immersive content
while adapting to available bandwidth. Two key scenarios benefit from DTS: foveated rendering
where quality varies based on gaze direction, and multi-layer environments where different
scene elements have different quality requirements.

In foveated rendering, a 360-degree video is divided into tiles. The tile where the user is
currently looking (determined by eye tracking) receives highest quality, while peripheral
tiles receive lower quality. As the user's gaze shifts, bandwidth allocation must dynamically
shift between tiles.

The original publisher encodes each tile at multiple quality levels and publishes them as
separate switching sets, indicating spatial relationships between tiles. The end subscriber
subscribes to all tiles within the field of view and as gaze direction changes, subscriber
assigns higher weight to the gaze tile and lower weights to
peripheral tiles. The relay responds rapidly to these updates, reallocating bandwidth to
deliver high quality for the gaze tile while maintaining lower quality for surrounding tiles.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                          VR Headset (Publisher)                              │
│                                                                              │
│   360° Video Tiles (each with quality variants)                              │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                │
│   │ Tile 1  │ │ Tile 2  │ │ Tile 3  │ │ Tile 4  │ │ Tile 5  │  ...           │
│   │ hi/lo   │ │ hi/lo   │ │ hi/lo   │ │ hi/lo   │ │ hi/lo   │                │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                │
│        │           │     [GAZE]│           │           │                     │
└────────┼───────────┼───────────┼───────────┼───────────┼─────────────────────┘
         │           │           │           │           │
         ▼           ▼           ▼           ▼           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Receives gaze direction updates (weights) from subscriber                  │
│   Allocates bandwidth: high quality to gaze tile, lower to periphery         │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Tile3@hi, others@lo
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                             End Subscriber                                   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐        │
│   │                      360° Rendered View                         │        │
│   │  ┌──────┐ ┌──────┐ ┌────────────────┐ ┌──────┐ ┌──────┐         │        │
│   │  │ lo   │ │ lo   │ │      hi        │ │ lo   │ │ lo   │         │        │
│   │  │      │ │      │ │   (gaze area)  │ │      │ │      │         │        │
│   │  └──────┘ └──────┘ └────────────────┘ └──────┘ └──────┘         │        │
│   └─────────────────────────────────────────────────────────────────┘        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

## Switching Set(s) with Guaranteed Streams

These use cases combine adaptive media with fixed-bandwidth streams that require
guaranteed delivery.

### Cloud Gaming {#usecase-gaming}

Cloud gaming services stream rendered game video from servers to players. The video stream
must adapt to network conditions while balancing resolution, frame rate, and latency based
on game type and player preferences. Different game genres have different requirements:
fast-paced action games prioritize frame rate and low latency, while strategy games may
prioritize resolution.

Additionally, different regions of the game screen may have different importance: the HUD
(heads-up display) with critical game information may need guaranteed quality, while the
main game world adapts to remaining bandwidth.

The original publisher (game server) encodes the game world video at multiple quality levels
and publishes the HUD as a separate fixed-bandwidth track with high priority. The publisher
minimizes encoding latency to maintain gameplay responsiveness. The end subscriber subscribes
to both the game video switching set and the HUD track, indicating that the HUD requires
guaranteed bandwidth. The relay reserves bandwidth for the HUD first, then selects the
appropriate game video quality from the remaining capacity, prioritizing low latency
throughout the forwarding path.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Game Server (Publisher)                              │
│                                                                              │
│   ┌─────────────────────────────────┐  ┌─────────────────────────────┐       │
│   │       Game World Video          │  │      HUD/Overlay            │       │
│   │  ┌───────┐ ┌───────┐ ┌───────┐  │  │  ┌───────┐                  │       │
│   │  │4K/60  │ │1080/60│ │720/60 │  │  │  │ Fixed │                  │       │
│   │  │25Mbps │ │8 Mbps │ │3 Mbps │  │  │  │200kbps│                  │       │
│   │  └───┬───┘ └───┬───┘ └───┬───┘  │  │  └───┬───┘                  │       │
│   └──────┼─────────┼─────────┼──────┘  └──────┼──────────────────────┘       │
│          │         │         │                │                              │
└──────────┼─────────┼─────────┼─────────────── ┼──────────────────────────────┘
           │         │         │                │
           ▼         ▼         ▼                ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Reserves HUD bandwidth first (subscriber-indicated guaranteed stream)      │
│   Selects game quality from remainder based on throughput thresholds         │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Game@1080/60 + HUD@fixed
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                              Player (Subscriber)                             │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────┐        │
│   │  ┌─────────────────────────────────────────────────────────┐    │        │
│   │  │                                                         │    │        │
│   │  │                    Game World                           │    │        │
│   │  │                  (adaptive quality)                     │    │        │
│   │  │                                                         │    │        │
│   │  └─────────────────────────────────────────────────────────┘    │        │
│   │  ┌─────────────────┐                      ┌─────────────────┐   │        │
│   │  │ Health: XXXX--- │                      │  Ammo: 30/120   │   │        │
│   │  └─────────────────┘    HUD (guaranteed)  └─────────────────┘   │        │
│   └─────────────────────────────────────────────────────────────────┘        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

### Live Sports Multi-View {#usecase-sports}

Live sports broadcasts offer multiple camera angles: main game camera, sideline cameras,
aerial views, and isolated player cameras. Viewers may want to watch multiple angles
simultaneously, with the ability to prioritize different views. A stats overlay stream
provides real-time game information. Bandwidth must be allocated across these streams
based on viewer preferences that may change during the event (e.g., switching focus to
replay angle).

The original publisher (broadcast origin) encodes each camera angle at multiple quality
levels and publishes a stats overlay as a separate guaranteed stream. All streams share
a common time reference for synchronization. The end subscriber subscribes to desired
camera angles and the stats overlay, assigning weights to indicate which views are most
important. During highlights or replays, the subscriber dynamically adjusts priority
weights to shift focus to the relevant camera. The relay allocates bandwidth according to
subscriber weights, maintains temporal sync across all forwarded streams, and responds
promptly to priority changes.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Broadcast Origin (Publisher)                         │
│                                                                              │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│  │  Main Camera  │ │   Sideline    │ │    Aerial     │ │  Stats/Score  │     │
│  │  hi/med/lo    │ │  hi/med/lo    │ │  hi/med/lo    │ │   (fixed)     │     │
│  └───────┬───────┘ └───────┬───────┘ └───────┬───────┘ └───────┬───────┘     │
│          │                 │                 │                 │             │
└──────────┼─────────────────┼─────────────────┼─────────────────┼─────────────┘
           │                 │                 │                 │
           ▼                 ▼                 ▼                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Reserves stats bandwidth (subscriber-indicated guaranteed stream)          │
│   Allocates remaining bandwidth based on subscriber-indicated weights        │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Main@hi, Sideline@med, Aerial@lo, Stats
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                             Viewer (Subscriber)                              │
│                                                                              │
│   ┌────────────────────────────────────────┐  ┌──────────────────────┐       │
│   │                                        │  │    Sideline View     │       │
│   │           Main Camera View             │  │    (medium quality)  │       │
│   │           (high quality)               │  ├──────────────────────┤       │
│   │                                        │  │    Aerial View       │       │
│   │                                        │  │    (low quality)     │       │
│   └────────────────────────────────────────┘  └──────────────────────┘       │
│   ┌─────────────────────────────────────────────────────────────────┐        │
│   │  SCORE: Home 2 - Away 1  |  Time: 73:24  |  Possession: 58%     │        │
│   └─────────────────────────────────────────────────────────────────┘        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

### Teleoperation and Robotics {#usecase-teleop}

Remote operation of robots, drones, or industrial equipment requires streaming multiple
video feeds with different importance levels. The primary control camera (showing the
manipulation task) requires highest quality and lowest latency. Secondary cameras
providing situational awareness can accept lower quality. Sensor telemetry streams
compete for bandwidth with video feeds.

The original publisher (robot or drone) encodes the primary control camera at multiple
quality levels with minimal encoding latency, encodes situational cameras at multiple
levels, and publishes sensor telemetry as a separate guaranteed stream. All streams
share a common time reference. The end subscriber subscribes to the primary camera
with highest priority and specifies latency requirements, subscribes to telemetry with
guaranteed bandwidth, and subscribes to situational cameras with lower priority. The
relay prioritizes latency for the primary camera, reserves bandwidth for telemetry,
and allocates remaining capacity to situational cameras.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Robot (Publisher)                                  │
│                                                                              │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐     │
│  │ Primary Cam   │ │  Left Cam     │ │  Right Cam    │ │  Telemetry    │     │
│  │ (manipulation)│ │ (situational) │ │ (situational) │ │  (sensors)    │     │
│  │  hi/med/lo    │ │  hi/lo        │ │  hi/lo        │ │  (fixed)      │     │
│  └───────┬───────┘ └───────┬───────┘ └───────┬───────┘ └───────┬───────┘     │
│          │                 │                 │                 │             │
└──────────┼─────────────────┼─────────────────┼─────────────────┼─────────────┘
           │                 │                 │                 │
           ▼                 ▼                 ▼                 ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                                  Relay                                       │
│                                                                              │
│   Allocates bandwidth based on subscriber-indicated weights                  │
│   Latency-critical: minimize delay for primary control feed                  │
│                                                                              │
└─────────────────────────────────────┬────────────────────────────────────────┘
                                      │
                                      │  Primary@hi, Left@lo, Right@lo, Telemetry
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         Operator Console (Subscriber)                        │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────┐         │
│   │                                                                │         │
│   │              Primary Camera (high quality, low latency)        │         │
│   │                                                                │         │
│   └────────────────────────────────────────────────────────────────┘         │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐           │
│   │   Left Camera    │  │   Right Camera   │  │   Telemetry      │           │
│   │   (low quality)  │  │   (low quality)  │  │ Temp: 45°C       │           │
│   │                  │  │                  │  │ Battery: 73%     │           │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
