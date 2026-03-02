---
title: "ATOM: AT Protocol Over MoQ Transport"
abbrev: "ATOM"
category: std

docname: draft-nandakumar-atproto-atom-latest
submissiontype: IETF
number:
date: 2026-02-28
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Media Over QUIC"
keyword:
 - authenticated transfer
 - AT Protocol
 - MOQT
 - decentralized social
 - firehose

venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "snandaku/draft-nandakumar-atproto-atom"
  latest: "https://snandaku.github.io/draft-nandakumar-atproto-atom/draft-nandakumar-atproto-atom.html"

author:
 -
    name: "Suhas Nandakumar"
    organization: "Cisco"
    email: "snandaku@cisco.com"
 -
    name: "Cullen Jennings"
    organization: "Cisco Systems"
    email: "fluffy@cisco.com"

normative:
  RFC9000:
  RFC8446:
  MoQ-TRANSPORT:
    title: "Media over QUIC Transport"
    date: 2026-01
    author:
    - ins: L. Curley
    - ins: K. Pugin
    - ins: S. Nandakumar
    - ins: V. Vasiliev
    - ins: I. Swett
    seriesinfo:
      Internet-Draft: draft-ietf-moq-transport-16
    target: https://datatracker.ietf.org/doc/draft-ietf-moq-transport/
  AT-ARCH:
    title: "Authenticated Transfer (AT) Protocol Architecture"
    date: 2025
    author:
    - ins: B. Newbold
    - ins: D. Holmgren
    seriesinfo:
      Internet-Draft: draft-newbold-at-architecture
    target: https://datatracker.ietf.org/doc/draft-newbold-at-architecture/
  AT-REPO:
    title: "Authenticated Transfer (AT) Repository and Synchronization"
    date: 2025
    author:
    - ins: D. Holmgren
    - ins: B. Newbold
    seriesinfo:
      Internet-Draft: draft-holmgren-at-repository
    target: https://datatracker.ietf.org/doc/draft-holmgren-at-repository/

informative:
  RFC7540:
  RFC8949:
  WebTransport:
    title: "The WebTransport Protocol Framework"
    date: 2023-08
    author:
    - ins: V. Vasiliev
    seriesinfo:
      RFC: 9297
    target: https://www.rfc-editor.org/rfc/rfc9297
  MoQ-C4M:
    title: "Common Access Token for Media over QUIC"
    date: 2025
    author:
    - ins: S. Jennings
    seriesinfo:
      Internet-Draft: draft-ietf-moq-c4m
    target: https://datatracker.ietf.org/doc/draft-ietf-moq-c4m/

--- abstract

This document specifies how the Authenticated Transfer (AT) Protocol
can leverage Media over QUIC Transport (MOQT) for efficient data
synchronization across decentralized social networks. The AT Protocol's
firehose event stream and repository synchronization mechanisms map
naturally to MOQT's publish/subscribe model, enabling scalable relay
infrastructure, priority-based delivery, and improved resilience for
large-scale social data distribution.

This specification addresses the challenges of the current WebSocket-based
transport and demonstrates how MOQT's relay architecture, group-based
caching, and multiplexed streams provide significant benefits for
AT Protocol deployments at scale.

--- middle

# Introduction

The Authenticated Transfer (AT) Protocol {{AT-ARCH}} provides a framework
for decentralized social web applications using self-certifying data
repositories. The protocol enables users to maintain control over their
data while participating in a federated network of Personal Data Servers
(PDS), relays, and application views (AppViews).

Currently, AT Protocol relies on two primary transport mechanisms for
data synchronization:

* **HTTP**: Used for batch repository exports via CAR (Content Addressable
  Archive) files
* **WebSocket**: Used for real-time event streams (firehose) with
  CBOR-encoded messages

While functional, these transport mechanisms face significant challenges
at scale, particularly for the firehose event stream that must distribute
updates across a network with millions of users generating billions of
records.

This document defines how Media over QUIC Transport (MOQT) {{MoQ-TRANSPORT}}
can serve as an enhanced transport for AT Protocol. MOQT enables low-latency
delivery while relay caching supports multiple latency spectrums from
real-time to eventual consistency. The protocol's native publish/subscribe
model, relay infrastructure, and priority-based delivery address current
scalability and resilience challenges.

