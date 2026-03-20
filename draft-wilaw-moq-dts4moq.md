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

This document defines Dynamic Track Switching (DTS) for Media over QUIC Transport (MOQT).
DTS enables relays to dynamically select which track to forward from a set of related
subscriptions based on available downstream bandwidth. Two approaches are specified:
a switching-set-assignment approach where subscribers explicitly group subscriptions,
and a track-filter approach using SUBSCRIBE_NAMESPACE with publisher-advertised track
properties. For scenarios with multiple switching sets, a relay adaptation algorithm
allocates bandwidth based on subscriber_priority, enabling use cases such as
adaptive bitrate streaming, video conferencing, and cloud gaming.


--- middle

# Introduction

This document defines Dynamic Track Switching (DTS) for Media over QUIC Transport [MOQT].
DTS enables relays to dynamically select which track to forward from a set of related
subscriptions based on available downstream bandwidth.

This document specifies two solution approaches:

- **Single Switching Set**: For scenarios like Adaptive Bitrate Streaming where one stream
  has multiple quality renditions.

- **Multiple Switching Sets**: For scenarios like video conferencing where multiple streams
  each have quality renditions and bandwidth must be allocated across them.

Both approaches share a common relay adaptation algorithm ({{relay-adaptation-algorithm}})
that handles bandwidth allocation and rendition selection.

# Requirements

This section describes the requirements that Dynamic Track Switching places on original
publishers, end subscribers, and relays. These requirements are derived from the use cases
described in {{usecase-appendix}}.

The use cases cover a range of real-world applications:

- **Single switching set:** A single media source with multiple quality renditions, such as
  Adaptive Bitrate Streaming ({{usecase-abr}}).

- **Multiple switching sets:** Several concurrent media sources each with quality renditions,
  requiring bandwidth allocation based on relative priorities—video conferencing
  ({{usecase-videoconf}}), screen sharing ({{usecase-screenshare}}), and VR/AR streaming
  ({{usecase-vr}}).

- **Switching set(s) with high-priority streams:** Adaptive media combined with fixed-bandwidth
  streams requiring priority delivery—cloud gaming ({{usecase-gaming}}), live sports
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

- SHOULD publish high-priority or fixed-bandwidth streams (such as HUD overlays, stats, or
  telemetry) as separate tracks with low publisher_priority values
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

- MUST indicate relative importance via subscriber_priority for each switching set to guide
  relay bandwidth allocation (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-vr}},
  {{usecase-sports}}, {{usecase-teleop}})

- SHOULD use content type characteristics from the catalog to determine appropriate relative
  priorities or weights (see {{usecase-screenshare}}, {{usecase-gaming}}, {{usecase-teleop}})

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
appropriate groups to downstream subscribers. Relays do not have access to the catalog;
switching metadata is obtained from track properties in publish messages and from
subscribe parameters in subscribe or subscribe namespace messages.

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

- MUST respect subscriber_priority values when selecting groups to forward
  (see {{usecase-videoconf}}, {{usecase-screenshare}}, {{usecase-sports}})

- SHOULD implement degradation based on subscriber_priority when total capacity is insufficient,
  reducing quality of lower-priority content before higher-priority content
  (see {{usecase-videoconf}}, {{usecase-screenshare}})

- MUST respond rapidly to subscriber_priority updates from subscribers
  (see {{usecase-videoconf}}, {{usecase-vr}}, {{usecase-sports}})

- MUST prioritize forwarding groups for high-priority streams before performing adaptive
  allocation for other streams (see {{usecase-gaming}}, {{usecase-sports}}, {{usecase-teleop}})

- MUST prioritize latency for latency-critical streams such as control cameras or interactive
  game video (see {{usecase-gaming}}, {{usecase-teleop}})

- MUST maintain temporal synchronization when forwarding multiple related streams
  (see {{usecase-sports}}, {{usecase-teleop}})

- SHOULD minimize switching latency when bandwidth conditions change (see {{usecase-abr}})

- SHOULD minimize switching latency when bandwidth conditions change

# Single Switching Set {#single-switching-set}

A single switching set represents one media source with multiple quality renditions. The relay
selects exactly one rendition to forward based on available bandwidth. This section describes
two approaches for implementing single switching set DTS.

