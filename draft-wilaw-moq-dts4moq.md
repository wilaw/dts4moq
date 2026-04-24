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
    email: "wilaw@akamai.com"
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

This document defines Dynamic Track Switching (DTS) for Media over QUIC Transport (MOQT).
DTS enables relays to dynamically select which track to forward from a switching set—a
collection of time-aligned tracks representing the same content at different throughput levels.
The relay selects exactly one track per switching set at each group boundary based on
available downstream bandwidth. Subscribers use the SWITCHING-SET-ASSIGNMENT parameter to
group subscriptions into switching sets and specify bandwidth allocation via fraction
(relative weight) and rank (degradation priority). Fraction determines target bandwidth
allocation; rank determines which sets degrade first when bandwidth is constrained.


--- middle

# Introduction

This document defines Dynamic Track Switching (DTS) for Media over QUIC Transport [MOQT].
DTS enables relays to dynamically select which track to forward from a switching set based
on available downstream bandwidth.

DTS addresses a range of real-time media applications where bandwidth-adaptive delivery
improves user experience. These include adaptive bitrate streaming for live video, video
conferencing with multiple participants at varying throughput levels, VR/AR streaming with
gaze-based quality allocation, and hybrid scenarios combining adaptive streams with
fixed-bandwidth overlays such as cloud gaming HUDs or sports statistics. These use cases
(detailed in {{usecase-appendix}}) drive the requirements for originla publishers, end subscribers, and relays specified in this document.

A switching set is a collection of MoQ tracks representing the same content encoded at
different throughput levels, typically from a single source. Tracks within a switching set are time-aligned at group boundaries, allowing the relay to switch between tracks without disrupting
errors. The relay selects exactly one track per switching set to forward at any given time.

Subscribers can create switching sets through two methods:

- **Individual SUBSCRIBE**: Subscriber sends SUBSCRIBE for each track with
  SWITCHING-SET-ASSIGNMENT parameter to group them into switching sets.

- **SUBSCRIBE_NAMESPACE with TRACK_FILTER**: Subscriber uses namespace subscription with
  a filter to select top-N tracks ranked by a specified property (e.g., VIEWCOUNT for
  selecting the most popular streams). The relay forwards matching tracks and the
  subscriber assigns them to switching sets via PUBLISH_OK.

Both methods result in the same relay behavior: the relay processes switching sets,
allocates bandwidth based on fraction and rank, and selects the best track per set
at each group boundary. Fraction determines target bandwidth allocation; rank determines
degradation priority when bandwidth is constrained, lower-ranked sets degrade first while
higher-ranked sets are protected.

# Requirements

This section describes the requirements that Dynamic Track Switching places on original
publishers, end subscribers, and relays. These requirements are derived from the use cases
described in {{usecase-appendix}}.

## Terminology

A **switching set** is a collection of one or more tracks that represent the same content at
different throughput levels. All tracks within a switching set MUST be time-aligned at group
boundaries to enable seamless switching. The relay selects exactly one track from the switching
set to forward at any given time.

A **throughput level** refers to the bandwidth requirement of a track within a switching set.
Tracks in a switching set are distinguished by their throughput requirements, allowing the
relay to select an appropriate track based on available bandwidth.

In media applications carrying video at various qualites, tracks within a switching set are commonly called **renditions**, representing different quality levels (e.g., 1080p at 5 Mbps, 720p at 2 Mbps). Each rendition corresponds to exactly one MoQ track.

~~~
┌──────────────────────────────────────────────────────────────────────┐
│                    Switching Set to MoQ Mapping                      │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                        ┌──────────┐                                  │
│                        │  Source  │                                  │
│                        └────┬─────┘                                  │
│                             │                                        │
│              ┌──────────────┼──────────────┐                         │
│              │              │              │                         │
│              v              v              v                         │
│         ┌────────┐    ┌────────┐    ┌────────┐                       │
│         │ 5 Mbps │    │ 2 Mbps │    │800 kbps│  (3 throughput levels)│
│         └────────┘    └────────┘    └────────┘                       │
│              │              │              │                         │
│              v              v              v                         │
│         ┌────────┐    ┌────────┐    ┌────────┐                       │
│         │ Track 1│    │ Track 2│    │ Track 3│   (3 MoQ Tracks)      │
│         └────────┘    └────────┘    └────────┘                       │
│              │              │              │                         │
│              └──────────────┼──────────────┘                         │
│                             v                                        │
│              ┌─────────────────────────────┐                         │
│              │       Switching Set         │                         │
│              │   (time-aligned at group    │                         │
│              │        boundaries)          │                         │
│              └─────────────────────────────┘                         │
│                                                                      │
│ Relay receives all tracks, forwards exactly ONE based on bandwidth   │
└──────────────────────────────────────────────────────────────────────┘
~~~


## Content Requirements

For tracks to participate in a dynamic switching set, they:

* MUST be time aligned at group boundaries. This enables the relay to switch
  between tracks at group boundaries without disruption.

* MUST hold equivalent and independent versions of the same content at
  different throughput levels. DTS is not a solution for switching between
  tracks with inter-track dependencies (e.g., layered coding where higher
  layers depend on lower layers).

* SHOULD have consistent group durations across all tracks within a
  switching set to simplify relay timing decisions.

## Original Publisher Requirements

Original publishers are responsible for producing tracks that can be dynamically
switched by relays.

- MUST publish multiple time-aligned tracks of the same content at different throughput
  levels

- MUST ensure groups across tracks are temporally aligned to enable seamless switching
  at group boundaries

- SHOULD advertise throughput requirements for each track to enable bandwidth-based
  selection

See {{usecase-abr}}, {{usecase-videoconf}}, {{usecase-vr}}, and {{usecase-sports}} for examples.

## End Subscriber Requirements

End subscribers are responsible for establishing subscriptions and communicating switching
preferences to relays.

- MUST subscribe to all desired tracks within a switching set and indicate which
  subscriptions form a switching set via SWITCHING-SET-ASSIGNMENT

- MUST be prepared to process any track within the switching set

- MUST specify fraction and rank for each switching set to guide relay bandwidth allocation

- MAY dynamically adjust fraction and rank via REQUEST_UPDATE based on changing conditions