## Current AT Protocol Architecture

~~~
AT Protocol Network Architecture:

                           ┌─────────────────┐
                           │   AppView       │
                           │  (Indexer)      │
                           │                 │
                           └────────▲────────┘
                                    │
                              Firehose (WS)
                                    │
                           ┌────────┴────────┐
                           │     Relay       │
                           │  (Aggregator)   │
                           └────────▲────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
       ┌──────┴──────┐       ┌──────┴──────┐       ┌──────┴──────┐
       │    PDS A    │       │    PDS B    │       │    PDS C    │
       │  (Hosts)    │       │  (Hosts)    │       │  (Hosts)    │
       │ - User 1    │       │ - User 2    │       │ - User 3    │
       │ - User 2    │       │ - User 5    │       │ - User 6    │
       └─────────────┘       └─────────────┘       └─────────────┘
~~~

The current architecture comprises:

Personal Data Server (PDS):
: Hosts user accounts and their data repositories. Each PDS provides
  a firehose endpoint for real-time updates and HTTP endpoints for
  repository exports.

Relay:
: Aggregates firehose streams from multiple PDS instances into a
  unified event stream. Full-network relays attempt to include all
  PDS instances in the network.

AppView:
: Application-specific indexing services that consume the firehose
  to build aggregated views (e.g., feeds, search, analytics).

## Requirements Language

{::boilerplate bcp14-tagged}

# Challenges with Current Transport

The current WebSocket-based transport for AT Protocol faces several
challenges that impact scalability, reliability, and operational
efficiency.

## Scalability Limitations

### Connection Overhead

Each subscriber to a PDS or relay firehose requires a dedicated
WebSocket connection. At network scale, this creates:

* Connection state overhead on servers hosting popular content
* TCP head-of-line blocking affecting all events on a connection
* Limited ability to prioritize critical events during congestion

### Message Size Constraints

The current protocol specifies a hard maximum of 5 MBytes per WebSocket
frame. While this accommodates most operations, it creates challenges for:

* Large repository commits (limited to 2MB blocks, 200 operations)
* Efficient batching of small updates
* Variable-size content distribution

### Single-Stream Bottleneck

~~~
Current Firehose: Single Stream for All Event Types

WebSocket Connection
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  #commit │ #commit │ #identity │ #commit │ #account │ #commit   │
│  (big)   │ (small) │ (urgent)  │ (med)   │ (urgent) │ (big)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    Head-of-line blocking:
                    Urgent #identity event must wait
                    for large #commit to complete
~~~

All event types flow through a single WebSocket stream without
differentiation, meaning:

* High-priority identity changes wait behind large commit events
* Account status updates cannot preempt in-progress transfers
* No mechanism for subscribers to filter by event type at transport layer

## Reliability Challenges

### Cursor-Based Replay Limitations

The current firehose relies on monotonic cursor values for replay after
disconnection. This approach requires consumers to track cursor state
externally, since the protocol provides no mechanism for server-side
cursor persistence. Gap detection depends entirely on commit chain
validation, which is computationally expensive and requires maintaining
local state. When gaps are detected, recovery requires fetching the
complete repository, creating a thundering herd risk when many
consumers reconnect simultaneously after an outage.

### No Native Late-Join Support

When a new consumer connects to a firehose, it must either start from
the current position and miss all prior events, or replay from a
cursor to catch up. There is no concept of "recent state" that would
allow quick synchronization to a meaningful starting point. Consumers
cannot receive cached recent events without initiating a full replay
sequence, which increases latency for new subscribers and places
additional load on the server.

### Connection Fragility

WebSocket connections over TCP are susceptible to network path changes
that cause connection drops. The protocol provides no mechanism for
connection migration when a client's IP address changes, such as when
moving between networks on a mobile device. Any network transition
requires establishing a completely new connection and resuming from
the last known cursor position.

## Operational Challenges

### Relay Infrastructure Complexity

