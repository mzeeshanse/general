# Video Gateway — High-Level Design (HLD)

| | |
| --- | --- |
| **Document Type** | Technical Architecture / Interview Response |
| **Status** | Submission Draft |
| **Version** | 2.0 |
| **Date** | August 2026 |

---

## 1. Purpose

This document presents a production-oriented high-level design for the Video Gateway described in the project brief.

The gateway is a back-end product that connects to third-party Video Management Systems (VMS), obtains live and recorded video, camera-control capabilities, and events, and exposes them through a common gateway interface and modern streaming protocols.

The gateway is **not** a replacement for a VMS. The VMS remains the source of truth for cameras, recordings and operator workflows. The gateway provides the integration and media-delivery layer between VMS platforms and consumers.

The architecture is deliberately divided into:

- **Edge / Control Communication Path** — HTTP/REST/gRPC requests, authentication, authorization, administration and control operations.
- **Video / Media Data Path** — actual video acquisition, media processing and protocol delivery.
- **Asynchronous Event Path** — events and audit messages through a message bus.

This separation is important because video data has very different performance and lifecycle characteristics from normal API traffic.

---

## 2. Scope and Requirements Alignment

The project brief requires the gateway to:

- Connect to multiple VMS systems through vendor SDKs, ONVIF or generic stream APIs.
- Keep the VMS as the source of truth and never connect directly to cameras.
- Discover cameras and provide a unified inventory.
- Deliver live video.
- Proxy recorded playback and support time-range requests, seeking and clip generation.
- Optionally archive selected streams on the gateway.
- Pass through PTZ, presets, tours, focus and iris controls through the VMS.
- Subscribe to motion, alarm, tamper and analytics events.
- Provide a substantial administration surface.
- Support Milestone XProtect, Genetec Security Center, Avigilon, Bosch BVMS, ONVIF-compatible systems and generic RTSP/HTTP systems.
- Produce RTSP, HLS, WebRTC, SRT and RTMP outputs.
- Avoid transcoding when the source is already compatible.
- Provide multi-tenant access control, auditing, observability and production operations.

The proposed architecture addresses these requirements while keeping VMS-specific SDK dependencies isolated behind adapters.

---

## 3. Architectural Principles

### 3.1 VMS remains the source of truth

The gateway does not replace the VMS and does not directly manage cameras.

![VMS remains the source of truth](diagrams/01-vms-source-of-truth.png)

The gateway reads and controls capabilities exposed by the VMS.

### 3.2 Control plane and media plane are separate

Normal API/control traffic and video bytes must not be treated as the same workload.

The **control plane** handles:

- Authentication
- Authorization
- Camera discovery
- VMS management
- Stream session creation
- Recording requests
- PTZ commands
- Configuration
- Administration
- Audit/event coordination

The **media plane** handles:

- Media acquisition
- Demuxing
- Transmuxing/repackaging
- Selective transcoding
- Segmentation
- Protocol-specific media output
- Long-running media sessions

The message bus is **not** part of the media data path.

### 3.3 nginx is an edge component, not the media engine

nginx is used as the edge reverse proxy / ingress component.

Its responsibilities may include:

- TLS termination
- External ingress
- HTTP routing
- Connection handling
- Edge-level load balancing
- Infrastructure-level policies

nginx is **not** responsible for:

- Video decoding
- Video encoding
- Transcoding
- Demuxing
- Media pipeline orchestration
- Codec conversion

Normal nginx reverse-proxy behavior should not be assumed to mean that every streaming protocol is proxied through nginx.

For HTTP-based media delivery such as HLS, nginx may participate in the delivery/routing path when appropriate.

For RTSP, SRT, RTMP or WebRTC media transport, the deployment must explicitly support the required protocol and connection model. The media engine remains responsible for media processing.

### 3.4 API Gateway is an architectural role; YARP is the implementation

The architecture uses an API Gateway as the application-level gateway.

Proposed implementation:

- **Architectural role:** API Gateway — REST + gRPC
- **Implementation:** YARP (Yet Another Reverse Proxy)