See {{usecase-videoconf}}, {{usecase-vr}}, {{usecase-screenshare}}, and {{usecase-sports}} for examples of dynamic updates.

## Relay Requirements

Relays are responsible for making dynamic track selection decisions and forwarding the
appropriate groups to downstream subscribers.

- MUST track available bandwidth to each downstream subscriber

- MUST select exactly one track per switching set to forward at any given time

- MUST switch tracks at group boundaries to maintain continuity

- MUST allocate bandwidth across switching sets based on fraction and rank

- MUST respond to fraction and rank updates from subscribers

- SHOULD select the highest-throughput track that fits within each set's allocated bandwidth

See {{usecase-abr}} for single switching set, {{usecase-videoconf}} for multiple sets,
and {{usecase-sports}} for rank-based protection examples.

# SWITCHING-SET-ASSIGNMENT Parameter {#switching-set-assignment}

The SWITCHING-SET-ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE,
REQUEST_UPDATE, or PUBLISH_OK message. This parameter assigns a subscription to a DTS
switching set and specifies bandwidth allocation and optional ranking.

~~~
SWITCHING-SET-ASSIGNMENT {
  Switching set ID (v64),
  Throughput threshold (v64),
  Set throughput fraction (v64),
  Activate switching (1),
  [Set rank (8)]
}
~~~

* **Switching set ID**: Integer identifying the switching set. A track MUST only be assigned
  to one switching set at a time. If a subscription attempts to assign a track that is
  already assigned to a different switching set, the relay MUST reject the subscription
  with a Parameter Error.

* **Throughput threshold**: Minimum throughput (kbps) required to select this track.

* **Set throughput fraction**: Relative weight for bandwidth allocation, expressed as an
  integer 1 <= N <= 10. Each set receives bandwidth proportional to its fraction:
  `target = B_total × fraction / sum_F`. Fractions are relative weights, not absolute
  percentages; for example, fractions of 6, 4, 3 (sum_F=13) allocate 46%, 31%, 23%
  respectively. This allows sets to be added or removed without requiring other sets to
  update their fractions. When multiple subscriptions in the same switching set specify
  different fraction values, the relay MUST use the value from the most recently received
  message for that set.

* **Activate switching**: When set to 0, DTS switching is paused for this set (use when
  more subscriptions will be added, or to temporarily freeze the current selection).
  When set to 1, the relay activates or resumes switching. Changes to the selected
  track take effect at the next group boundary.

* **Set rank** (optional): Degradation priority when bandwidth is constrained, expressed as
  an 8-bit unsigned integer (1-255). Default is 1. Values outside this range MUST result in
  a protocol error. Lower values indicate higher priority (protected from degradation).
  When bandwidth is sufficient, all sets receive their fraction-based allocation. When
  bandwidth is constrained, lower-ranked sets (higher numeric value) degrade first: they
  select lower-throughput tracks or receive no bandwidth, while higher-ranked sets maintain
  their target allocation. See {{allocation-algorithm}} for details.

# Subscriber Setup Methods {#subscriber-setup}

Subscribers can create switching sets through two methods. Both support single or multiple
switching sets and result in identical relay behavior. The key difference is where the
subscriber specifies SWITCHING-SET-ASSIGNMENT:

| Method | Track Discovery | Assignment In | Use When |
|--------|-----------------|---------------|----------|
| Individual SUBSCRIBE | Subscriber knows tracks | SUBSCRIBE | Tracks known in advance |
| SUBSCRIBE_NAMESPACE | Relay discovers tracks | PUBLISH_OK | Tracks discovered dynamically |

## Method 1: Individual SUBSCRIBE {#method-subscribe}

The subscriber sends a separate SUBSCRIBE message for each track, including the
SWITCHING-SET-ASSIGNMENT parameter to assign the track to a switching set.

Tracks are grouped into a switching set by specifying the same Switching set ID. The
subscriber sets activate=0 for all tracks except the last one in each set, then sets
activate=1 on the final track to signal that the set is complete and the relay should
begin active track selection.

The Set throughput fraction indicates what portion of total bandwidth this switching set
should receive. The fraction values across all sets should sum to 10 (representing 100%
of available bandwidth), though the relay will normalize if they do not.

This example shows a subscriber creating two switching sets for a live sports broadcast:
main camera (set 1, rank=1) and sideline camera (set 2, rank=2), each with two throughput
levels. The main camera has higher priority and is protected when bandwidth is constrained.

~~~
Subscriber                              Relay
  |                                       |
  |  // Switching Set 1: Main Camera (protected)
  |                                       |
  |  SUBSCRIBE track=main/1080p           |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=3000,           |
  |      fraction=6, rank=1, activate=0}  |
  |-------------------------------------->|
  |                                       |
  |  SUBSCRIBE track=main/480p            |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=800,            |
  |      fraction=6, rank=1, activate=1}  |
  |-------------------------------------->|
  |                                       |
  |  (Relay activates DTS for set 1)      |
  |                                       |
  |  // Switching Set 2: Sideline Camera (best effort)
  |                                       |
  |  SUBSCRIBE track=sideline/720p        |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, throughput=1500,           |
  |      fraction=4, rank=2, activate=0}  |
  |-------------------------------------->|
  |                                       |
  |  SUBSCRIBE track=sideline/360p        |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, throughput=400,            |
  |      fraction=4, rank=2, activate=1}  |
  |-------------------------------------->|
  |                                       |
  |  (Relay activates DTS for set 2)      |
  |                                       |
  |  Result: Two switching sets           |
  |    Set 1 (main):     60%, rank=1      |
  |    Set 2 (sideline): 40%, rank=2      |
  |                                       |
~~~

**Modifying the switching set**:

- **Add track**: Send SUBSCRIBE with SWITCHING-SET-ASSIGNMENT pointing to existing set
- **Remove track**: Unsubscribe from that track
- **Disable DTS**: Send REQUEST_UPDATE with activate=0
- **Resume DTS**: Send REQUEST_UPDATE with activate=1

## Method 2: SUBSCRIBE_NAMESPACE with TRACK_FILTER {#method-trackfilter}