## Approach 1: SUBSCRIBE with Switching-Set Parameters {#approach-subscribe}

The subscriber explicitly groups tracks into a switching set using new SUBSCRIBE parameters.

### Subscribe Parameters

#### SWITCHING-SET-ASSIGNMENT Parameter

The SWITCHING-SET-ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE or
REQUEST_UPDATE message. This parameter assigns the subscription to a DTS switching set.

~~~
SWITCHING-SET-ASSIGNMENT {
  Switching set ID (vi64),
  Throughput threshold (vi64),
  Set throughput fraction (vi64),
  Activate switching (1)
}
~~~

* **Switching set ID**: Integer identifying the switching set. Zero indicates no switching set.
* **Throughput threshold**: Minimum throughput (kbps) required to select this track.
  MUST be omitted if Switching set ID is zero.
* **Selection time limit**: Maximum time (ms) the relay waits for this track's group before
  disqualifying it. MUST be omitted if Switching set ID is zero.

The relative priority of switching sets for bandwidth allocation is indicated using the
SUBSCRIBER_PRIORITY parameter defined in [MOQT] Section 9.3.5. Lower subscriber_priority
values indicate higher importance. The relay adaptation algorithm ({{relay-adaptation-algorithm}})
uses subscriber_priority to allocate bandwidth across multiple switching sets.

#### DTS-ACTIVATION Parameter

The DTS-ACTIVATION parameter (Parameter Type 0x43) enables or disables DTS for a switching set.

For tracks to participate in a dynamic switching set, they

* **Switching set ID**: References a previously defined switching set.
* **State**: 1 activates DTS, 0 deactivates it.

### Client Workflow

1. Client determines available renditions via catalog or out-of-band mechanism.
2. Client selects a unique integer identifier for the switching set.
3. For each rendition, client sends SUBSCRIBE with SWITCHING-SET-ASSIGNMENT parameter
   containing: set ID, throughput threshold, and selection time limit. The SUBSCRIBER_PRIORITY
   parameter indicates the relative importance of this switching set.
4. On the final SUBSCRIBE, client includes DTS-ACTIVATION with State=1 to activate.

~~~
Client                                  Relay
  |                                       |
  |  SUBSCRIBE track=1080p                |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=5000,           |
  |      timeout=100}                     |
  |    SUBSCRIBER_PRIORITY=10             |
  |-------------------------------------->|
  |                                       |
  |  SUBSCRIBE track=720p                 |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=2000,           |
  |      timeout=100}                     |
  |    SUBSCRIBER_PRIORITY=10             |
  |-------------------------------------->|
  |                                       |
  |  SUBSCRIBE track=480p                 |
  |    SWITCHING-SET-ASSIGNMENT{          |
  |      id=1, throughput=800,            |
  |      timeout=100}                     |
  |    SUBSCRIBER_PRIORITY=10             |
  |    DTS-ACTIVATION{id=1, state=1}      |
  |-------------------------------------->|
  |                                       |
  |  (Relay activates DTS for set 1)      |
  |                                       |
~~~

To modify the switching set:

- **Disable DTS**: Send REQUEST_UPDATE with DTS-ACTIVATION State=0
- **Remove track**: Send REQUEST_UPDATE with SWITCHING-SET-ASSIGNMENT id=0
- **Add track**: Send new SUBSCRIBE with SWITCHING-SET-ASSIGNMENT pointing to existing set

### Relay Workflow

1. On receiving SWITCHING-SET-ASSIGNMENT, relay adds subscription to the switching set
   (creating it if needed). Forward state is set to 0.

2. On receiving DTS-ACTIVATION with State=1, relay activates track selection and begins
   monitoring bandwidth using the algorithm in {{relay-adaptation-algorithm}}.

3. When Group N arrives from any track in the set, relay evaluates which track to forward:
   - Select track with highest throughput threshold <= estimated bandwidth
   - Set Forward state to 1 for selected track, 0 for others
   - If no track fits, set all Forward states to 0

4. If the arriving track is the selected track, forward immediately.

5. If the arriving track is not selected, cache it and start selection timer.

6. If selected track arrives before timeout, forward it and stop timer.

7. If timeout expires, remove that track from candidates and re-evaluate selection.

