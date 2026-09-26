# System Design: Distributed Video Streaming & Transcoding Platform

A highly scalable, fault-tolerant video processing and on-demand streaming architecture (similar to YouTube or Netflix) designed to ingest, transcode, store, and stream millions of concurrent video streams globally with ultra-low latency, zero buffering, and adaptive bitrate switching.

---

## 1. Problem Statement

Building an Internet-scale video platform presents unique architectural challenges that differ fundamentally from standard transactional CRUD services:
- **Massive Binary Ingestion**: Millions of creators uploading multi-gigabyte raw video files concurrently over unreliable networks.
- **Heavy Compute Workloads**: Transcoding high-resolution source videos (4K, 1080p) into dozens of codecs (H.264, VP9, AV1), resolutions, and bitrates is CPU/GPU intensive and requires distributed DAG orchestration.
- **Zero Buffering & Adaptive Streaming**: Millions of global viewers stream videos over unpredictable network conditions (ranging from 3G mobile to gigabit fiber), requiring instant quality switching (ABR - Adaptive Bitrate Streaming via HLS/DASH).
- **Petabyte-Scale Storage & High CDN Costs**: Video represents >65% of global internet bandwidth. Storage and bandwidth costs must be optimized through intelligent tiered caching and hot/cold storage lifecycle management.

---

## 2. System Requirements

### Functional Requirements

| Requirement | Description |
| :--- | :--- |
| **Resumable Chunked Upload** | Creators can upload large video files in chunks with automatic retry on network drops. |
| **Distributed Transcoding** | Ingested raw video is automatically split, transcoded into multiple formats/resolutions, and packaged into HLS/DASH playlists. |
| **Adaptive Bitrate Streaming (ABR)** | Viewers can stream video smoothly; quality switches seamlessly between 240p to 4K based on available bandwidth. |
| **Video Metadata & Search** | Users can search, view metadata, like, comment, and track video view counts in real time. |
| **Thumbnails & Storyboard Generation** | Auto-extracts high-res preview thumbnails and scrub-bar storyboards during transcoding. |

### Non-Functional Requirements

| Requirement | Target Metric | Architectural Approach |
| :--- | :--- | :--- |
| **Low Playback Startup Latency** | Time-to-first-frame < 200ms | CDN edge caching, pre-warmed connections, optimized initial chunk size (2s-4s). |
| **High Availability** | 99.99% playback availability | Multi-region CDN routing, geo-distributed object storage replicas. |
| **Scalability** | 100M+ DAU, 1M concurrent streams | Stateless API tier, asynchronous Kafka/SQS task queues, auto-scaling worker fleet. |
| **Data Durability & Reliability** | Zero raw/transcoded video loss | Erasure-coded multi-AZ object storage (S3/GCS) with lifecycle policies. |
| **Bandwidth & Cost Efficiency** | Minimized egress costs | Tiered CDN caching (Edge Point-of-Presence -> Regional Shield -> Origin), modern codecs (AV1/VP9). |

---

## 3. Scale & Capacity Estimations

### User & Traffic Numbers
- **Daily Active Users (DAU)**: 100 Million
- **Daily Video Uploads**: 500,000 videos/day
- **Average Video Duration**: 10 minutes
- **Raw Upload Size**: ~500 MB average (1080p raw upload)
- **Daily View Count**: 1 Billion views/day
- **Average Viewing Time per User**: 30 minutes/day

### Storage & Ingestion Estimations
```text
Daily Raw Ingest:
  500,000 videos * 500 MB = 250 TB / day
  Ingest Bandwidth = 250 TB / 86,400s ≈ 23 Gbps average (Peak ~60 Gbps)

Transcoding Multiplier:
  Each video is encoded into 5 resolutions (1080p, 720p, 480p, 360p, 240p).
  Total transcoded storage ≈ 1.5x raw size = 375 TB / day.
  Annual Storage Required ≈ 375 TB * 365 ≈ 136.8 PB / year.

Streaming Egress Bandwidth:
  Average Bitrate (Across 720p/1080p mix) ≈ 2.5 Mbps
  Total Concurrent Viewers at Peak (10% DAU) = 10,000,000 concurrent streams
  Peak CDN Egress Bandwidth = 10M streams * 2.5 Mbps = 25 Tbps (Handled by Global CDNs)
```

---

## 4. High-Level Architecture

```mermaid
flowchart TD
    subgraph Ingestion & Upload
        Creator[Video Creator / Mobile App] -->|1. Get Signed Multipart URL| API[API Gateway]
        API --> UploadService[Upload Orchestration Service]
        Creator -->|2. Direct Chunked Upload| S3Raw[(Raw S3 Bucket)]
        S3Raw -->|3. ObjectCreated Event| UploadService
    end

    subgraph Distributed Transcoding Pipeline
        UploadService -->|4. Publish Transcode Job| Kafka[Kafka Job Queue]
        Kafka --> Coordinator[Transcoding DAG Coordinator]
        Coordinator --> Splitter[Video Chunk Splitter]
        Splitter --> WorkerFleet[Distributed Transcoding Workers - GPU/CPU]
        WorkerFleet --> Stitcher[HLS/DASH Manifest Stitcher & Packager]
        Stitcher --> S3Processed[(Processed S3 Bucket - Chunks & .m3u8)]
    end

    subgraph Metadata & Realtime
        UploadService --> VideoDB[(Video Metadata DB - PostgreSQL/Cassandra)]
        WorkerFleet -->|Status Updates| Redis[(Redis Job Cache & PubSub)]
        Coordinator --> PushGateway[WebSocket Notification Gateway]
        PushGateway --> Creator
    end

    subgraph Delivery & Playback
        Viewer[Video Viewer / Player] -->|5. Request Manifest & Chunks| CDN[Global CDN / Edge Cache]
        CDN -->|Cache Miss| S3Processed
    end
```