The subscriber specifies selection criteria via TRACK_FILTER, and the relay dynamically
discovers tracks. The subscriber assigns tracks to switching sets via PUBLISH_OK rather
than SUBSCRIBE.

The relay monitors all tracks within the namespace and selects the top-N tracks ranked
by the specified property (e.g., LOUDNESS for voice activity detection, VIEWCOUNT for
popularity). For each matching track, the relay sends a PUBLISH message. The subscriber
assigns tracks to switching sets via PUBLISH_OK, just as in {{method-subscribe}} but
using PUBLISH_OK instead of SUBSCRIBE.

The selection is dynamic: as property values change, tracks may be added or removed
from the top-N set. When the selection changes:

1. For a newly selected track: Relay sends PUBLISH, subscriber responds with
   PUBLISH_OK + SWITCHING-SET-ASSIGNMENT
2. For a deselected track: Relay sends REQUEST_UPDATE with fwd=0

The subscriber is responsible for grouping tracks into switching sets based on
application-level knowledge (e.g., which tracks represent the same source at different
throughput levels).

~~~
TRACK_FILTER {
  Property Type (v64),       // e.g., LOUDNESS for voice activity
  MaxTracksSelected (v64),   // Number of tracks to select
  Timeout (v64)
}
~~~

Note: TRACK_FILTER is defined in the MOQT SUBSCRIBE_NAMESPACE extension
(see https://github.com/moq-wg/moq-transport/pull/1518).

~~~
Subscriber                              Relay
  |                                       |
  |  SUBSCRIBE_NAMESPACE (conf, video)    |
  |    TRACK_FILTER{LOUDNESS, max=4}      |
  |-------------------------------------->|
  |                                       |
  |  (Relay selects top 4 by LOUDNESS:    |
  |   alice, bob, carol, dave)            |
  |                                       |
  |  PUBLISH (alice, *)                   |
  |<--------------------------------------|
  |  PUBLISH_OK                           |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=2000,           |
  |      fraction=4, activate=0}          |
  |-------------------------------------->|
  |                                       |
  |  PUBLISH (bob, *)                     |
  |<--------------------------------------|
  |  PUBLISH_OK                           |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, throughput=2000,           |
  |      fraction=2, activate=0}          |
  |-------------------------------------->|
  |                                       |
  |  ... (carol, dave similarly)          |
  |                                       |
  |  (Relay creates 4 switching sets      |
  |   with subscriber-assigned fractions) |
  |                                       |
~~~

The subscriber assigns fraction values in PUBLISH_OK to control bandwidth allocation.
Switching set 1 gets fraction=4 (40%), sets 2-4 get fraction=2 (20% each).

# Relay Behavior {#relay-behavior}

This section defines the unified relay behavior for Dynamic Track Switching. The relay
processes switching sets identically regardless of how they were created (individual
SUBSCRIBE or SUBSCRIBE_NAMESPACE).

## Processing Switching Sets

The relay maintains state for each switching set:

- Set of subscriptions belonging to the switching set
- Set throughput fraction
- Set rank
- Activation state (active or pending)
- Currently selected track (Forward state = 1)

When the relay receives a subscription with SWITCHING-SET-ASSIGNMENT:

1. Add subscription to the specified switching set (create set if needed)
2. Set Forward state to 0 for the new subscription
3. Store the Set throughput fraction and rank as properties of the set
4. If Activate switching = 1, begin active track selection

## Dynamic Updates

Subscribers can update switching set parameters at any time via REQUEST_UPDATE. The relay
processes these updates and adjusts bandwidth allocation accordingly.

~~~
// Reallocate bandwidth: reduce set 1 from 40% to 20%, increase set 2 from 20% to 40%
REQUEST_UPDATE (set 1 subscription)
  SWITCHING-SET-ASSIGNMENT{id=1, fraction=2}

REQUEST_UPDATE (set 2 subscription)
  SWITCHING-SET-ASSIGNMENT{id=2, fraction=4}
~~~

## Bandwidth Allocation {#allocation-algorithm}

The relay maintains:

- `B_total`: Estimated downstream bandwidth capacity for the subscriber connection
- `sum_F`: Sum of all set fractions, updated incrementally as subscriptions are added or removed

On a periodic bandwidth estimate update or other selection trigger (e.g., new group
boundary), the relay executes the following algorithm:

~~~
for each set in rank order (ascending):
  set.target = B_total × set.fraction / sum_F
  set.allocated = min(set.target, B_remaining)
  set.selected = best track where throughput <= set.allocated
  B_remaining -= set.selected.throughput
~~~

Finally, set Forward state = 1 for each selected track, 0 for others. The rank ordering
ensures higher-priority sets receive their target allocation first; lower-priority sets
absorb any bandwidth shortfall.

~~~
┌─────────────────────────────────────────────────────────────────────┐
│                    Relay Adaptation Algorithm                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Precondition: sum_F maintained incrementally on subscribe/remove   │
│                                                                     │
│             +----------------------------------------+              │
│             | Sort sets by rank (ascending)          |              │
│             | B_remaining = B_total                  |              │
│             +-------------------+--------------------+              │
│                                 |                                   │
│                                 v                                   │
│             +----------------------------------------+              │
│             | for each set (in rank order):          |<-----+       │
│             |   target = B_total × fraction / sum_F  |      |       │
│             |   allocated = min(target, B_remaining) |      |       │
│             |   selected = best track where          |      |       │
│             |              throughput <= allocated   |      |       │
│             |   B_remaining -= selected throughput   |      |       │
│             +-------------------+--------------------+      |       │
│                                 |                           |       │
│                                 v                           |       │
│                    +--------------------+                   |       │
│                    | More sets?         |---YES-------------+       │
│                    +---------+----------+                           │
│                              | NO                                   │
│                              v                                      │
│                    +--------------------+                           │
│                    | Forward selected   |                           │
│                    | tracks             |                           │
│                    +--------------------+                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
~~~

## Fraction and Rank Interaction

**Fraction** determines target bandwidth allocation. Each set's target is:

~~~
set.target = B_total × set.fraction / sum_F
~~~

**Rank** determines allocation order when bandwidth is constrained. Sets are processed
in rank order (lower value = higher priority). Each set receives:

~~~
set.allocated = min(set.target, B_remaining)
~~~