## Approach 2: Track Filter with Publisher Properties {#approach-trackfilter}

The subscriber uses SUBSCRIBE with a TRACK_FILTER parameter. The publisher advertises
track properties that the relay uses for selection.

### Publisher Track Properties

Publishers advertise metadata on each track to enable relay-based selection. The
publisher_priority property defined in [MOQT] Section 11.3 indicates relative quality
ordering where lower values indicate higher priority (higher quality).

~~~
PUBLISH track=(video, 1080p)
  Track Properties:
    THROUGHPUT = 5000 (kbps)
    publisher_priority = 1 (highest quality)

PUBLISH track=(video, 720p)
  Track Properties:
    THROUGHPUT = 2000 (kbps)
    publisher_priority = 2

PUBLISH track=(video, 480p)
  Track Properties:
    THROUGHPUT = 800 (kbps)
    publisher_priority = 3 (lowest quality)
~~~

* **THROUGHPUT**: Bandwidth requirement in kbps
* **publisher_priority**: Relative quality ordering per [MOQT] (lower = higher quality)

### Client Workflow

Client sends a single SUBSCRIBE with track filter:

~~~
Client                                  Relay
  |                                       |
  |  SUBSCRIBE                            |
  |    Track Filter = {                   |
  |      Property Type = THROUGHPUT,      |
  |      MaxTracksSelected = 1            |
  |    }                                  |
  |    SWITCHING-SET-ASSIGNMENT{id=1}     |
  |    SUBSCRIBER_PRIORITY=10             |
  |    DTS-ACTIVATION{id=1, state=1}      |
  |-------------------------------------->|
  |                                       |
  |  (Relay discovers tracks via filter,  |
  |   selects based on THROUGHPUT prop)   |
  |                                       |
~~~

The client specifies:
- **Track Filter**: Selects tracks with THROUGHPUT property, limit to 1 active track
- **SUBSCRIBER_PRIORITY**: For bandwidth allocation across multiple switching sets (lower = higher priority)

### Relay Workflow

1. Relay receives SUBSCRIBE with track filter and DTS parameters.

2. Relay identifies all tracks matching the filter (tracks with THROUGHPUT property).

3. Using publisher-advertised THROUGHPUT values, relay applies the adaptation algorithm
   ({{relay-adaptation-algorithm}}) to select the best track for available bandwidth.

4. Relay uses publisher_priority to break ties (prefer lower value = higher quality).

5. At each group boundary, relay re-evaluates selection based on current bandwidth.

## Example: Adaptive Bitrate Streaming {#example-abr}

A live video stream is encoded at three quality levels. The subscriber wants the relay
to automatically select the best quality based on available bandwidth.

**Publisher Setup**:

~~~
Track: (live-stream, video, 1080p)  ->  5000 kbps, publisher_priority=1
Track: (live-stream, video, 720p)   ->  2000 kbps, publisher_priority=2
Track: (live-stream, video, 480p)   ->   800 kbps, publisher_priority=3
~~~

**Using Approach 1 (Explicit SUBSCRIBE)**:

~~~
SUBSCRIBE (live-stream, video, 1080p)
  SWITCHING-SET-ASSIGNMENT{id=1, throughput=5000, timeout=100}
  SUBSCRIBER_PRIORITY=10

SUBSCRIBE (live-stream, video, 720p)
  SWITCHING-SET-ASSIGNMENT{id=1, throughput=2000, timeout=100}
  SUBSCRIBER_PRIORITY=10

SUBSCRIBE (live-stream, video, 480p)
  SWITCHING-SET-ASSIGNMENT{id=1, throughput=800, timeout=100}
  SUBSCRIBER_PRIORITY=10
  DTS-ACTIVATION{id=1, state=1}
~~~

**Using Approach 2 (Track Filter)**:

~~~
SUBSCRIBE
  Track Filter = {Property=THROUGHPUT, MaxTracksSelected=1}
  SWITCHING-SET-ASSIGNMENT{id=1}
  SUBSCRIBER_PRIORITY=10
  DTS-ACTIVATION{id=1, state=1}
~~~

**Relay Behavior** (both approaches):

With estimated bandwidth of 3 Mbps, relay applies {{relay-adaptation-algorithm}}:

1. Evaluate renditions: 5000 > 3000 (skip), 2000 <= 3000 (select), 800 <= 3000 (candidate)
2. Select highest quality that fits: 720p @ 2000 kbps
3. Forward 720p groups, discard 1080p and 480p groups

~~~
+---------------------------------------------------------------------+
|                  ABR: Bandwidth = 3 Mbps                            |
+---------------------------------------------------------------------+
|                                                                     |
|  Available Renditions:                                              |
|  +----------+  +----------+  +----------+                           |
|  |  1080p   |  |   720p   |  |   480p   |                           |
|  | 5000kbps |  | 2000kbps |  |  800kbps |                           |
|  |  prio=1  |  |  prio=2  |  |  prio=3  |                           |
|  +----+-----+  +----+-----+  +----+-----+                           |
|       |             |             |                                 |
|       X        [SELECTED]         |                                 |
|  (exceeds BW)       |        (lower quality)                        |
|                     v                                               |
|              +------------+                                         |
|              | Forward    |                                         |
|              | 720p@2Mbps |                                         |
|              +------------+                                         |
|                                                                     |
+---------------------------------------------------------------------+
~~~

# Multiple Switching Sets {#multiple-switching-sets}

Multiple switching sets are used when a subscriber receives several independent streams,
each with its own quality renditions. The relay must allocate bandwidth across all sets
based on their subscriber_priority values (lower = higher priority per [MOQT] Section 9.3.5).

## Using SUBSCRIBE_NAMESPACE with Track Filtering

For multiple switching sets, SUBSCRIBE_NAMESPACE with track filters provides a scalable
approach. Each namespace subscription represents one switching set.

### Publisher Track Properties

Publishers advertise properties that enable both quality selection within a set and
priority ordering across sets. The publisher_priority property per [MOQT] indicates
relative quality (lower value = higher quality):

~~~
PUBLISH track=(conf, alice, video, 1080p)
  THROUGHPUT = 2000, publisher_priority = 1

PUBLISH track=(conf, alice, video, 720p)
  THROUGHPUT = 800, publisher_priority = 2

PUBLISH track=(conf, alice, video, 360p)
  THROUGHPUT = 400, publisher_priority = 3
~~~

### Client Workflow

Client sends SUBSCRIBE_NAMESPACE for each participant with subscriber_priority indicating
relative importance (lower value = higher priority):

~~~
SUBSCRIBE_NAMESPACE (conf, alice, video)
  Track Filter = {Property=THROUGHPUT, MaxTracksSelected=1}
  SWITCHING-SET-ASSIGNMENT{id=1}
  SUBSCRIBER_PRIORITY=1   // Active speaker (highest priority)

SUBSCRIBE_NAMESPACE (conf, bob, video)
  Track Filter = {Property=THROUGHPUT, MaxTracksSelected=1}
  SWITCHING-SET-ASSIGNMENT{id=2}
  SUBSCRIBER_PRIORITY=100

SUBSCRIBE_NAMESPACE (conf, carol, video)
  Track Filter = {Property=THROUGHPUT, MaxTracksSelected=1}
  SWITCHING-SET-ASSIGNMENT{id=3}
  SUBSCRIBER_PRIORITY=100

SUBSCRIBE_NAMESPACE (conf, dave, video)
  Track Filter = {Property=THROUGHPUT, MaxTracksSelected=1}
  SWITCHING-SET-ASSIGNMENT{id=4}
  SUBSCRIBER_PRIORITY=100
  DTS-ACTIVATION{id=0, state=1}  // Activate all sets
~~~

### Updating Priorities

When conditions change (e.g., active speaker changes), client sends REQUEST_UPDATE
to adjust subscriber_priority:

~~~
REQUEST_UPDATE (subscription for alice)
  SUBSCRIBER_PRIORITY=100   // Alice no longer speaking

REQUEST_UPDATE (subscription for bob)
  SUBSCRIBER_PRIORITY=1     // Bob is now active speaker
~~~

### Relay Workflow

1. Relay maintains multiple switching sets, each with its subscriber_priority.

2. At each group boundary, relay applies the adaptation algorithm
   ({{relay-adaptation-algorithm}}) to allocate bandwidth across all sets.