---

## 5. End-to-End Workflows

### 5.1 Resumable Chunked Video Upload

Direct-to-storage upload reduces API gateway load and avoids double-hop buffering:

```mermaid
sequenceDiagram
    autonumber
    actor Creator
    participant Gateway as API Gateway
    participant UploadSvc as Upload Service
    participant S3 as Raw Object Storage (S3)
    participant Kafka as Message Broker

    Creator->>Gateway: POST /api/v1/videos/upload-session (file_size, checksum, mime)
    Gateway->>UploadSvc: Initialize Upload Session
    UploadSvc->>S3: Initiate Multipart Upload
    S3-->>UploadSvc: upload_id & Presigned URLs for chunks
    UploadSvc-->>Creator: upload_id, chunk_size (8MB), presigned_urls[]
    
    loop For each 8MB Chunk (Parallel)
        Creator->>S3: PUT chunk_data (presigned_url, PartNumber, MD5)
        S3-->>Creator: 200 OK (ETag)
    end

    Creator->>Gateway: POST /api/v1/videos/complete-upload (upload_id, parts[])
    Gateway->>UploadSvc: Complete Multipart Upload
    UploadSvc->>S3: CompleteMultipartUpload(upload_id, parts)
    UploadSvc->>Kafka: Publish `VideoUploadedEvent`
    UploadSvc-->>Creator: Upload Complete (Status: PROCESSING)
```

---

### 5.2 Distributed Transcoding DAG (Directed Acyclic Graph)

A 10-minute video is not processed as a single monolithic file. It is broken into independent Group-of-Pictures (GOP) chunks and processed in parallel across hundreds of workers:

```mermaid
flowchart TD
    RawFile[Raw 1080p MP4 Video] --> Demux[Audio/Video Demuxer & Keyframe Splitter]
    
    Demux --> Chunk1[Chunk 1: 00:00 - 02:00]
    Demux --> Chunk2[Chunk 2: 02:00 - 04:00]
    Demux --> Chunk3[Chunk 3: 04:00 - 06:00]
    Demux --> AudioTrack[Audio Extraction: AAC 128kbps / Opus]
    
    Chunk1 --> Enc1_1080[1080p Worker]
    Chunk1 --> Enc1_720[720p Worker]
    Chunk1 --> Enc1_480[480p Worker]

    Chunk2 --> Enc2_1080[1080p Worker]
    Chunk2 --> Enc2_720[720p Worker]
    Chunk2 --> Enc2_480[480p Worker]

    Enc1_1080 & Enc2_1080 & Enc1_720 & Enc2_720 & AudioTrack --> Packager[HLS/DASH Packager]
    Packager --> MasterPlaylist["Master Playlist (master.m3u8)"]
    Packager --> TSChunks["Media Segments (.ts / .m4s)"]
```

---

### 5.3 Adaptive Bitrate Streaming (ABR) Playback

The client-side video player continually samples network throughput and buffer fill levels to request the optimal segment quality:

```mermaid
sequenceDiagram
    autonumber
    actor Player as Video Player
    participant CDN as CDN Edge
    participant S3 as Origin Storage (S3)

    Player->>CDN: GET /videos/{id}/master.m3u8
    CDN-->>Player: Master Playlist (Bitrate Variants: 1080p, 720p, 480p)
    
    Note over Player: Player selects initial bitrate based on network check (e.g. 720p)
    Player->>CDN: GET /videos/{id}/720p/index.m3u8
    CDN-->>Player: Variant Playlist (List of segment URLs)
    
    Player->>CDN: GET /videos/{id}/720p/segment_001.ts
    CDN-->>Player: Video Segment 1 (Buffer 0s-5s)
    
    Note over Player: Bandwidth drops! Player switches seamlessly to 480p
    Player->>CDN: GET /videos/{id}/480p/segment_002.ts
    CDN-->>Player: Video Segment 2 (Buffer 5s-10s)
```

---

## 6. Detailed Documentation

Explore in-depth design documents for each component:

* 📐 [Detailed Architecture & Component Deep Dive](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/architecture.md)
* 🔌 [API Contracts & Endpoints](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/api_design.md)
* 📊 [Capacity Planning & Bandwidth Math](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/capacity_planning.md)
* ⚙️ [Distributed Transcoding DAG Pipeline](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/transcoding_pipeline_dag.md)
* 📡 [HLS & DASH Adaptive Bitrate Streaming](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/hls_dash_streaming.md)
* 🗄️ [Database Schema & Data Model](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/database_schema.md)
* 🌐 [CDN & Edge Caching Strategy](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/cdn_and_caching_strategy.md)
* 🛡️ [Failure Scenarios & Resilience](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/docs/failure_scenarios.md)

---

## 7. Examples & Reference

* 📄 [Master & Variant HLS Playlist Sample](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/examples/master_playlist.m3u8)
* 📦 [Sample Transcoding Event Payload](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/examples/sample_transcoding_job.json)
* 💻 [Chunking & ABR Pseudo-code](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/examples/pseudo_code.md)

---

## License

This project is licensed under the MIT License - see the [LICENSE](file:///Users/rummansiddiqui/Downloads/systemdesigns/system-design-video-streaming/LICENSE) file for details.