When bandwidth is sufficient, all sets receive `allocated = target`. When bandwidth is
constrained, higher-priority sets (lower rank value) are protected while lower-priority
sets receive less than their target or nothing.

### Allocation Examples

#### Example 1: Sufficient Bandwidth (all sets receive target)

~~~
Set A: fraction=6, rank=1, tracks at 3000/1500 kbps
Set B: fraction=2, rank=2, tracks at 1000/500 kbps
Set C: fraction=2, rank=2, tracks at 1000/500 kbps

B_total = 5000 kbps, sum_F = 10, B_remaining = 5000

Set A (rank=1): target = 5000×6/10 = 3000
                allocated = min(3000, 5000) = 3000 → selects 3000 kbps
                B_remaining = 5000 - 3000 = 2000
Set B (rank=2): target = 5000×2/10 = 1000
                allocated = min(1000, 2000) = 1000 → selects 1000 kbps
                B_remaining = 2000 - 1000 = 1000
Set C (rank=2): target = 5000×2/10 = 1000
                allocated = min(1000, 1000) = 1000 → selects 1000 kbps
                B_remaining = 1000 - 1000 = 0

Result: All sets receive allocated = target
~~~

#### Example 2: Constrained Bandwidth (low-priority degrades)

Same setup, less bandwidth:

~~~
B_total = 4000 kbps, sum_F = 10, B_remaining = 4000

Set A (rank=1): target = 2400, allocated = min(2400, 4000) = 2400
                → selects 1500 kbps, B_remaining = 2500
Set B (rank=2): target = 800, allocated = min(800, 2500) = 800
                → selects 500 kbps, B_remaining = 2000
Set C (rank=2): target = 800, allocated = min(800, 2000) = 800
                → selects 500 kbps, B_remaining = 1500

Result: All sets receive allocated = target, but select lower throughput tracks
~~~

#### Example 3: Severely Constrained (low-priority dropped)

~~~
B_total = 2000 kbps, sum_F = 10, B_remaining = 2000

Set A (rank=1): target = 1200, allocated = min(1200, 2000) = 1200
                → selects 1500 kbps, B_remaining = 500
Set B (rank=2): target = 400, allocated = min(400, 500) = 400
                → selects 500 kbps, B_remaining = 0
Set C (rank=2): target = 400, allocated = min(400, 0) = 0
                → no track selected

Result: Set C receives allocated < target (dropped)
~~~

#### Example 4: Equal Rank (proportional degradation)

When all sets share the same rank, they degrade together:

~~~
Set A: fraction=5, rank=1, tracks at 2000/1000/500 kbps
Set B: fraction=3, rank=1, tracks at 1500/750 kbps
Set C: fraction=2, rank=1, tracks at 800/400 kbps

B_total = 3000 kbps, sum_F = 10, B_remaining = 3000

Set A (rank=1): target = 1500, allocated = min(1500, 3000) = 1500
                → selects 1000 kbps, B_remaining = 2000
Set B (rank=1): target = 900, allocated = min(900, 2000) = 900
                → selects 750 kbps, B_remaining = 1250
Set C (rank=1): target = 600, allocated = min(600, 1250) = 600
                → selects 400 kbps, B_remaining = 850

Result: All sets receive allocated = target, degrade proportionally
~~~

## Implementation Pseudocode

~~~
// sum_F maintained incrementally on subscribe/remove
function select_tracks(B_total, sum_F, sets[]):
  sort sets by rank (ascending)
  B_remaining = B_total

  for each set in sorted order:
    set.target    = B_total * set.fraction / sum_F
    set.allocated = min(set.target, B_remaining)
    set.selected  = best track where throughput <= set.allocated
    B_remaining  -= set.selected.throughput

  return sets[].selected
~~~

## Bandwidth Estimation

The relay MUST maintain a bandwidth estimate (B_total) for each downstream subscriber.
This estimate is obtained periodically from the QUIC stack (e.g., congestion window,
pacing rate, smoothed RTT) and MAY be supplemented by external sources such as SCONE
{{?I-D.ietf-scone-protocol}} or application-level feedback. The estimate drives track
selection decisions for all switching sets associated with that subscriber

# Examples {#examples}

This section provides focused examples demonstrating relay behavior in common
scenarios. Each example references a detailed use case in {{usecase-appendix}}
that includes architecture diagrams and extended requirements.

## Example: 2-Track ABR

This example demonstrates the simplest DTS scenario: a single switching set with two
tracks at different throughput levels from one source. See {{usecase-abr}} for the full use case
description and architecture.

**Setup**: Video at 1080p (2000 kbps) and 480p (500 kbps), fraction=10

The subscriber creates a single switching set by sending two SUBSCRIBE messages with
the same set ID. The first subscription (1080p) sets activate=0 to indicate more tracks
will follow. The second subscription (480p) sets activate=1 to signal the set is complete.
Since this is the only switching set, fraction=10 allocates 100% of bandwidth to it.

~~~
Subscriber subscribes to both tracks:
  SUBSCRIBE (video, 1080p)
    SWITCHING-SET-ASSIGNMENT{id=1, throughput=2000, fraction=10}
  SUBSCRIBE (video, 480p)
    SWITCHING-SET-ASSIGNMENT{id=1, throughput=500, activate=1}
~~~

Once the relay receives the second subscription with activate=1, it begins active track
selection. The relay evaluates the bandwidth estimate and selects the highest-quality
track whose throughput threshold fits within the allocated bandwidth.

When bandwidth is 3 Mbps, the relay calculates B_set = 3000 kbps. The 1080p track
requires 2000 kbps, which fits, so the relay selects 1080p and sets Forward=1 for that
track. The 480p track gets Forward=0.

~~~
Relay behavior at 3 Mbps:
  B_set = 3000 × 10/10 = 3000 kbps
  Select: 2000 <= 3000 → 1080p
  Forward: 1080p groups
~~~

If bandwidth drops to 1 Mbps, the relay recalculates B_set = 1000 kbps. The 1080p track
(2000 kbps) no longer fits, so the relay switches to 480p (500 kbps). At the next group
boundary, Forward state changes: 1080p gets Forward=0, 480p gets Forward=1.

