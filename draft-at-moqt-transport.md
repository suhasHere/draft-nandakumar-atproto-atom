---
title: "Media over QUIC Transport (MOQT) for Authenticated Transfer Protocol"
abbrev: "AT over MOQT"
category: std

docname: draft-at-moqt-transport-latest
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
  github: "ietf-wg-moq/draft-at-moqt-transport"
  latest: "https://ietf-wg-moq.github.io/draft-at-moqt-transport/draft-at-moqt-transport.html"

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
can serve as an enhanced transport for AT Protocol, leveraging MOQT's
native publish/subscribe model, relay infrastructure, and priority-based
delivery to address current scalability and resilience challenges.

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
disconnection. However:

* Consumers must track cursor state externally
* Gap detection requires commit chain validation
* Recovery from gaps requires full repository fetch (thundering herd risk)

### No Native Late-Join Support

When a new consumer connects to a firehose:

* Must start from current position or replay from cursor
* No concept of "recent state" for quick synchronization
* Cannot receive cached recent events without full replay

### Connection Fragility

WebSocket connections over TCP are susceptible to:

* Network path changes causing connection drops
* No connection migration during IP address changes
* Full reconnection required after network transitions

## Operational Challenges

### Relay Infrastructure Complexity

Current relays must:

* Maintain persistent WebSocket connections to all upstream PDS instances
* Handle reconnection and cursor tracking for each upstream
* Implement custom caching and replay logic
* Build proprietary distribution mechanisms for downstream subscribers

### Limited Quality of Service

No native mechanism exists for:

* Differentiating between event types by priority
* Rate limiting based on consumer capacity
* Graceful degradation during overload conditions

# MOQT Transport Benefits

MOQT addresses the challenges identified above through its native
architecture designed for large-scale media distribution.

## Native Publish/Subscribe Model

MOQT's publish/subscribe model aligns naturally with AT Protocol's
firehose semantics:

~~~
MOQT Pub/Sub Model for AT Firehose:

┌─────────────────────────────────────────────────────────────────┐
│                      MOQT Relay Network                         │
│                                                                 │
│    ┌──────────────────────────────────────────────────────┐    │
│    │                                                      │    │
│    │   Namespace: at/{pds}/firehose                       │    │
│    │                                                      │    │
│    │   Track: commits    ──────▶ #commit events           │    │
│    │   Track: identity   ──────▶ #identity events         │    │
│    │   Track: account    ──────▶ #account events          │    │
│    │   Track: sync       ──────▶ #sync events             │    │
│    │                                                      │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                 │
│  Publishers (PDS)              Subscribers (Relays, AppViews)   │
│       │                                     ▲                   │
│       └──────── PUBLISH ─────────────────────┘                  │
│                 SUBSCRIBE                                       │
└─────────────────────────────────────────────────────────────────┘
~~~

Benefits include:

* Subscribers choose specific event tracks of interest
* Publishers announce available tracks via PUBLISH_NAMESPACE
* No connection per subscriber - relays distribute efficiently
* Track-level subscription granularity reduces unnecessary traffic

## Priority-Based Delivery

MOQT's 0-255 priority scale enables differentiated event delivery:

| AT Event Type | MOQT Priority | Rationale |
|---------------|---------------|-----------|
| #account (takedown) | 0-15 | Critical - affects content availability |
| #identity | 16-31 | Urgent - key rotation, handle changes |
| #account (status) | 32-63 | Important - account state changes |
| #sync | 64-95 | Moderate - state reset events |
| #commit (small) | 96-127 | Normal - typical record operations |
| #commit (large) | 128-191 | Background - bulk content updates |
| Repository export | 192-255 | Bulk - full sync operations |

~~~
Priority-Based Event Delivery:

MOQT Connection (Multiplexed Streams)
│
├── Stream 1 (Priority 16): #identity event ──────▶ Delivered first
├── Stream 2 (Priority 100): #commit (small) ─────▶ Delivered second
└── Stream 3 (Priority 160): #commit (large) ─────▶ Delivered last

Result: Critical events bypass large transfers without blocking
~~~

## Relay Infrastructure

MOQT relays provide purpose-built infrastructure for AT Protocol
distribution:

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

Relay benefits:

* Hierarchical distribution reduces origin load
* Geographic optimization for latency
* Built-in caching at each relay tier
* Standardized relay protocol (no custom implementation)

## Group-Based Organization and Caching

MOQT groups enable efficient caching and late-join support:

~~~
MOQT Object Hierarchy for Firehose:

Track: at/{pds}/firehose/commits/{did}
│
├── Group 0 (Commits 0-99)
│   ├── Object 0: #commit event (rev: tid-0001)
│   ├── Object 1: #commit event (rev: tid-0002)
│   └── ...
│
├── Group 1 (Commits 100-199)
│   ├── Object 0: #commit event (rev: tid-0100)
│   └── ...
│
└── Group N (Current)  ◄─── Late-join point
    ├── Object 0: #commit event (rev: tid-N00)
    └── Object 1: #commit event (rev: tid-N01)

Relay Cache:
┌────────────────────────────────────────────┐
│ Hot Cache: Groups N, N-1 (recent events)   │
│ Warm Cache: Groups N-2 to N-10             │
│ Cold: Fetch from upstream on demand        │
└────────────────────────────────────────────┘
~~~

Benefits:

* Late-joining subscribers receive recent group immediately
* Relay caches reduce upstream load
* Group boundaries provide natural replay points
* Cursor semantics map to Group ID + Object ID

## QUIC Transport Advantages

Leveraging QUIC {{RFC9000}} provides:

* **Connection Migration**: Maintain firehose subscription during
  network changes (IP address change, WiFi to cellular)
* **Multiplexed Streams**: Multiple logical channels without
  head-of-line blocking
* **Built-in Encryption**: TLS 1.3 {{RFC8446}} mandatory
* **Flow Control**: Per-stream and connection-level flow control
  prevents subscriber overload
* **0-RTT Resumption**: Fast reconnection after brief disconnections

# MOQT Transport Mapping

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

### Setup Parameters

The following MOQT setup parameters are defined for AT Protocol:

| Parameter | ID | Type | Description |
|-----------|----|------|-------------|
| at-version | 0x41540001 | varint | AT Protocol version |
| at-supported-events | 0x41540002 | varint | Bitmask of supported event types |
| at-relay-caps | 0x41540003 | varint | Relay capability flags |

Event type bitmask values:

* 0x01: #commit events
* 0x02: #identity events
* 0x04: #account events
* 0x08: #sync events

## Namespace Structure

AT Protocol firehose events map to MOQT namespaces as follows:

~~~
AT Protocol Namespace Hierarchy:

Track Namespace (tuple fields):
┌─────────┬───────────────┬────────────────┬─────────────────┐
│ Field 1 │   Field 2     │    Field 3     │    Field 4      │
│  "at"   │  {pds-host}   │   "firehose"   │  {event-type}   │
└─────────┴───────────────┴────────────────┴─────────────────┘

Track Name:
┌────────────────────────────────────────────────────────────┐
│                    {did} or "all"                          │
└────────────────────────────────────────────────────────────┘

Examples:
  at / bsky.social / firehose / commits -- did:plc:abc123
  at / pds.example.com / firehose / identity -- all
  at / relay.bsky.network / firehose / account -- all
~~~

### Track Categories

Commits Track:
: Namespace: `at/{host}/firehose/commits`
: Carries #commit events with repository updates. Track name is the
  DID of the repository or "all" for aggregated streams.

Identity Track:
: Namespace: `at/{host}/firehose/identity`
: Carries #identity events signaling DID document or handle changes.

Account Track:
: Namespace: `at/{host}/firehose/account`
: Carries #account events for hosting status changes.

Sync Track:
: Namespace: `at/{host}/firehose/sync`
: Carries #sync events asserting current repository state.

## Message Serialization

AT Protocol firehose events are encapsulated in MOQT objects with
the following structure:

~~~
MOQT Object Structure for AT Events:

┌─────────────────────────────────────────────────────────────┐
│                 MOQT Object Header                          │
│  Track ID │ Group ID │ Object ID │ Priority │ Extensions    │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│              AT Event Extension Headers                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ at-event-type (0x41544501): varint                     │ │
│  │ at-repo-did (0x41544502): string                       │ │
│  │ at-repo-rev (0x41544503): string (TID)                 │ │
│  │ at-seq (0x41544504): varint (cursor equivalent)        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│              AT Event Payload (CBOR)                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  {                                                     │ │
│  │    "repo": "did:plc:...",                              │ │
│  │    "rev": "tid-string",                                │ │
│  │    "since": "tid-string",                              │ │
│  │    "blocks": <CAR bytes>,                              │ │
│  │    "ops": [...]                                        │ │
│  │  }                                                     │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
~~~