YARP is a .NET reverse-proxy toolkit that can implement application/API gateway responsibilities.

This is intentionally different from nginx:

| Component | Role |
| --- | --- |
| **nginx** | Infrastructure / edge proxy |
| **API Gateway (YARP)** | Application / API routing layer |

The architecture does not require YARP to process video bytes.

---

## 4. High-Level Architecture

The architecture is best represented with separate diagrams rather than placing nginx, YARP and the media engine into one ambiguous flow.

---

## 5. Diagram 1 — Edge and Control Communication Path

**Purpose:** HTTP/REST/gRPC and control communication. This diagram does **not** represent the actual video-byte path.

![Diagram 1 — Edge and Control Communication Path](diagrams/02-control-path.png)

```mermaid
flowchart LR
    C[Consumer<br/>Web / Mobile / Integration] --> NG[nginx<br/>Edge Reverse Proxy<br/>TLS / Ingress / Load Balancing]

    NG --> GW[API Gateway<br/>REST + gRPC<br/>Implementation: YARP]

    GW --> AUTH[Auth / RBAC Service]
    GW --> DEV[Device / VMS Service]
    GW --> STR[Stream Service]
    GW --> REC[Recording Service]
    GW --> EVT[Event Service]
    GW --> ADM[Administration APIs]

    DEV --> CONN[VMS Connector Layer]
    STR --> CONN
    REC --> CONN

    CONN --> VMS[VMS Platforms<br/>Milestone / Genetec / Avigilon / Bosch / ONVIF / Generic]
```

### Diagram interpretation

1. Consumer sends an API/control request.
2. nginx receives the external connection.
3. nginx performs edge-level responsibilities and forwards the API request.
4. The API Gateway provides application-level routing.
5. YARP is the proposed implementation of the API Gateway.
6. Authentication and authorization are applied before protected operations proceed.
7. Application services perform business operations.
8. VMS connectors translate gateway contracts into vendor-specific VMS operations.
9. The VMS remains the source of truth.

### Important boundary

No video stream should be assumed to flow through the API Gateway.

The API Gateway returns metadata such as:

- stream URL
- stream token
- session ID
- playback handle
- operation result

The actual media transfer follows the separate media path.

---

## 6. Diagram 2 — Video / Media Data Path

**Purpose:** the actual video path.

![Diagram 2 — Video / Media Data Path](diagrams/03-media-path.png)

```mermaid
flowchart LR
    VMS[VMS<br/>Live / Recorded Media] --> CONN[VMS Connector<br/>Vendor SDK / ONVIF / RTSP / HTTP]

    CONN --> SOURCE[Media Source]

    SOURCE --> MP[Media Pipeline<br/>GStreamer / FFmpeg]

    MP --> DEMUX[Demux]
    DEMUX --> DECIDE{Source already<br/>compatible?}

    DECIDE -->|Yes| PASS[Passthrough / Transmux]
    DECIDE -->|No| TRANS[Selective Transcoding]

    PASS --> PACK[Protocol Packaging / Output]
    TRANS --> PACK

    PACK --> RTSP[RTSP]
    PACK --> HLS[HLS]
    PACK --> WEBRTC[WebRTC]
    PACK --> SRT[SRT]
    PACK --> RTMP[RTMP]

    RTSP --> C[Consumer]
    HLS --> EDGE[HTTP Edge / nginx where applicable]
    EDGE --> C
    WEBRTC --> C
    SRT --> C
    RTMP --> C
```

### Critical architecture statement

The **Media Pipeline** is the media-processing component. nginx is **not** the media engine.

The gateway should use passthrough or repackaging whenever the source is compatible with the requested output. Transcoding should only occur when required by codec, container, profile or protocol constraints.

---

## 7. Protocol-Specific Edge Considerations

The output protocols have different connection and delivery characteristics.