~~~
Relay behavior at 1 Mbps:
  B_set = 1000 × 10/10 = 1000 kbps
  Select: 2000 > 1000, 500 <= 1000 → 480p
  Forward: 480p groups
~~~

~~~
┌──────────────────────────────────────────────────────────────────────┐
│                      2-Track ABR Example                             │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Bandwidth = 3 Mbps           Bandwidth = 1 Mbps                    │
│                                                                      │
│   ┌────────┐  ┌────────┐       ┌────────┐  ┌────────┐                │
│   │ 1080p  │  │  480p  │       │ 1080p  │  │  480p  │                │
│   │ 2 Mbps │  │ 500kbps│       │ 2 Mbps │  │ 500kbps│                │
│   └───┬────┘  └────────┘       └────────┘  └───┬────┘                │
│       │                                        │                     │
│  [SELECTED]                               [SELECTED]                 │
│       │                                        │                     │
│       v                                        v                     │
│   Forward                                  Forward                   │
│   1080p                                    480p                      │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
~~~

## Example: 2x2 Grid Layout

This example demonstrates multiple switching sets with equal bandwidth allocation. Four
participants are displayed in a grid layout, each receiving the same share of bandwidth.
See {{usecase-videoconf}} for the full use case description including active speaker
priority handling.

**Setup**: Each participant has 720p (800 kbps) and 360p (300 kbps), equal fractions

The subscriber uses SUBSCRIBE_NAMESPACE with TRACK_FILTER to request the top 4 tracks by
LOUDNESS. The relay sends PUBLISH for each matching track. The subscriber assigns tracks
to switching sets via PUBLISH_OK with SWITCHING-SET-ASSIGNMENT, using fraction=2 for each
set (20% of bandwidth per participant). The remaining 20% provides headroom for audio and
protocol overhead.

~~~
Subscriber                              Relay
  |                                       |
  |  SUBSCRIBE_NAMESPACE (conf, video)    |
  |    TRACK_FILTER{LOUDNESS, max=4}      |
  |-------------------------------------->|
  |                                       |
  |  PUBLISH (alice, *)                   |
  |<--------------------------------------|
  |  PUBLISH_OK                           |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, fraction=2}                |
  |-------------------------------------->|
  |                                       |
  |  PUBLISH (bob, *)                     |
  |<--------------------------------------|
  |  PUBLISH_OK                           |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, fraction=2}                |
  |-------------------------------------->|
  |                                       |
  |  ... (carol id=3, dave id=4 similarly)|
  |                                       |
~~~

With equal fractions and equal rank (default), the relay uses equal allocation mode. All
participants receive the same quality level, and when bandwidth drops, all degrade together.

~~~
At 4 Mbps total:
  B_set = 4000 × 2/10 = 800 kbps per set
  All sets: 800 <= 800 → select 720p
  Forward: alice@720p, bob@720p, carol@720p, dave@720p

At 2 Mbps total:
  B_set = 2000 × 2/10 = 400 kbps per set
  All sets: 800 > 400, 300 <= 400 → select 360p
  Forward: alice@360p, bob@360p, carol@360p, dave@360p
~~~

~~~
┌──────────────────────────────────────────────────────────────────────┐
│                      2x2 Grid Layout                                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  4 Mbps: All participants at 720p                                    │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐                            │
│  │  Alice (720p)   │  │   Bob (720p)    │                            │
│  │  fraction=2     │  │  fraction=2     │                            │
│  └─────────────────┘  └─────────────────┘                            │
│  ┌─────────────────┐  ┌─────────────────┐                            │
│  │  Carol (720p)   │  │  Dave (720p)    │                            │
│  │  fraction=2     │  │  fraction=2     │                            │
│  └─────────────────┘  └─────────────────┘                            │
│                                                                      │
│  2 Mbps: All participants degrade equally to 360p                    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
~~~

## Example: VR Headset / Foveated Rendering

This example demonstrates dynamic bandwidth reallocation based on user interaction.
In foveated rendering, the tile where the user is looking (gaze tile) receives higher
quality than peripheral tiles. As the user's gaze moves, bandwidth allocation shifts
in real-time. See {{usecase-vr}} for the full use case description.

**Setup**: 5 tiles, each with hi (1 Mbps) and lo (200 kbps). Gaze tile gets fraction=4
and rank=1 (protected); peripheral tiles get fraction=1 and rank=2 (best effort).

The subscriber uses individual SUBSCRIBE messages since the tile layout is known in
advance. Each tile becomes a separate switching set with two tracks (hi and lo). The
gaze tile is protected with rank=1, ensuring it maintains quality when bandwidth drops.

~~~
Subscriber                              Relay
  |                                       |
  |  // Peripheral tiles (rank=2)         |
  |  SUBSCRIBE (tile1, hi)                |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=1000,           |
  |      fraction=1, rank=2}              |
  |-------------------------------------->|
  |  SUBSCRIBE (tile1, lo)                |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=200, activate=1}|
  |-------------------------------------->|
  |                                       |
  |  ... (tiles 2, 4, 5 similarly)        |
  |                                       |
  |  // Gaze tile (rank=1, protected)     |
  |  SUBSCRIBE (tile3, hi)                |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=3, throughput=1000,           |
  |      fraction=4, rank=1}              |
  |-------------------------------------->|
  |  SUBSCRIBE (tile3, lo)                |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=3, throughput=200, activate=1}|
  |-------------------------------------->|
  |                                       |
~~~

At 3 Mbps with sufficient bandwidth, all tiles receive their target allocation:

~~~
At 3 Mbps (sufficient): sum_F=8, B_remaining=3000
  Gaze (rank=1):   target=1500, allocated=1500 → hi (1 Mbps)
                   B_remaining=2000
  Peripheral×4:    target=375 each, allocated=375 → lo (200 kbps)
~~~

When bandwidth drops, the gaze tile (rank=1) is protected while peripheral tiles
absorb the shortfall:

~~~
At 1.5 Mbps (constrained): sum_F=8, B_remaining=1500
  Gaze (rank=1):   target=750, allocated=750 → lo (200 kbps fits)
                   B_remaining=1300
  Peripheral×4:    target=188 each → lo (200 kbps)
  Result: Gaze processed first, guaranteed its allocation