3. For each set, relay selects the best rendition using publisher_priority property.

4. Relay responds to subscriber_priority updates by re-running allocation at next group boundary.

## Example: Video Conferencing

### 2x2 Grid Layout

Four participants displayed equally. All have equal subscriber_priority.

**Configuration**:

~~~
Participants: Alice, Bob, Carol, Dave
Each has renditions: 1080p (2000kbps), 720p (800kbps), 360p (400kbps)
subscriber_priority: All equal at 100 (equal priority)
Available bandwidth: 6 Mbps
~~~

**Bandwidth Allocation** (see {{relay-adaptation-algorithm}}):

~~~
+----------------------------------------------------------------------+
|                    2x2 Grid: 6 Mbps Available                        |
+----------------------------------------------------------------------+
|                                                                      |
|  Equal priority allocation: 6000 / 4 = 1500 kbps each                |
|                                                                      |
|  +---------------+  +---------------+  +---------------+  +--------+ |
|  |    Alice      |  |     Bob       |  |    Carol      |  |  Dave  | |
|  |   sp=100      |  |    sp=100     |  |    sp=100     |  | sp=100 | |
|  |  1500 kbps    |  |   1500 kbps   |  |   1500 kbps   |  |1500kbps| |
|  |  -> 720p      |  |   -> 720p     |  |   -> 720p     |  | -> 720p| |
|  |    (800)      |  |     (800)     |  |     (800)     |  |  (800) | |
|  +---------------+  +---------------+  +---------------+  +--------+ |
|                                                                      |
|  (sp = subscriber_priority, equal values = equal bandwidth share)    |
|  Total used: 3200 kbps    Headroom: 2800 kbps                        |
|  After redistribution: All remain at 720p (no upgrade fits)          |
|                                                                      |
|  +--------------+  +--------------+                                  |
|  |   Alice      |  |    Bob       |                                  |
|  |   720p       |  |    720p      |                                  |
|  +--------------+  +--------------+                                  |
|  +--------------+  +--------------+                                  |
|  |   Carol      |  |    Dave      |                                  |
|  |   720p       |  |    720p      |                                  |
|  +--------------+  +--------------+                                  |
|                                                                      |
+----------------------------------------------------------------------+
~~~

### Top-4 with Active Speaker

Active speaker gets priority, others share remaining bandwidth.

**Configuration**:

~~~
Alice (active speaker): subscriber_priority = 1 (highest priority)
Bob, Carol, Dave: subscriber_priority = 100 each
Available bandwidth: 6 Mbps
~~~

**Bandwidth Allocation**:

The relay allocates bandwidth inversely proportional to subscriber_priority
(lower priority value = higher bandwidth share):

~~~
+----------------------------------------------------------------------+
|                 Top-4 Active Speaker: 6 Mbps                         |
+----------------------------------------------------------------------+
|                                                                      |
|  Priority-based allocation (lower sp = more bandwidth)               |
|                                                                      |
|  +---------------------------+  +--------+  +--------+  +--------+   |
|  |         Alice             |  |  Bob   |  | Carol  |  |  Dave  |   |
|  |        sp=1               |  |  sp=100|  |  sp=100|  |  sp=100|   |
|  |   ~52% = 3158 kbps        |  |947 kbps|  |947 kbps|  |947 kbps|   |
|  |      -> 1080p (2000)      |  |->720p  |  |->720p  |  |->720p  |   |
|  +---------------------------+  +--------+  +--------+  +--------+   |
|                                                                      |
|  Total: 2000 + 800 + 800 + 800 = 4400 kbps                           |
|                                                                      |
|  +-----------------------------+  +----------+  +----------+         |
|  |                             |  |          |  |          |         |
|  |     Alice (1080p)           |  |Bob (720p)|  |Carol     |         |
|  |     Active Speaker          |  |          |  |(720p)    |         |
|  |                             |  +----------+  +----------+         |
|  |                             |  +----------+                       |
|  |                             |  |Dave      |                       |
|  |                             |  |(720p)    |                       |
|  +-----------------------------+  +----------+                       |
|                                                                      |
+----------------------------------------------------------------------+
~~~

### DTS with Bandwidth Reduction

Same configuration, but bandwidth drops to 3 Mbps.

**Bandwidth Allocation**:

~~~
+----------------------------------------------------------------------+
|              Active Speaker: Bandwidth Drops to 3 Mbps               |
+----------------------------------------------------------------------+
|                                                                      |
|  BEFORE (6 Mbps):  Alice@1080p, Bob@720p, Carol@720p, Dave@720p      |
|                                                                      |
|                         | BW DROP |                                  |
|                         v         v                                  |
|                                                                      |
|  AFTER (3 Mbps):                                                     |
|                                                                      |
|  +---------------------------+  +--------+  +--------+  +--------+   |
|  |         Alice             |  |  Bob   |  | Carol  |  |  Dave  |   |
|  |        w=100              |  |  w=30  |  |  w=30  |  |  w=30  |   |
|  |   52.6% = 1579 kbps       |  |474 kbps|  |474 kbps|  |474 kbps|   |
|  |      -> 720p (800)        |  |->360p  |  |->360p  |  |->360p  |   |
|  +---------------------------+  +--------+  +--------+  +--------+   |
|                                                                      |
|  Total: 800 + 400 + 400 + 400 = 2000 kbps                            |
|                                                                      |
|  Priority preserved: Alice (720p) > Bob/Carol/Dave (360p)            |
|                                                                      |
+----------------------------------------------------------------------+
~~~

## Example: Cloud Gaming (Non-Conferencing)

Cloud gaming with adaptive game video and high-priority HUD overlay.

**Configuration**:

~~~
Game video: 4 renditions (8000/4000/2000/1000 kbps), subscriber_priority = 100
HUD overlay: 1 rendition (200 kbps), subscriber_priority = 1 (high priority)
Available bandwidth: 10 Mbps
~~~

**Bandwidth Allocation**:

~~~
+----------------------------------------------------------------------+
|                    Cloud Gaming: 10 Mbps                             |
+----------------------------------------------------------------------+
|                                                                      |
|  Priority allocation (lower subscriber_priority = higher priority):  |
|  HUD (priority=1): allocated first -> uses 200 kbps                  |
|  Game (priority=100): remaining -> 9800 kbps available               |
|                                                                      |
|  Game selects: 8000 kbps (4K)                                        |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |  +----------------------------------------------------------+  |  |
|  |  |                                                          |  |  |
|  |  |              GAME VIDEO (4K @ 8000 kbps)                  |  |  |
|  |  |                                                          |  |  |
|  |  +----------------------------------------------------------+  |  |
|  |  +-----------------+                    +-----------------+    |  |
|  |  | Health: ####... |  HUD (200 kbps)    |  Ammo: 30/120   |    |  |
|  |  +-----------------+  sub_priority=1    +-----------------+    |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
|  At 5 Mbps: HUD stays 200 kbps, Game drops to 4000 kbps (1080p)      |
|                                                                      |
+----------------------------------------------------------------------+
~~~

The low subscriber_priority (1) ensures HUD is prioritized. Since HUD has only one
rendition, it uses minimal bandwidth and remaining capacity goes to game video.

# Relay Adaptation Algorithm {#relay-adaptation-algorithm}

This section defines the relay adaptation algorithm for Dynamic Track Switching. The algorithm
is agnostic to whether there is a single switching set or multiple switching sets—it handles
both cases uniformly through priority-based bandwidth allocation using subscriber_priority.

## Algorithm Overview

The relay executes this algorithm at each group boundary or when bandwidth estimates change:

1. **Measure available bandwidth**: Maintain an estimate of downstream bandwidth capacity, B_total.

2. **Calculate priority-based allocation**: Order switching sets by subscriber_priority
   (lower value = higher priority). Allocate bandwidth starting with highest priority sets.
   For sets with equal priority, divide bandwidth equally among them.

   For a single switching set, it receives B_total.

3. **Select rendition per switching set**: Within each set, select the highest-quality
   rendition with throughput requirement <= B_i. If no rendition fits, the set receives
   no bandwidth for this group.

4. **Redistribute unused bandwidth**: If a set cannot use its full allocation (e.g.,
   selected rendition < B_i, or no rendition fits), redistribute unused bandwidth
   proportionally to other sets and re-evaluate their rendition selection.

5. **Forward selected groups**: Forward groups from the selected rendition of each set.