Current relays face significant operational burden in maintaining the
firehose infrastructure. Each relay must establish and maintain persistent
WebSocket connections to all upstream PDS instances it aggregates, scaling
linearly with the number of sources. When connections fail, relays must
handle reconnection logic and cursor tracking independently for each
upstream source. Custom caching and replay logic must be implemented at
the application layer, as the transport provides no native support for
these operations. Additionally, relays must build proprietary distribution
mechanisms to serve their own downstream subscribers, duplicating effort
across the ecosystem.

### Limited Quality of Service

The current transport provides no native mechanism for quality of service
differentiation. All event types receive equal treatment regardless of
their operational importance, so critical account status changes compete
with routine content updates for bandwidth. Rate limiting based on consumer
capacity must be implemented entirely at the application layer, with no
transport-level flow control. Graceful degradation during overload
conditions is difficult to achieve, often resulting in connection failures
rather than controlled service reduction.

# MOQT Transport Benefits

MOQT addresses the challenges identified above through its native
architecture designed for large-scale media distribution.

## Native Publish/Subscribe Model

MOQT's publish/subscribe model aligns naturally with AT Protocol's
firehose semantics. Subscribers select specific event tracks of interest
while publishers announce available tracks via PUBLISH_NAMESPACE. Since
relays handle distribution, no dedicated connection is required per
subscriber, and track-level subscription granularity reduces unnecessary
traffic.

PUBLISH_NAMESPACE enables dynamic discovery of available content. When
a PDS comes online, it announces its namespaces to connected relays,
advertising firehose tracks, repository sync tracks, and blob stores.
Relays learn about new PDS instances automatically without static
configuration, enabling the network to scale organically as new servers
join.

SUBSCRIBE_NAMESPACE allows subscribers to discover available tracks
using prefix-based queries. A full-network relay can subscribe to
at/firehose/ to discover all PDS instances publishing firehose events,
then selectively subscribe to each. An AppView can query a relay's
namespace to enumerate available event types before subscribing to
specific tracks of interest. This discovery mechanism eliminates the
need for out-of-band coordination when adding new publishers or
subscribers to the network.

A PDS advertises its available namespaces via PUBLISH_NAMESPACE:

| Namespace | Description |
|-----------|-------------|
| `at/firehose/pds.example.com` | Firehose events from this PDS |
| `at/repo/pds.example.com` | Repository sync tracks |
| `at/blobs/pds.example.com` | Blob storage |

A relay discovering PDS instances issues SUBSCRIBE_NAMESPACE with
`at/firehose/` and receives a NAMESPACE response listing all matching
hosts (potentially thousands of PDS instances).

## Priority-Based Delivery

MOQT's 0-255 priority scale enables differentiated event delivery,
ensuring that critical account takedowns (priority 0-15) and urgent
identity changes such as key rotations (priority 16-31) are delivered
ahead of routine commit events (priority 96-191). This prevents large
content transfers from blocking time-sensitive operations.

| AT Event Type | MOQT Priority | Rationale |
|---------------|---------------|-----------|
| #account (takedown) | 0-15 | Critical - affects content availability |
| #identity | 16-31 | Urgent - key rotation, handle changes |
| #account (status) | 32-63 | Important - account state changes |
| #sync | 64-95 | Moderate - state reset events |
| #commit (small) | 96-127 | Normal - typical record operations |
| #commit (large) | 128-191 | Background - bulk content updates |
| Repository export | 192-255 | Bulk - full sync operations |


## Relay Infrastructure

MOQT relays provide purpose-built infrastructure for AT Protocol
distribution. Hierarchical relay topologies reduce origin load by
distributing cached content from regional nodes, while geographic
placement optimizes latency for subscribers. Because the relay protocol
is standardized, operators need not implement custom distribution logic.

~~~
MOQT Relay Network for AT Protocol:

                    ┌─────────────────┐
                    │  Global Relay   │
                    │  (Federation)   │◄─── Cross-region aggregation
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Regional    │   │ Regional    │   │ Regional    │
    │ Relay A     │   │ Relay B     │   │ Relay C     │
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           │                 │                 │
     ┌─────┴─────┐     ┌─────┴─────┐     ┌─────┴─────┐
     │           │     │           │     │           │
   PDS 1      PDS 2  PDS 3      PDS 4  PDS 5      PDS 6
     ▲           ▲     ▲           ▲     ▲           ▲
     │           │     │           │     │           │
   Users       Users Users       Users Users       Users