~~~

When the user's gaze shifts from tile 3 to tile 5, the subscriber sends
REQUEST_UPDATE messages to swap both fraction and rank:

~~~
When gaze shifts from tile 3 to tile 5:
  REQUEST_UPDATE (tile 3)
    SWITCHING-SET-ASSIGNMENT{fraction=1, rank=2}  // demote
  REQUEST_UPDATE (tile 5)
    SWITCHING-SET-ASSIGNMENT{fraction=4, rank=1}  // promote

Relay reallocates: tile 5 now protected, tile 3 best-effort
~~~

~~~
┌─────────────────────────────────────────────────────────────────────┐
│                    VR Foveated Rendering                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  360° View with Gaze at Tile 3 (protected, rank=1):                 │
│                                                                     │
│  ┌──────┐ ┌──────┐ ┌──────────┐ ┌──────┐ ┌──────┐                   │
│  │Tile 1│ │Tile 2│ │  Tile 3  │ │Tile 4│ │Tile 5│                   │
│  │  lo  │ │  lo  │ │   hi     │ │  lo  │ │  lo  │                   │
│  │f=1,r2│ │f=1,r2│ │ f=4,r1   │ │f=1,r2│ │f=1,r2│                   │
│  └──────┘ └──────┘ └──────────┘ └──────┘ └──────┘                   │
│                      [GAZE]                                         │
│                                                                     │
│  Gaze shifts to Tile 5 → REQUEST_UPDATE swaps fraction and rank     │
│                                                                     │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐                   │
│  │Tile 1│ │Tile 2│ │Tile 3│ │Tile 4│ │  Tile 5  │                   │
│  │  lo  │ │  lo  │ │  lo  │ │  lo  │ │   hi     │                   │
│  │f=1,r2│ │f=1,r2│ │f=1,r2│ │f=1,r2│ │ f=4,r1   │                   │
│  └──────┘ └──────┘ └──────┘ └──────┘ └──────────┘                   │
│                                         [GAZE]                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
~~~

## Example: Live Sports with Protected Main Camera

This example demonstrates Set-Rank mode where one stream is protected during bandwidth
degradation. A live sports broadcast has a main camera that must maintain quality, while
a secondary replay camera can degrade if bandwidth is constrained.

**Setup**: Main camera with 1080p (3 Mbps) and 480p (800 kbps), rank=1. Replay camera
with 720p (1.5 Mbps) and 360p (400 kbps), rank=2.

The subscriber uses individual SUBSCRIBE to create two switching sets with different
ranks. The main camera (rank=1) has higher priority than the replay camera (rank=2).
When bandwidth is sufficient, both receive their best quality. When bandwidth drops,
the relay protects the main camera by degrading the replay camera first.

~~~
Subscriber                              Relay
  |                                       |
  |  // Main camera - protected (rank=1)  |
  |  SUBSCRIBE (main, 1080p)              |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=3000,           |
  |      fraction=6, rank=1}              |
  |-------------------------------------->|
  |  SUBSCRIBE (main, 480p)               |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=800,            |
  |      fraction=6, rank=1, activate=1}  |
  |-------------------------------------->|
  |                                       |
  |  // Replay camera - best effort (rank=2)|
  |  SUBSCRIBE (replay, 720p)             |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, throughput=1500,           |
  |      fraction=4, rank=2}              |
  |-------------------------------------->|
  |  SUBSCRIBE (replay, 360p)             |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=2, throughput=400,            |
  |      fraction=4, rank=2, activate=1}  |
  |-------------------------------------->|
  |                                       |
~~~

Sets are processed in rank order; the main camera (rank=1) receives its allocation first,
and the replay camera (rank=2) absorbs any shortfall.

~~~
At 5 Mbps (sufficient): sum_F=10, B_remaining=5000
  main (rank=1):   target=3000, allocated=min(3000,5000)=3000 → 1080p
                   B_remaining=2000
  replay (rank=2): target=2000, allocated=min(2000,2000)=2000 → 720p
  Result: Both receive allocated = target

At 3.5 Mbps (constrained): sum_F=10, B_remaining=3500
  main (rank=1):   target=2100, allocated=min(2100,3500)=2100 → 1080p (3000)
                   B_remaining=500
  replay (rank=2): target=1400, allocated=min(1400,500)=500 → 360p (400)
  Result: replay receives allocated < target (degraded)

At 2 Mbps (severely constrained): sum_F=10, B_remaining=2000
  main (rank=1):   target=1200, allocated=min(1200,2000)=1200 → 480p (800)
                   B_remaining=1200
  replay (rank=2): target=800, allocated=min(800,1200)=800 → 720p won't fit → 400
  Result: Both degraded, main processed first
~~~

~~~
┌──────────────────────────────────────────────────────────────────────┐
│                    Set-Rank Mode: Protected Stream                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  5 Mbps: Both streams at best quality                                │
│  ┌─────────────────┐  ┌─────────────────┐                            │
│  │  Main (1080p)   │  │  Replay (720p)  │                            │
│  │  rank=1         │  │  rank=2         │                            │
│  └─────────────────┘  └─────────────────┘                            │
│                                                                      │
│  3.5 Mbps: Main protected, replay degrades first                     │
│  ┌─────────────────┐  ┌─────────────────┐                            │
│  │  Main (1080p)   │  │  Replay (360p)  │                            │
│  │  [PROTECTED]    │  │  [DEGRADED]     │                            │
│  └─────────────────┘  └─────────────────┘                            │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
~~~

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

## Use Cases: Single Switching Set

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
            ┌──────────────────────────────────────────┐
            │          Original Publisher              │
            │                                          │
            │  ┌─────────┐ ┌─────────┐ ┌─────────┐     │
            │  │ 1080p   │ │  720p   │ │  480p   │     │
            │  │ 5 Mbps  │ │ 2 Mbps  │ │ 800kbps │     │
            │  └────┬────┘ └────┬────┘ └────┬────┘     │
            │       │           │           │          │
            └───────┼───────────┼───────────┼──────────┘
                    │           │           │
                    ▼           ▼           ▼
            ┌──────────────────────────────────────────┐
            │                 Relay                    │
            │                                          │
            │  Receives all renditions, selects one    │
            │  based on throughput thresholds and      │
            │  downstream bandwidth                    │
            └───────────────────┬──────────────────────┘
                                │
                                │  Selected rendition
                                │  (e.g., 720p @ 2 Mbps)
                                ▼
            ┌──────────────────────────────────────────┐
            │             End Subscriber               │
            │                                          │
            │  Receives single stream, quality varies  │
            │  based on available bandwidth            │
            │                                          │
            └──────────────────────────────────────────┘