~~~
+-----------------------------------------------------------------------------+
|                     Relay Adaptation Algorithm                              |
+-----------------------------------------------------------------------------+
                                    |
                                    v
                    +-------------------------------+
                    |  1. Measure B_total           |
                    |     (downstream bandwidth)    |
                    +---------------+---------------+
                                    |
                                    v
                    +-------------------------------+
                    |  2. Calculate allocation      |
                    |     per switching set         |
                    |     (order by sub_priority,   |
                    |      lower = higher priority) |
                    +---------------+---------------+
                                    |
                                    v
                    +-------------------------------+
                    |  3. Select rendition per set  |
                    |     Pick highest quality      |
                    |     where throughput <= B_i   |
                    +---------------+---------------+
                                    |
                                    v
                    +-------------------------------+
                    |  4. Redistribute unused BW    |
                    |     Reallocate proportionally |
                    |     Re-evaluate selections    |
                    +---------------+---------------+
                                    |
                                    v
                    +-------------------------------+
                    |  5. Forward selected groups   |
                    +---------------+---------------+
                                    |
                                    v
                         +---------------------+
                         |  Wait for next      |
                         |  group boundary or  |<--------+
                         |  BW change          |         |
                         +----------+----------+         |
                                    |                    |
                                    +--------------------+
~~~

## Single vs Multiple Switching Sets

The algorithm handles both cases identically:

**Single Switching Set** (e.g., ABR):
- One set receives B_total
- Select best rendition that fits in B_total

**Multiple Switching Sets** (e.g., video conferencing):
- N sets with subscriber_priority values p_1, p_2, ..., p_N
- Sets ordered by priority (lower value = higher priority)
- Higher priority sets receive bandwidth allocation first
- Redistribution ensures efficient bandwidth utilization

## High-Priority Single-Rendition Streams

Streams with only one rendition (e.g., HUD, telemetry) use the same algorithm:
- Assign low subscriber_priority value (high priority) to ensure bandwidth allocation
- The single rendition either fits or doesn't
- Remaining bandwidth is allocated to lower priority sets

This approach avoids special "guaranteed bandwidth" handling while still prioritizing
critical streams through the subscriber_priority mechanism.

## Bandwidth Estimation

The relay SHOULD maintain a bandwidth estimate that:
- Is smoothed to avoid oscillation from transient measurements
- Accounts for the maximum group duration of tracks being switched
- Is updated at least once per group boundary
- May use QUIC congestion control signals when available

## Selection Timer

When the preferred track's group has not arrived but a lower-priority track's group has:

1. Relay caches the lower-priority group
2. Starts selection timer based on the preferred track's Selection time limit
3. If preferred group arrives before timeout, forward it
4. If timeout expires, remove preferred track from candidates and re-select

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
sets (one per participant) and assigns subscriber_priority values to indicate importance—for
example, giving the active speaker a lower subscriber_priority (higher priority) than other
participants. When the active speaker changes, the subscriber adjusts priority values. The
relay allocates its forwarding capacity across all switching sets according to the
subscriber-indicated priorities, selecting appropriate quality levels for each participant
stream to fit within the total available bandwidth.

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
│     - Publisher-indicated rendition priorities                               │
│     - Total available bandwidth                                              │
│     - Subscriber-indicated priorities (subscriber_priority)                  │
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
quality (throughput) for the screen share to ensure text remains readable. The relay manages bandwidth allocation between the two content types,
degrading camera video quality before reducing screen share quality when bandwidth becomes constrained.

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
a lower subscriber_priority (higher priority) to the gaze tile and higher subscriber_priority
values to peripheral tiles. The relay responds rapidly to these updates, reallocating bandwidth
to deliver high quality for the gaze tile while maintaining lower quality for surrounding tiles.

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
levels and publishes a stats overlay as a high-priority stream. All streams share
a common time reference for synchronization. The end subscriber subscribes to desired
camera angles and the stats overlay, assigning subscriber_priority values to indicate which
views are most important (lower value = higher priority). During highlights or replays,
the subscriber dynamically adjusts priorities to shift focus to the relevant camera. The
relay allocates bandwidth according to subscriber priorities, maintains temporal sync across
all forwarded streams, and responds promptly to priority changes.

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