The payload format remains compatible with the existing CBOR encoding
defined in {{AT-REPO}}, ensuring backward compatibility with existing
AT Protocol implementations.

## Group Organization

MOQT groups organize firehose events for efficient caching and replay:

~~~
Group Organization for Firehose:

Track: at/{pds}/firehose/commits -- did:plc:user1

Time ───────────────────────────────────────────────────────▶

│ Group 0            │ Group 1            │ Group 2 (live)   │
│ (seq: 0-999)       │ (seq: 1000-1999)   │ (seq: 2000+)     │
├────────────────────┼────────────────────┼──────────────────┤
│ Obj 0: #commit     │ Obj 0: #commit     │ Obj 0: #commit   │
│ Obj 1: #commit     │ Obj 1: #commit     │ Obj 1: (pending) │
│ ...                │ ...                │                  │
│ Obj 999: #commit   │ Obj 999: #commit   │                  │
├────────────────────┼────────────────────┼──────────────────┤
│     Cached         │    Cached          │     Live         │
└────────────────────┴────────────────────┴──────────────────┘

Cursor Mapping:
  AT cursor: 1500  ───▶  MOQT: Group 1, Object 500
  AT cursor: 2001  ───▶  MOQT: Group 2, Object 1
~~~

Group boundaries are determined by:

* Fixed object count per group (e.g., 1000 events)
* Time-based boundaries (e.g., 1-minute groups)
* Size-based boundaries (e.g., 10MB per group)

The specific grouping strategy is implementation-defined but MUST
be consistent within a track to enable proper cursor mapping.

# Protocol Operations

## Firehose Subscription

Subscribers connect to the firehose using standard MOQT subscription:

~~~
Firehose Subscription Flow:

Subscriber                    MOQT Relay                      PDS
    │                             │                            │
    │── SUBSCRIBE_NAMESPACE ─────▶│                            │
    │   (at/*/firehose)           │                            │
    │                             │                            │
    │◄─ NAMESPACE ────────────────│                            │
    │   (available PDS hosts)     │                            │
    │                             │                            │
    │── SUBSCRIBE ───────────────▶│                            │
    │   (at/{pds}/firehose/       │                            │
    │    commits -- all)          │                            │
    │   StartGroup: Latest        │                            │
    │                             │                            │
    │◄─ SUBSCRIBE_OK ─────────────│                            │
    │   (Latest = Group N)        │                            │
    │                             │                            │
    │◄══ Objects (live events) ═══│◄══ Objects ════════════════│
    │                             │                            │
    │   [Subscriber processing]   │   [Relay caches objects]   │
    │                             │                            │
~~~

### Subscription Options

Start Position:
: Subscribers specify where to begin receiving events using MOQT's
  StartGroup and StartObject parameters:

  * `Latest`: Begin with current live events (default)
  * `Absolute(group, object)`: Resume from specific cursor position
  * `Earliest`: Receive all cached events (relay-dependent)

Filter:
: Track name filtering enables per-DID subscriptions:

  * `all`: Receive events for all repositories
  * `{did}`: Receive events only for specific DID

## Relay Aggregation

MOQT relays aggregate firehose streams from multiple upstream sources:

~~~
Relay Aggregation Architecture:

                    Full-Network Relay
                   ┌────────────────────┐
                   │  Aggregates all    │
                   │  regional relays   │
                   │                    │
                   │  Publishes:        │
                   │  at/relay.network/ │
                   │    firehose/*      │
                   └─────────▲──────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
 ┌──────┴──────┐      ┌──────┴──────┐      ┌──────┴──────┐
 │ Regional    │      │ Regional    │      │ Regional    │
 │ Relay A     │      │ Relay B     │      │ Relay C     │
 │             │      │             │      │             │
 │ Subscribes: │      │ Subscribes: │      │ Subscribes: │
 │ at/pds-*    │      │ at/pds-*    │      │ at/pds-*    │
 │ (region A)  │      │ (region B)  │      │ (region C)  │
 └──────▲──────┘      └──────▲──────┘      └──────▲──────┘
        │                    │                    │
   ┌────┴────┐          ┌────┴────┐          ┌────┴────┐
   │         │          │         │          │         │
 PDS A1   PDS A2      PDS B1   PDS B2      PDS C1   PDS C2
~~~

Relays perform:

* **Subscription Aggregation**: Single subscription to each upstream,
  multiplexed to many downstream subscribers
* **Namespace Republishing**: Aggregate events under relay's namespace
* **Event Caching**: Store recent groups for late-join and replay
* **Deduplication**: Prevent duplicate events from multiple paths

## Repository Synchronization

Full repository exports can also leverage MOQT for improved
distribution:

~~~
Repository Export over MOQT:

Track: at/{pds}/repo/{did}/export

Group 0 (CAR Header + Commit)
├── Object 0: CAR header
├── Object 1: Signed commit block
└── Object 2: MST root block

Group 1 (MST Level 0)
├── Object 0: MST node (prefix: a-f)
├── Object 1: MST node (prefix: g-m)
└── Object 2: MST node (prefix: n-z)

Group 2 (Records: app.bsky.feed.*)
├── Object 0: Record block (post/1)
├── Object 1: Record block (post/2)
└── ...

Group 3 (Records: app.bsky.graph.*)
├── Object 0: Record block (follow/1)
└── ...

Subscriber can:
- Subscribe to specific groups (e.g., only posts)
- Receive cached export from relay
- Resume interrupted downloads
~~~

This approach provides:

* Parallel download of repository sections
* Relay caching of popular exports
* Resume capability for large repositories
* Priority-based delivery (commit first, then critical records)

## Event Type Processing

### #commit Events

Commit events contain repository updates:

~~~json
{
  "repo": "did:plc:z72i7hdynmk6r22z27h6tvur",
  "rev": "3juj472xqex2v",
  "since": "3juj472xqex2u",
  "blocks": "<CAR-encoded-blocks>",
  "ops": [
    {
      "action": "create",
      "path": "app.bsky.feed.post/3juj472xqex2v",
      "cid": "bafyrei..."
    }
  ]
}
~~~

Mapping to MOQT:

* Track: `at/{host}/firehose/commits -- {did}`
* Priority: 96-127 (small) or 128-191 (large, based on blocks size)
* Group Order: Ascending (preserve commit sequence)

### #identity Events

Identity events signal potential DID document changes:

~~~json
{
  "did": "did:plc:z72i7hdynmk6r22z27h6tvur",
  "handle": "alice.bsky.social",
  "seq": 12345678
}
~~~

Mapping to MOQT:

* Track: `at/{host}/firehose/identity -- all`
* Priority: 16-31 (high priority)
* Group Order: Descending (prioritize recent changes)

### #account Events

Account events indicate hosting status changes:

~~~json
{
  "did": "did:plc:z72i7hdynmk6r22z27h6tvur",
  "active": false,
  "status": "suspended",
  "seq": 12345679
}
~~~

Mapping to MOQT:

* Track: `at/{host}/firehose/account -- all`
* Priority: 0-15 (takedown) or 32-63 (other status)
* Group Order: Descending (prioritize recent status)

### #sync Events

Sync events assert current repository state:

~~~json
{
  "did": "did:plc:z72i7hdynmk6r22z27h6tvur",
  "rev": "3juj472xqex2v",
  "blocks": "<commit-block-only>"
}
~~~

Mapping to MOQT:

* Track: `at/{host}/firehose/sync -- {did}`
* Priority: 64-95
* Group Order: Ascending

# Reliability and Recovery

## Cursor Mapping

AT Protocol cursors map to MOQT group and object identifiers:

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

MOQT's group-based delivery simplifies gap detection:

~~~
Gap Detection Flow:

Subscriber State:
  Last received: Group 5, Object 999
  Expected next: Group 6, Object 0

Received: Group 6, Object 5  ← Gap detected!

Recovery Options:

1. Request Missing Objects (preferred)
   │
   └── SUBSCRIBE with StartGroup=6, StartObject=0
       └── Relay serves from cache

2. Request Full Group
   │
   └── SUBSCRIBE with StartGroup=6
       └── Receive entire group

3. Full Repository Sync (fallback)
   │
   └── Fetch CAR export via HTTP or MOQT repo track
       └── Validate against #sync event
~~~

## Disconnection Handling

MOQT over QUIC provides superior disconnection handling:

~~~
Reconnection Scenarios:

Scenario 1: Brief Disconnection (< 30s)
┌────────────────────────────────────────────────────────────┐
│ QUIC 0-RTT Resumption                                      │
│                                                            │
│ Subscriber                          Relay                  │
│     │                                 │                    │
│     │── QUIC 0-RTT ──────────────────▶│                    │
│     │   (session ticket + request)    │                    │
│     │                                 │                    │
│     │◄── Continue from last position ─│                    │
│     │    (cached state preserved)     │                    │
└────────────────────────────────────────────────────────────┘

Scenario 2: Extended Disconnection
┌────────────────────────────────────────────────────────────┐
│ Full Reconnection with Cursor                              │
│                                                            │
│ Subscriber saves: at-seq = 12345678                        │
│                                                            │
│ On reconnect:                                              │
│   1. Establish new MOQT session                            │
│   2. SUBSCRIBE with StartGroup/StartObject from cursor     │
│   3. Relay serves cached objects if available              │
│   4. Otherwise, relay fetches from upstream                │
└────────────────────────────────────────────────────────────┘

Scenario 3: Connection Migration
┌────────────────────────────────────────────────────────────┐
│ QUIC Connection Migration                                  │
│                                                            │
│ Subscriber (mobile)                 Relay                  │
│     │                                 │                    │
│     │══ Streaming on WiFi ═══════════▶│                    │
│     │                                 │                    │
│  [Network change: WiFi → Cellular]    │                    │
│     │                                 │                    │
│     │── QUIC PATH_CHALLENGE ─────────▶│                    │
│     │◄─ QUIC PATH_RESPONSE ───────────│                    │
│     │                                 │                    │
│     │══ Continue streaming (no gap) ══▶│                   │
└────────────────────────────────────────────────────────────┘
~~~

# Authentication and Authorization

## Connection Authentication

AT Protocol authentication integrates with MOQT at multiple levels:

~~~
Authentication Layers:

┌─────────────────────────────────────────────────────────────┐
│ Layer 1: TLS Client Authentication                          │
│                                                             │
│ - Mutual TLS for service-to-service (PDS ↔ Relay)           │
│ - Server-only TLS for public subscribers                    │
└─────────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: MOQT Setup Authentication                          │
│                                                             │
│ - Bearer token in CLIENT_SETUP extension                    │
│ - OAuth 2.0 access token validation                         │
│ - DPoP proof for proof-of-possession                        │
└─────────────────────────────────────────────────────────────┘
                           │
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Track-Level Authorization                          │
│                                                             │
│ - C4M tokens with namespace scopes                          │
│ - Per-subscription authorization checks                     │
└─────────────────────────────────────────────────────────────┘
~~~

## Relay Authorization

MOQT relays enforce authorization using Common Access Tokens {{MoQ-C4M}}:

~~~json
{
  "iss": "https://auth.bsky.network",
  "sub": "did:web:indexer.example.com",
  "aud": "https://relay.bsky.network",
  "exp": 1709164800,
  "scope": "at:subscribe at:subscribe_namespace",
  "at_namespaces": [
    "at/*/firehose/commits",
    "at/*/firehose/identity"
  ],
  "at_rate_limit": 10000
}
~~~