~~~

## Use Cases: Multiple Switching Sets

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
sets (one per participant) and assigns Set throughput fraction values to indicate bandwidth
allocation—for example, giving the active speaker a higher fraction than other participants.
When the active speaker changes, the subscriber adjusts fraction values. The relay allocates
its forwarding capacity across all switching sets according to the subscriber-indicated
fractions, selecting appropriate quality levels for each participant stream to fit within
the allocated bandwidth.

~~~
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Original Publishers                                 │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │ Participant │  │ Participant │  │ Participant │  │ Participant │   ...    │
│  │     A       │  │     B       │  │     C       │  │     D       │          │
│  │ hi/med/lo   │  │ hi/med/lo   │  │ hi/med/lo   │  │ hi/med/lo   │          │
│  │ prio:1/2/3  │  │ prio:1/2/3  │  │ prio:1/2/3  │  │ prio:1/2/3  │          │
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
│     - Subscriber-indicated Set throughput fraction per set                   │
│     - Total available bandwidth                                              │
│     - Publisher-indicated rendition throughput thresholds                    │
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
publishing each content type as a separate switching set with content type indicated in the
catalog. The end subscriber subscribes to both switching sets, assigning higher weight to
screen share than camera video based on content type, and may specify a minimum acceptable
quality (throughput) for the screen share to ensure text remains readable. The relay manages
bandwidth allocation between the two content types, degrading camera video quality before
reducing screen share quality when bandwidth becomes constrained.

~~~
┌────────────────────────────────────────────────────────────────────┐
│                        Original Publisher                          │
│                                                                    │
│  ┌────────────────────────┐    ┌────────────────────────┐          │
│  │     Screen Share       │    │     Camera Video       │          │
│  │ ┌───────┐ ┌───────┐    │    │ ┌───────┐ ┌───────┐    │          │
│  │ │1080p  │ │ 720p  │    │    │ │ 720p  │ │ 360p  │    │          │
│  │ │2 Mbps │ │800kbps│    │    │ │1.5Mbps│ │400kbps│    │          │
│  │ └───┬───┘ └───┬───┘    │    │ └───┬───┘ └───┬───┘    │          │
│  │     │         │        │    │     │         │        │          │
│  └─────┼─────────┼────────┘    └─────┼─────────┼────────┘          │
│        │         │                   │         │                   │
└────────┼─────────┼───────────────────┼─────────┼───────────────────┘
         │         │                   │         │
         ▼         ▼                   ▼         ▼
┌────────────────────────────────────────────────────────────────────┐
│                            Relay                                   │
│                                                                    │
│  Manages two switching sets                                        │
│  Allocates bandwidth based on subscriber-indicated weights         │
│                                                                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               │  Screen@1080p + Camera@360p
                               │  (prioritizing screen readability)
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                         End Subscriber                             │
│                                                                    │
│  ┌─────────────────────────────────┐  ┌──────────────────┐         │
│  │                                 │  │                  │         │
│  │        Screen Share             │  │  Camera (small)  │         │
│  │     (high quality for text)     │  │  (lower quality) │         │
│  │                                 │  │                  │         │
│  └─────────────────────────────────┘  └──────────────────┘         │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
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
subscribes to all tiles within the field of view and as gaze direction changes, assigns
a higher Set throughput fraction to the gaze tile and lower fractions to peripheral tiles.
The relay responds rapidly to these updates, reallocating bandwidth to deliver high quality
for the gaze tile while maintaining lower quality for surrounding tiles.

~~~
┌────────────────────────────────────────────────────────────────────┐
│                      VR Headset (Publisher)                        │
│                                                                    │
│  360° Video Tiles (each with quality variants)                     │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │
│  │ Tile 1 │ │ Tile 2 │ │ Tile 3 │ │ Tile 4 │ │ Tile 5 │  ...       │
│  │ hi/lo  │ │ hi/lo  │ │ hi/lo  │ │ hi/lo  │ │ hi/lo  │            │
│  └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘            │
│      │          │    [GAZE]│          │          │                 │
└──────┼──────────┼──────────┼──────────┼──────────┼─────────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
┌────────────────────────────────────────────────────────────────────┐
│                            Relay                                   │
│                                                                    │
│  Receives gaze direction updates (weights) from subscriber         │
│  Allocates bandwidth: high quality to gaze tile, lower to periph.  │
│                                                                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               │  Tile3@hi, others@lo
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                         End Subscriber                             │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                    360° Rendered View                    │      │
│  │  ┌─────┐ ┌─────┐ ┌─────────────┐ ┌─────┐ ┌─────┐         │      │
│  │  │ lo  │ │ lo  │ │     hi      │ │ lo  │ │ lo  │         │      │
│  │  │     │ │     │ │ (gaze area) │ │     │ │     │         │      │
│  │  └─────┘ └─────┘ └─────────────┘ └─────┘ └─────┘         │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
~~~

## Use Cases: Switching Sets with High-Priority Streams

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
and publishes the HUD as a separate fixed-bandwidth track. The publisher
minimizes encoding latency to maintain gameplay responsiveness. The end subscriber subscribes
to both the game video switching set and the HUD track, indicating that the HUD requires
guaranteed bandwidth. The relay reserves bandwidth for the HUD first, then selects the
appropriate game video quality from the remaining capacity, prioritizing low latency
throughout the forwarding path.