~~~

## Group-Based Organization and Caching

MOQT groups enable efficient caching and late-join support. Late-joining
subscribers receive the most recent group immediately rather than
replaying from the beginning, and relay caches at each tier reduce
upstream load. Group boundaries serve as natural replay points, with
cursor semantics mapping directly to Group ID and Object ID.

~~~
Group Hierarchy:  [G0: events 0-99] [G1: events 100-199] ... [GN: live]
                         ↓                 ↓                    ↓
Relay Cache:          cached            cached              caching
~~~

### Late-Join via SUBSCRIBE

MOQT's SUBSCRIBE message includes filter parameters that control the
starting position for delivery. As defined in {{MoQ-TRANSPORT}}, these
filters include LatestGroup (start from the newest available group),
LatestObject (start from the most recent object), AbsoluteStart
(begin at a specific group/object position), and AbsoluteRange
(request a bounded range). These filters allow subscribers to join
at any point and then continue receiving live and future content.

SUBSCRIBE delivers content from the specified starting position
forward, keeping subscribers at the live edge as new content arrives.
For historical content retrieval, subscribers use FETCH operations
instead (see {{fetch-for-on-demand-retrieval}}).

This capability supports several AT Protocol scenarios.
A new subscriber joining a track uses LatestGroup to begin
with current activity.

### FETCH for On-Demand Retrieval {#fetch-for-on-demand-retrieval}

MOQT's FETCH operation complements SUBSCRIBE by enabling selective
retrieval of specific objects. While SUBSCRIBE keeps clients at the
live edge, FETCH retrieves historical content
regardless of how far back it exists. FETCH specifies exact group and
object ranges, retrieving precisely the needed data without establishing
a continuous subscription.

When a subscriber detects a gap in received events, it can FETCH the
missing objects directly rather than replaying an entire group. When
verifying a specific record, a client can FETCH just the MST path blocks
needed for cryptographic validation. For deep historical queries, an
analytics service can FETCH specific commit ranges from months prior.

Relay caching amplifies the benefit of FETCH operations. Popular content
such as viral images, high-profile account commits, and frequently
accessed MST nodes remain in relay cache. Subsequent FETCH requests
for this content are served locally, reducing latency and eliminating
load on origin servers. For a viral post with an embedded image,
thousands of FETCH requests for the image blob resolve to a single
origin fetch followed by cache hits at the relay tier.

### Caching Benefits Summary

The combination of SUBSCRIBE late-join and FETCH with relay caching
provides several benefits for AT Protocol deployments:

- Fan-out efficiency: Origin publishes once, relay serves many subscribers
- Late-join speed: New subscribers receive cached history immediately
- Gap recovery: Missing events fetched from relay cache, not origin
- Popular content: High-demand blobs and commits served from cache
- Reduced origin load: Relay tier absorbs read amplification

## QUIC Transport Advantages

Leveraging QUIC {{RFC9000}} provides connection migration so that
firehose subscriptions survive network changes such as IP address
transitions or WiFi-to-cellular handoffs. Multiplexed streams eliminate
head-of-line blocking between logical channels, while mandatory TLS 1.3
{{RFC8446}} ensures built-in encryption. Per-stream and connection-level
flow control prevents subscriber overload, and 0-RTT resumption enables
fast reconnection after brief disconnections.

# MOQT Mapping

This section defines how AT Protocol data maps to MOQT primitives.

## Connection Establishment

AT Protocol endpoints using MOQT establish connections following
the standard MOQT setup procedure:

~~~
AT-over-MOQT Connection Establishment:

PDS/Relay                                            Subscriber
    │                                                     │
    │◄─── QUIC Connection (ALPN: "moqt") ─────────────────│
    │──── QUIC Connection Established ───────────────────▶│
    │                                                     │
    │◄─── MOQT CLIENT_SETUP ──────────────────────────────│
    │     (at-version: 1, supported-events: 0x0F)         │
    │                                                     │
    │──── MOQT SERVER_SETUP ─────────────────────────────▶│
    │     (at-version: 1, relay-capabilities: 0x07)       │
    │                                                     │
    │──── PUBLISH_NAMESPACE ─────────────────────────────▶│
    │     (at/{pds-host}/firehose)                        │
    │                                                     │
    │◄─── SUBSCRIBE_NAMESPACE ────────────────────────────│
    │     (at/{pds-host}/firehose)                        │
    │                                                     │
    │──── NAMESPACE (tracks available) ──────────────────▶│
    │     (commits, identity, account, sync)              │
    │                                                     │
    │◄─── SUBSCRIBE (commits) ────────────────────────────│
    │                                                     │
    │──── SUBSCRIBE_OK ──────────────────────────────────▶│
    │                                                     │
    │═════ Firehose Event Stream ════════════════════════▶│
~~~

Setup parameters for AT Protocol:

| Parameter | ID | Type | Description |
|-----------|----|------|-------------|
| at-version | 0x41540001 | varint | AT Protocol version |
| at-supported-events | 0x41540002 | varint | Bitmask of supported event types |
| at-relay-caps | 0x41540003 | varint | Relay capability flags |