Token claims:

* `at_namespaces`: Allowed namespace patterns
* `at_rate_limit`: Maximum events per second
* `scope`: Permitted MOQT operations

## Data Integrity

AT Protocol's cryptographic guarantees are preserved:

* Commit signatures verified per {{AT-REPO}} specification
* DID resolution performed independently by consumers
* MST validation confirms data integrity
* MOQT transport provides authentication, not data authorization

# Security Considerations

## Transport Security

All AT-over-MOQT connections MUST use TLS 1.3 {{RFC8446}} or later.
QUIC's mandatory encryption protects:

* Firehose event content from eavesdropping
* Subscription patterns from traffic analysis (partial)
* Authentication credentials in transit

## Relay Trust Model

MOQT relays operate as trusted intermediaries:

* Relays see all events they forward (no E2E encryption at relay layer)
* Rate limiting and access control enforced by relays
* Event ordering and caching controlled by relays

Consumers MUST:

* Validate commit signatures regardless of relay trust
* Independently resolve DID documents
* Detect and report event manipulation

## Denial of Service

MOQT's built-in flow control mitigates DoS risks:

* Per-stream flow control prevents single-publisher flooding
* Subscription limits bound per-connection resource usage
* Priority scheduling ensures critical events delivered during congestion

Additional mitigations:

* Rate limiting at subscription creation
* Namespace-based access control
* Anomaly detection for unusual patterns

## Privacy Considerations

Firehose events contain public data but reveal subscriber patterns:

* Subscription namespaces visible to relays
* Connection metadata (IP, timing) observable
* Consider namespace aggregation for privacy-sensitive subscribers

# IANA Considerations

## MOQT Setup Parameter Registration

This document requests registration of the following MOQT setup parameters:

| Parameter Name | Parameter ID | Description |
|----------------|--------------|-------------|
| at-version | 0x41540001 | AT Protocol version |
| at-supported-events | 0x41540002 | Supported event type bitmask |
| at-relay-caps | 0x41540003 | Relay capability flags |

## MOQT Extension Header Registration

This document requests registration of the following MOQT object
extension headers:

| Header Name | Header ID | Description |
|-------------|-----------|-------------|
| at-event-type | 0x41544501 | AT event type identifier |
| at-repo-did | 0x41544502 | Repository DID |
| at-repo-rev | 0x41544503 | Repository revision (TID) |
| at-seq | 0x41544504 | Cursor sequence number |

--- back

# Acknowledgments

The authors would like to thank the IETF MOQ working group for their
work on the MOQT specification, and the Bluesky team for their work
on the AT Protocol specifications.

# Comparison: WebSocket vs MOQT Transport

| Feature | WebSocket (Current) | MOQT |
|---------|---------------------|------|
| Connection Model | 1:1 per subscriber | Relay-distributed |
| Event Filtering | Application layer | Track subscription |
| Priority Handling | None | Native (0-255) |
| Head-of-Line Blocking | Yes (TCP) | No (QUIC streams) |
| Late Join | Cursor replay | Group cache |
| Connection Migration | No | QUIC native |
| Flow Control | TCP only | Per-stream |
| Multiplexing | Single stream | Multi-track |
| Caching | Custom | Relay native |
| Standard Relay Protocol | No | Yes |

# Example: Full Firehose Subscription

~~~
Complete Subscription Flow:

AppView Indexer                MOQT Relay               PDS Hosts
       │                            │                       │
       │── QUIC Connect ───────────▶│                       │
       │◄─ QUIC Connected ──────────│                       │
       │                            │                       │
       │── CLIENT_SETUP ───────────▶│                       │
       │   at-version: 1            │                       │
       │   at-supported-events: 0xF │                       │
       │                            │                       │
       │◄─ SERVER_SETUP ────────────│                       │
       │   at-relay-caps: 0x7       │                       │
       │                            │                       │
       │── SUBSCRIBE_NAMESPACE ────▶│                       │
       │   (at/*/firehose)          │                       │
       │                            │                       │
       │◄─ NAMESPACE ───────────────│                       │
       │   (1000+ PDS hosts)        │                       │
       │                            │                       │
       │── SUBSCRIBE ──────────────▶│                       │
       │   Track: at/relay.bsky.    │                       │
       │         network/firehose/  │                       │
       │         commits -- all     │                       │
       │   StartGroup: Latest       │                       │
       │   Priority: 0-191          │                       │
       │                            │                       │
       │◄─ SUBSCRIBE_OK ────────────│                       │
       │   ContentExists: true      │                       │
       │   Latest: Group 5678       │                       │
       │                            │                       │
       │◄══ Object Stream ══════════│◄══════════════════════│
       │   Group 5678, Obj 0        │   #commit events      │
       │   Group 5678, Obj 1        │   from all PDS        │
       │   ...                      │                       │
       │                            │                       │
       │   [Process events,         │   [Relay caches       │
       │    validate signatures,    │    recent groups,     │
       │    update indexes]         │    manages upstream   │
       │                            │    subscriptions]     │
~~~

# Migration Considerations

Deploying MOQT alongside existing WebSocket transport:

1. **Parallel Operation**: Run both transports simultaneously
2. **Feature Detection**: Advertise MOQT support in well-known endpoint
3. **Gradual Migration**: Subscribers choose transport based on capability
4. **Relay Bridging**: MOQT relays can subscribe to WebSocket upstreams
5. **Monitoring**: Compare delivery latency and reliability metrics

~~~
Migration Architecture:

                    ┌─────────────────┐
                    │   Hybrid Relay  │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │  WebSocket  │ │◄── Legacy PDS
                    │ │  Ingress    │ │
                    │ └──────┬──────┘ │
                    │        │        │
                    │ ┌──────▼──────┐ │
                    │ │   Event     │ │
                    │ │   Bridge    │ │
                    │ └──────┬──────┘ │
                    │        │        │
                    │ ┌──────▼──────┐ │
                    │ │    MOQT     │ │──▶ New Subscribers
                    │ │   Egress    │ │
                    │ └─────────────┘ │
                    │                 │
                    │ ┌─────────────┐ │
                    │ │    MOQT     │ │◄── New PDS
                    │ │   Ingress   │ │
                    │ └─────────────┘ │
                    └─────────────────┘
~~~