~~~
┌────────────────────────────────────────────────────────────────────┐
│                     Game Server (Publisher)                        │
│                                                                    │
│  ┌───────────────────────────────┐  ┌─────────────────────┐        │
│  │      Game World Video         │  │    HUD/Overlay      │        │
│  │ ┌───────┐ ┌───────┐ ┌───────┐ │  │ ┌───────┐           │        │
│  │ │4K/60  │ │1080/60│ │720/60 │ │  │ │ Fixed │           │        │
│  │ │25Mbps │ │8 Mbps │ │3 Mbps │ │  │ │200kbps│           │        │
│  │ └───┬───┘ └───┬───┘ └───┬───┘ │  │ └───┬───┘           │        │
│  └─────┼─────────┼─────────┼─────┘  └─────┼───────────────┘        │
│        │         │         │              │                        │
└────────┼─────────┼─────────┼──────────────┼────────────────────────┘
         │         │         │              │
         ▼         ▼         ▼              ▼
┌────────────────────────────────────────────────────────────────────┐
│                            Relay                                   │
│                                                                    │
│  Reserves HUD bandwidth (subscriber-indicated guaranteed stream)   │
│  Selects game quality from remainder based on throughput           │
│                                                                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               │  Game@1080/60 + HUD@fixed
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                       Player (Subscriber)                          │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ ┌──────────────────────────────────────────────────────┐ │      │
│  │ │                                                      │ │      │
│  │ │                   Game World                         │ │      │
│  │ │                 (adaptive quality)                   │ │      │
│  │ │                                                      │ │      │
│  │ └──────────────────────────────────────────────────────┘ │      │
│  │ ┌──────────────┐                    ┌──────────────┐     │      │
│  │ │Health: XXX---│                    │ Ammo: 30/120 │     │      │
│  │ └──────────────┘  HUD (guaranteed)  └──────────────┘     │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
~~~

### Live Sports Multi-View {#usecase-sports}

Live sports broadcasts offer multiple camera angles: main game camera, sideline cameras,
aerial views, and isolated player cameras. Viewers may want to watch multiple angles
simultaneously, with the ability to prioritize different views. A stats overlay stream
provides real-time game information. Bandwidth must be allocated across these streams
based on viewer preferences that may change during the event (e.g., switching focus to
replay angle).

The original publisher (broadcast origin) encodes each camera angle at multiple quality
levels and publishes a stats overlay as a fixed-bandwidth stream. All streams share
a common time reference for synchronization. The end subscriber subscribes to desired
camera angles and the stats overlay, assigning Set throughput fraction values to indicate
bandwidth allocation for each view. During highlights or replays, the subscriber dynamically
adjusts fractions to shift focus to the relevant camera. The relay allocates bandwidth
according to the fractions, maintains temporal sync across all forwarded streams, and
responds promptly to fraction changes.

~~~
┌────────────────────────────────────────────────────────────────────┐
│                   Broadcast Origin (Publisher)                     │
│                                                                    │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐       │
│  │ Main Cam   │ │  Sideline  │ │   Aerial   │ │Stats/Score │       │
│  │ hi/med/lo  │ │ hi/med/lo  │ │ hi/med/lo  │ │  (fixed)   │       │
│  └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘       │
│        │              │              │              │              │
└────────┼──────────────┼──────────────┼──────────────┼──────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
┌────────────────────────────────────────────────────────────────────┐
│                            Relay                                   │
│                                                                    │
│  Reserves stats bandwidth (subscriber-indicated guaranteed stream) │
│  Allocates remaining bandwidth based on subscriber-indicated wts   │
│                                                                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               │  Main@hi, Sideline@med, Aerial@lo, Stats
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                       Viewer (Subscriber)                          │
│                                                                    │
│  ┌────────────────────────────────┐  ┌───────────────────┐         │
│  │                                │  │   Sideline View   │         │
│  │      Main Camera View          │  │  (medium quality) │         │
│  │      (high quality)            │  ├───────────────────┤         │
│  │                                │  │   Aerial View     │         │
│  │                                │  │   (low quality)   │         │
│  └────────────────────────────────┘  └───────────────────┘         │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ SCORE: Home 2 - Away 1 | Time: 73:24 | Possession: 58%   │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
~~~

### Teleoperation and Robotics {#usecase-teleop}

Remote operation of robots, drones, or industrial equipment requires streaming multiple
video feeds with different importance levels. The primary control camera (showing the
manipulation task) requires highest quality and lowest latency. Secondary cameras
providing situational awareness can accept lower quality. Sensor telemetry streams
compete for bandwidth with video feeds.

The original publisher (robot or drone) encodes the primary control camera at multiple
quality levels with minimal encoding latency, encodes situational cameras at multiple
levels, and publishes sensor telemetry as a separate fixed-bandwidth stream. All streams
share a common time reference. The end subscriber subscribes to the primary camera
with highest priority and specifies latency requirements, subscribes to telemetry with
guaranteed bandwidth, and subscribes to situational cameras with lower priority. The
relay prioritizes latency for the primary camera, reserves bandwidth for telemetry,
and allocates remaining capacity to situational cameras.

~~~
┌────────────────────────────────────────────────────────────────────┐
│                        Robot (Publisher)                           │
│                                                                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │ Primary Cam │ │  Left Cam   │ │  Right Cam  │ │  Telemetry  │   │
│  │(manipulate) │ │(situational)│ │(situational)│ │  (sensors)  │   │
│  │ hi/med/lo   │ │   hi/lo     │ │   hi/lo     │ │  (fixed)    │   │
│  └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘   │
│         │               │               │               │          │
└─────────┼───────────────┼───────────────┼───────────────┼──────────┘
          │               │               │               │
          ▼               ▼               ▼               ▼
┌────────────────────────────────────────────────────────────────────┐
│                            Relay                                   │
│                                                                    │
│  Allocates bandwidth based on subscriber-indicated weights         │
│  Latency-critical: minimize delay for primary control feed         │
│                                                                    │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               │  Primary@hi, Left@lo, Right@lo, Telemetry
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│                    Operator Console (Subscriber)                   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                                                          │      │
│  │          Primary Camera (high quality, low latency)      │      │
│  │                                                          │      │
│  └──────────────────────────────────────────────────────────┘      │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │
│  │  Left Camera   │  │  Right Camera  │  │   Telemetry    │        │
│  │  (low quality) │  │  (low quality) │  │ Temp: 45°C     │        │
│  │                │  │                │  │ Battery: 73%   │        │
│  └────────────────┘  └────────────────┘  └────────────────┘        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