Event type bitmask values: 0x01 (#commit), 0x02 (#identity), 0x04 (#account), 0x08 (#sync).

## Track Types

ATOM defines three track types for different use cases:

### Firehose Tracks

Firehose tracks carry real-time event streams from PDS hosts.

Namespace structure: `at/firehose/{host}/{event-type} -- {did|all}`

| Track | Namespace | Content | Priority |
|-------|-----------|---------|----------|
| Commits | `at/firehose/{host}/commits` | Repository updates | 96-191 |
| Identity | `at/firehose/{host}/identity` | DID/handle changes | 16-31 |
| Account | `at/firehose/{host}/account` | Hosting status | 0-63 |
| Sync | `at/firehose/{host}/sync` | State assertions | 64-95 |

Groups organize events by sequence number (e.g., 1000 events per group).
Track name is either a specific DID or "all" for aggregated streams.

### Repository Sync Tracks

Repository sync tracks provide full block-level repository data.

Namespace: `at/repo/{host}/{did}/sync`

The MOQT hierarchy maps to AT Protocol repository concepts:

| MOQT Level | AT Protocol Concept | Purpose |
|------------|---------------------|---------|
| Track | Repository (DID) | Subscription unit |
| Group | Commit/Revision | Temporal progression |
| Subgroup | Collection + MST path | Structural organization |
| Object | Record/Block | Individual data items |

Each group represents a commit. Subgroup 0 contains the signed commit
and MST structure; subsequent subgroups contain collections with MST
proof paths for independent verification.

| Subgroup | Contents |
|----------|----------|
| 0 | Commit block + full MST (verification anchor) |
| 1+ | Collection records + MST proof path |

Objects within a group are ordered: signed commit (ObjectID 0), MST root,
MST intermediate nodes, then records. The at-block-cid extension header
carries the CID for integrity verification.

~~~
MST Structure and MOQT Mapping:

Repository (DID: did:plc:abc123)
│
├── Commit (rev: 3kf7x2...)          ──────────────▶ Group N
│   │
│   ├── Signed Commit Block          ──────────────▶ Subgroup 0, Object 0
│   │
│   └── MST (Merkle Search Tree)
│       │
│       ├── Root Node (CID: bafyr...)──────────────▶ Subgroup 0, Object 1
│       │   ├── key: "app.bsky.feed.post/3k..."
│       │   └── children: [L, R]
│       │
│       ├── Left Node               ───────────────▶ Subgroup 0, Object 2
│       │   └── key: "app.bsky.actor.profile/self"
│       │
│       └── Right Node              ───────────────▶ Subgroup 0, Object 3
│           └── key: "app.bsky.graph.follow/3k..."
│
├── Collection: app.bsky.feed.post   ──────────────▶ Subgroup 1
│   ├── Record 3k7abc... + MST proof ──────────────▶ Object 0
│   └── Record 3k7def... + MST proof ──────────────▶ Object 1
│
├── Collection: app.bsky.actor.profile ────────────▶ Subgroup 2
│   └── Record self + MST proof      ──────────────▶ Object 0
│
└── Collection: app.bsky.graph.follow ─────────────▶ Subgroup 3
    └── Record 3k9xyz... + MST proof ──────────────▶ Object 0
~~~

The MST proof path included with each record contains the minimal set
of tree nodes required to verify that record's inclusion in the commit.
This enables independent verification of individual records without
downloading the entire repository.

### Blob Store Track

Blob store tracks enable cross-repository deduplication of large media.

Namespace: `at/blobs/{host}`

The Group ID is derived from the CID (first 8 bytes of multihash as uint64),
Object ID is always 0. This deterministic mapping ensures identical blobs
from different repositories resolve to the same MOQT location.

## Object Structure

AT Protocol events are encapsulated in MOQT objects:

~~~
MOQT Object Structure:

┌─────────────────────────────────────────────────────────────┐
│ Extension Headers:                                          │
│   at-event-type (0x41544501): varint                        │
│   at-repo-did (0x41544502): string                          │
│   at-repo-rev (0x41544503): string (TID)                    │
│   at-seq (0x41544504): varint (cursor equivalent)           │
│   at-block-cid (0x41544505): bytes (for repo sync blocks)   │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│ Payload: CBOR-encoded event (compatible with AT-REPO)       │
└─────────────────────────────────────────────────────────────┘
~~~

## Subscription and Filtering

### Firehose Subscription

Subscribers specify start position and filtering:

* `StartGroup=Latest`: Begin with live events (default)
* `StartGroup=N`: Resume from specific cursor position
* Track name `all` or specific DID for filtering

### Repository Sync

Subgroup filtering enables selective synchronization:

~~~
Selective Sync Examples:

Full repository:     SubgroupFilter: [0, 1, 2, 3, ...]
Posts only:          SubgroupFilter: [0, 1]  (~15% bandwidth)
Verification only:   SubgroupFilter: [0]     (~2% bandwidth)
~~~

Temporal navigation with groups:

* `Subscribe StartGroup=LATEST`: Current state + live updates
* `Fetch StartGroup=N, EndGroup=M`: Historical range retrieval

### Late-Join and Catch-Up

New subscribers request `StartGroup=LATEST` to receive current state
immediately. After disconnection, subscribers use `Fetch` with their
last known group to retrieve missed commits before resuming live
subscription.

## Priority and Parallelism

MOQT priority enables efficient delivery ordering:

| Block Type | Priority | Rationale |
|------------|----------|-----------|
| Account takedown | 0-15 | Safety-critical |
| Identity changes | 16-31 | Affects routing |
| Signed Commit | 0-15 | Required for verification |
| MST Root | 16-31 | Unlocks tree traversal |
| MST Nodes | 32-63 | Enables record verification |
| Small Records | 64-127 | Core content |
| Large Records | 128-191 | Less urgent |
| Blobs | 192-255 | Media, lazy-loadable |

Content-addressable blocks enable parallel fetching across QUIC streams,
with total transfer time bounded by the slowest stream rather than the
sum of all transfers.

## Relay Aggregation

MOQT relays aggregate firehose streams from multiple upstream sources:

~~~
Relay Aggregation:

                    Full-Network Relay
                   ┌────────────────────┐
                   │  at/firehose/      │
                   │  relay.network/*   │
                   └─────────▲──────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
 ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
 │ Regional    │      │ Regional    │      │ Regional    │
 │ Relay       │      │ Relay       │      │ Relay       │
 └──────▲──────┘      └──────▲──────┘      └──────▲──────┘
        │                    │                    │
      PDS hosts            PDS hosts            PDS hosts
~~~

Relays perform subscription aggregation, namespace republishing,
event caching for late-join, and deduplication.

# Reliability and Recovery

ATOM leverages MOQT's group-based delivery model and QUIC's transport
features to provide robust reliability mechanisms for AT Protocol
data synchronization. The key capabilities include cursor mapping
that translates AT Protocol sequence numbers to MOQT group/object
positions, gap detection that identifies missing events through
sequential object tracking, and multiple recovery strategies ranging
from targeted object fetches to full repository synchronization.
QUIC's connection migration further ensures that subscribers
maintain continuous delivery even when network conditions change.

## Cursor Mapping

AT Protocol cursors map to MOQT group and object identifiers, enabling
seamless translation between the existing cursor-based replay mechanism
and MOQT's hierarchical addressing. Each firehose track maintains a
mapping table that records the sequence number base for each group,
allowing bidirectional conversion between AT Protocol cursors and
MOQT positions.

When a subscriber needs to resume from a specific cursor position,
it converts the cursor value to the corresponding group and object
identifiers. The conversion uses integer division to determine the
group number and the modulo operation to determine the object offset
within that group. This deterministic mapping ensures that any cursor
value can be precisely located within the MOQT track structure.

~~~
Cursor Translation:

AT Cursor (seq: 12345678)
         │
         ▼
┌────────────────────────────────────────┐
│ Cursor Mapping Table (per track)       │
│                                        │
│ seq_base: 12340000                     │
│ group_size: 1000                       │
│                                        │
│ seq 12345678                           │
│   = seq_base + (group * group_size)    │
│     + object                           │
│   = 12340000 + (5 * 1000) + 678        │
│                                        │
│ MOQT: Group 5, Object 678              │
└────────────────────────────────────────┘
~~~

Publishers MUST include the `at-seq` extension header in each object
to enable accurate cursor reconstruction.

## Gap Detection and Recovery

MOQT's group-based delivery model simplifies gap detection compared
to the current cursor-only approach. This section describes how gaps
are detected and the recovery strategies available to subscribers.

### Detecting Missing Objects

Because objects within a group are sequentially numbered and groups
themselves are ordered, subscribers can immediately identify when
expected objects are missing by comparing received object identifiers
against their expected sequence.

~~~
Gap Detection:

Subscriber State:
  Last received: Group 5, Object 999
  Expected next: Group 6, Object 0

Incoming object: Group 6, Object 5

     Expected                    Received
        │                           │
        ▼                           ▼
   ┌─────────┐                 ┌─────────┐
   │ G6, O0  │  ← Missing! →   │ G6, O5  │
   └─────────┘                 └─────────┘
        │
        ▼
   Gap detected: Objects 0-4 missing from Group 6
~~~

### Recovery: Fetch Missing Objects

The preferred recovery approach is to fetch only the specific missing
objects using MOQT's FETCH operation with precise group and object
boundaries. If relay caches contain the missing data, recovery completes
without any load on the origin server.

~~~
Fetch Missing Objects:

Subscriber                           Relay                    Origin
    │                                  │                        │
    │── FETCH ────────────────────────▶│                        │
    │   StartGroup=6, StartObject=0    │                        │
    │   EndGroup=6, EndObject=4        │                        │
    │                                  │                        │
    │                          [Cache hit?]                     │
    │                              │ Yes                        │
    │◄── Objects 0-4 ──────────────┘                            │
    │    (from cache)                                           │
    │                                                           │
    │    Gap filled, resume live stream                         │
~~~

### Recovery: Fetch Full Group

For larger gaps spanning multiple groups, subscribers may request
entire groups rather than specifying individual object ranges.
This approach is simpler but may retrieve more data than strictly
necessary.

~~~
Fetch Full Group:

Subscriber                           Relay                    Origin
    │                                  │                        │
    │   [Gap spans Groups 6-8]         │                        │
    │                                  │                        │
    │── FETCH ────────────────────────▶│                        │
    │   StartGroup=6, EndGroup=8       │                        │
    │                                  │                        │
    │                          [Partial cache]                  │
    │◄── Group 6 (cached) ─────────────┤                        │
    │◄── Group 7 (cached) ─────────────┤                        │
    │                                  │── FETCH ──────────────▶│
    │                                  │   (Group 8)            │
    │                                  │◄── Group 8 ────────────│
    │◄── Group 8 ──────────────────────┤                        │
    │                                                           │
    │    All groups received, resume live stream                │
~~~

### Recovery: Full Repository Sync

As a fallback when cached data is unavailable or gaps are too
extensive, subscribers can perform a full repository synchronization
using either HTTP CAR exports or the MOQT repository sync track.
This approach validates the entire repository state against a
#sync event.

~~~
Full Repository Sync:

Subscriber                           PDS/Relay
    │                                  │
    │   [Gap too large or cache miss]  │
    │                                  │
    │── GET /xrpc/com.atproto.sync. ──▶│
    │   getRepo?did=...                │
    │                                  │
    │◄── CAR file ─────────────────────│
    │    (complete repository export)  │
    │                                  │
    │   [Validate against #sync event] │
    │                                  │
    │   1. Parse CAR blocks            │
    │   2. Verify MST root matches     │
    │   3. Update local state          │
    │   4. Resume live subscription    │
~~~

## Disconnection Handling

MOQT over QUIC provides superior disconnection handling through
three complementary mechanisms: 0-RTT session resumption, cursor-based
reconnection, and transparent connection migration.

### Brief Disconnections: 0-RTT Resumption

For brief disconnections (typically under 30 seconds), QUIC's 0-RTT
resumption allows subscribers to restore their session with minimal
latency. The subscriber presents a session ticket from the previous
connection, and the relay resumes delivery from where it left off
if the cached session state remains valid. This typically completes
in a single round trip.

~~~
QUIC 0-RTT Resumption:

Subscriber                                Relay
    │                                       │
    │   [Connection lost briefly]           │
    │                                       │
    │   [Reconnecting with session ticket]  │
    │                                       │
    │── QUIC 0-RTT ────────────────────────▶│
    │   ClientHello + session ticket        │
    │   + early data (SUBSCRIBE)            │
    │                                       │
    │◄── ServerHello + Finished ────────────│
    │                                       │
    │◄── Continue from last position ───────│
    │    (relay cached subscriber state)    │
    │                                       │
    │══ Resume live stream (minimal gap) ══▶│
~~~

### Extended Disconnections: Cursor-Based Reconnection

For extended disconnections where session state has expired, subscribers
perform a full reconnection using their saved cursor position. The
subscriber first uses FETCH to retrieve any missed events from their
last known position, then establishes a new SUBSCRIBE starting from
the latest group to receive live updates. Relay caches serve the
historical data when available, minimizing load on upstream servers.

~~~
Full Reconnection with Cursor:

Subscriber                                Relay
    │                                       │
    │   [Disconnected for extended period]  │
    │   [Saved state: at-seq = 12345678]    │
    │                                       │
    │── QUIC Handshake (full) ─────────────▶│
    │◄── Connection Established ────────────│
    │                                       │
    │── CLIENT_SETUP ──────────────────────▶│
    │◄── SERVER_SETUP ──────────────────────│
    │                                       │
    │   [Convert cursor to Group/Object]    │
    │   seq 12345678 → Group 5, Object 678  │
    │                                       │
    │── FETCH ─────────────────────────────▶│
    │   StartGroup=5, StartObject=678       │
    │   EndGroup=LATEST                     │
    │                                       │
    │◄── Historical objects (cached) ───────│
    │                                       │
    │── SUBSCRIBE ─────────────────────────▶│
    │   StartGroup=LATEST                   │
    │◄── SUBSCRIBE_OK ──────────────────────│
    │                                       │
    │══ Live stream resumes ═══════════════▶│
~~~

### Network Transitions: Connection Migration

For network transitions such as WiFi-to-cellular handoffs, QUIC's
connection migration enables seamless continuation without any
application-layer intervention. The subscriber's IP address changes
transparently while the MOQT session continues uninterrupted, ensuring
no gaps in the event stream during mobile network transitions.

~~~
QUIC Connection Migration:

Subscriber (mobile)                       Relay
    │                                       │
    │══ Streaming on WiFi ═════════════════▶│
    │   (IP: 192.168.1.50)                  │
    │                                       │
    │   [User moves, WiFi signal lost]      │
    │   [Device switches to cellular]       │
    │   [New IP: 10.0.0.75]                 │
    │                                       │
    │── QUIC PATH_CHALLENGE ───────────────▶│
    │   (from new IP address)               │
    │                                       │
    │◄── QUIC PATH_RESPONSE ────────────────│
    │   (path validated)                    │
    │                                       │
    │══ Continue streaming on cellular ════▶│
    │   (no gap, no reconnection needed)    │
    │                                       │
    │   [Application layer unaware of       │
    │    network transition]                │
~~~

# Authentication and Authorization

TODO

# Security Considerations

TODO

# IANA Considerations

TODO

--- back