| Protocol | Typical purpose | nginx position |
| --- | --- | --- |
| **RTSP** | NVRs, players, integrations | Do not assume normal HTTP reverse proxying; use a protocol-capable media endpoint. |
| **HLS** | Browser/mobile/CDN distribution | nginx can participate as HTTP edge/proxy/static segment delivery where appropriate. |
| **WebRTC** | Low-latency interactive monitoring | nginx can handle HTTP/signaling edge traffic where applicable; real-time media transport is handled separately. |
| **SRT** | Long-distance resilient contribution | Use a protocol-capable media endpoint; do not assume standard nginx HTTP proxy behavior. |
| **RTMP** | Legacy ingest/external CDN | Requires appropriate protocol support; do not represent generic nginx HTTP proxying as RTMP media handling. |

This prevents the architecture diagram from making the incorrect implication that nginx automatically proxies every media protocol.

---

## 8. Diagram 3 — Relationship Between Control Plane and Media Plane

The two paths are connected by session/control metadata, not by sending the video through the control services.

![Diagram 3 — Control plane and media plane](diagrams/04-control-vs-media.png)

```mermaid
flowchart TB
    C[Consumer]

    subgraph CONTROL[CONTROL / API PATH]
        NG[nginx<br/>Edge]
        GW[API Gateway<br/>REST + gRPC<br/>YARP]
        AUTH[Auth / RBAC]
        STR[Stream Service]
    end

    subgraph MEDIA[MEDIA / VIDEO PATH]
        CONN[VMS Connector]
        MP[Media Pipeline<br/>GStreamer / FFmpeg]
        OUT[Protocol Output<br/>RTSP / HLS / WebRTC / SRT / RTMP]
    end

    VMS[VMS]

    C -->|Stream request| NG
    NG --> GW
    GW --> AUTH
    AUTH --> STR

    STR -->|Open / reuse session| CONN
    CONN --> VMS
    VMS -->|Media| CONN
    CONN --> MP
    MP --> OUT
    OUT --> C

    STR -.->|Stream URL / token / session metadata| GW
    GW -.->|Response| C
```

### Key point

The control path establishes and authorizes the media session.

The media path then carries the actual video.

---

## 9. Diagram 4 — Live Stream Request Sequence

This sequence shows the relationship between API/control and media delivery.

![Diagram 4 — Live Stream Request Sequence](diagrams/05-live-stream-sequence.png)

```mermaid
sequenceDiagram
    autonumber
    participant C as Consumer
    participant NG as nginx
    participant GW as API Gateway (YARP)
    participant AUTH as Auth/RBAC
    participant STR as Stream Service
    participant R as Redis
    participant CONN as VMS Connector
    participant VMS as VMS
    participant MP as Media Pipeline
    participant BUS as Message Bus
    participant EVT as Event Service
    participant CH as ClickHouse

    C->>NG: POST /streams
    NG->>GW: Forward REST/gRPC request
    GW->>AUTH: Authenticate / Authorize
    AUTH-->>GW: Allowed
    GW->>STR: RequestStream(cameraId, profile, protocol)

    STR->>R: Atomic lookup/create session

    alt Existing session
        R-->>STR: Existing session + refcount
        STR->>R: Increment refcount
    else No session
        STR->>CONN: OpenLiveStream()
        CONN->>VMS: SDK / ONVIF / VMS API
        VMS-->>CONN: Media source / handle
        CONN->>MP: Create / attach pipeline
        MP-->>CONN: Pipeline ready/starting
        CONN-->>STR: Session handle
        STR->>R: Store session + refcount=1
    end

    STR-->>GW: Stream URL / token / session metadata
    GW-->>C: 200 OK + stream endpoint

    STR-)BUS: Publish session event
    BUS-)EVT: Deliver event
    EVT-)CH: Store audit/event

    Note over C,MP: Separate MEDIA DATA PATH begins

    C->>NG: Connect to media endpoint where applicable
    C->>MP: Media connection according to protocol
    MP-->>C: Video / audio media

    Note over STR,R: On disconnect, decrement refcount
    Note over STR,R: At zero, close after short idle grace period
```
